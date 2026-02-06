# 2. 資料庫綱要設計 (Database Schema Design) - Deep Dive

## 2.1 資料庫優化策略 (Optimization Strategy)
*   **Partitioning (分割)**:
    *   `SERVICE_ORDERS`: 依據 `created_month` 進行 Range Partition，以優化歷史報表查詢與 archival。
    *   `AUDIT_LOGS`: 依據 `event_date` 進行 Daily Partition。
*   **Indexing (索引)**:
    *   使用 **GIN Index** 於 `cases.fhir_data` (JSONB) 欄位，加速 FHIR 屬性查詢。
    *   使用 **GiST Index** 於 `providers.service_area` (Geometry) 欄位，加速 GIS 範圍搜尋。
*   **Encryption**:
    *   敏感欄位 (PII) 使用 **Column-level Encryption** (pgcrypto 或 AP 層加密)，Key ID 儲存於 KMS。

---

## 2.2 詳細 Schema 定義 (DDL Highlight)

### Schema: `CORE`

#### Table: `cases` (個案主檔)
```sql
CREATE TABLE core.cases (
    case_uuid       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    national_id_hash CHAR(64) NOT NULL, -- SHA256 w/ Salt
    enc_name        TEXT NOT NULL,      -- AES-256 Encrypted
    enc_dob         TEXT NOT NULL,      -- AES-256 Encrypted
    cms_level       SMALLINT CHECK (cms_level BETWEEN 1 AND 8),
    welfare_status  VARCHAR(20) DEFAULT 'GENERAL', -- GENERAL, MID_LOW, LOW
    fhir_data       JSONB,              -- 完整 FHIR Patient Resource
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_cases_nid_hash ON core.cases(national_id_hash);
CREATE INDEX idx_cases_fhir ON core.cases USING GIN (fhir_data);
```

#### Table: `subsidy_ledger` (補助額度帳本 - 智慧租賃專用)
為了支援「滾動式 3 年 6 萬」邏輯，我們採用 **Event Sourcing** 概念設計此表。

```sql
CREATE TABLE core.subsidy_ledger (
    ledger_uuid     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid       UUID NOT NULL REFERENCES core.cases(case_uuid),
    txn_type        VARCHAR(20) NOT NULL, -- GRANT, RESERVE, COMMIT, ROLLBACK
    category        VARCHAR(50) NOT NULL, -- RENTAL_DEVICE_SMART, HOME_CARE_B
    amount          DECIMAL(12, 2) NOT NULL, -- 正負值
    ref_order_uuid  UUID,
    effective_date  DATE NOT NULL,
    created_at      TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (effective_date);

-- 計算 Logic:
-- SELECT SUM(amount) FROM subsidy_ledger 
-- WHERE case_uuid = ? AND effective_date >= NOW() - INTERVAL '3 years';
```

#### Table: `match_requests` (媒合請求)
包含 GIS 資料與複雜篩選條件。

```sql
CREATE TABLE core.match_requests (
    request_uuid    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid       UUID NOT NULL,
    target_geo      GEOMETRY(POINT, 4326) NOT NULL, -- 案主居住地
    constraints     JSONB, -- {"gender": "F", "skills": ["BA01", "AA05"]}
    priority_score  INT DEFAULT 0,
    status          VARCHAR(20) DEFAULT 'OPEN',
    expires_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_match_geo ON core.match_requests USING GIST (target_geo);
```

#### Table: `providers` (服務提供者主檔)
用於 B 單位導入、服務區域、聯絡窗口與合約狀態管理。

```sql
CREATE TABLE core.providers (
    provider_uuid     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    org_code          VARCHAR(32) UNIQUE, -- 機構代碼 (若有)
    name              TEXT NOT NULL,
    contact_name      TEXT,
    contact_phone     TEXT,
    service_categories TEXT[] NOT NULL, -- e.g., ['BA', 'DA', 'EF_R']
    service_area      GEOMETRY(POLYGON, 4326),
    status            VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, SUSPENDED, OFFBOARDED
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_providers_area ON core.providers USING GIST (service_area);
```

#### Table: `provider_staff` (服務人員與證照)
用於排班/技能匹配，並支援證照有效性與撤銷。

```sql
CREATE TABLE core.provider_staff (
    staff_uuid        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    provider_uuid     UUID NOT NULL REFERENCES core.providers(provider_uuid),
    name              TEXT NOT NULL,
    license_codes     TEXT[] NOT NULL, -- e.g., ['AA01', 'BA02']
    license_expiry    DATE,
    status            VARCHAR(20) DEFAULT 'ACTIVE',
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_staff_provider ON core.provider_staff(provider_uuid);
```

#### Table: `consent_records` (同意紀錄)
以可稽核方式保存個案同意 (範圍/期限/撤回)；撤回不刪除，僅標記失效。

```sql
CREATE TABLE core.consent_records (
    consent_uuid      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid         UUID NOT NULL REFERENCES core.cases(case_uuid),
    scope             JSONB NOT NULL, -- e.g., {"share_fhir": true, "share_with": ["PROVIDER"], "fields": [...]}
    valid_from        TIMESTAMPTZ NOT NULL,
    valid_until       TIMESTAMPTZ,
    revoked_at        TIMESTAMPTZ,
    revoked_reason    TEXT,
    created_by        UUID,
    created_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_consent_case ON core.consent_records(case_uuid);
```

#### Table: `case_episodes` (個案服務週期)
支援同一個案多次啟動/結案 (例如再入院後 90 天內重啟)。

```sql
CREATE TABLE core.case_episodes (
    episode_uuid      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid         UUID NOT NULL REFERENCES core.cases(case_uuid),
    status            VARCHAR(20) NOT NULL, -- NEW/ACTIVE/CLOSED
    open_reason       TEXT,
    close_reason      TEXT,
    opened_at         TIMESTAMPTZ DEFAULT NOW(),
    closed_at         TIMESTAMPTZ
);

CREATE INDEX idx_episode_case ON core.case_episodes(case_uuid);
```

#### Table: `incident_reports` (事件通報)
記錄爽約/拒絕服務/安全事件等現場狀況，並可附證據連結。

```sql
CREATE TABLE core.incident_reports (
    report_uuid       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid         UUID NOT NULL REFERENCES core.cases(case_uuid),
    service_order_uuid UUID,
    report_type       VARCHAR(30) NOT NULL, -- NO_SHOW, REFUSED, SAFETY_INCIDENT
    severity          VARCHAR(10) DEFAULT 'LOW', -- LOW, MEDIUM, HIGH
    description       TEXT,
    evidence_urls     TEXT[],
    occurred_at       TIMESTAMPTZ NOT NULL,
    created_by        UUID,
    created_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_incident_case ON core.incident_reports(case_uuid);
```

#### Table: `incident_tickets` (事件工單)
用於追蹤處理狀態與升級 (ACK/ESCALATED)。

```sql
CREATE TABLE core.incident_tickets (
    ticket_uuid       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_uuid       UUID REFERENCES core.incident_reports(report_uuid),
    ticket_type       VARCHAR(30) NOT NULL, -- RESCHEDULE, CANCEL, INCIDENT
    status            VARCHAR(20) DEFAULT 'OPEN', -- OPEN, ACKED, ESCALATED, CLOSED
    assigned_to       UUID,
    ack_deadline      TIMESTAMPTZ,
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    updated_at        TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_ticket_status ON core.incident_tickets(status);
```

#### Table: `handoff_summaries` (共案交接摘要)
以版本化方式保存交接摘要；每次服務結束更新版本。

```sql
CREATE TABLE core.handoff_summaries (
    summary_uuid      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    case_uuid         UUID NOT NULL REFERENCES core.cases(case_uuid),
    version           INT NOT NULL DEFAULT 1,
    summary           JSONB NOT NULL, -- preferences, risks, notes
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE (case_uuid, version)
);

CREATE INDEX idx_handoff_case ON core.handoff_summaries(case_uuid);
```

#### Table: `handoff_acks` (交接確認)
新接手服務者需 ACK 才能開始打卡。

```sql
CREATE TABLE core.handoff_acks (
    ack_uuid          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    summary_uuid      UUID NOT NULL REFERENCES core.handoff_summaries(summary_uuid),
    staff_uuid        UUID NOT NULL,
    acked_at          TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_handoff_ack_summary ON core.handoff_acks(summary_uuid);
```

#### Table: `metrics_coverage_gap` (覆蓋率與等待時間)
用於政策 KPI 追蹤，避免區域資源缺口。

```sql
CREATE TABLE analytics.metrics_coverage_gap (
    period_month      DATE NOT NULL,
    region_code       VARCHAR(10) NOT NULL, -- 縣市/鄉鎮
    eligible_count    INT NOT NULL,
    served_count      INT NOT NULL,
    coverage_rate     DECIMAL(5,2) NOT NULL, -- served/eligible
    avg_wait_days     DECIMAL(6,2) NOT NULL,
    p90_wait_days     DECIMAL(6,2) NOT NULL,
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (period_month, region_code)
);
```

#### Table: `metrics_workforce_load` (人力負荷)
追蹤個管/照專平均案量與超載比例。

```sql
CREATE TABLE analytics.metrics_workforce_load (
    period_month      DATE NOT NULL,
    region_code       VARCHAR(10) NOT NULL,
    staff_type        VARCHAR(20) NOT NULL, -- CARE_MANAGER, SUPERVISOR
    active_cases      INT NOT NULL,
    staff_count       INT NOT NULL,
    avg_cases_per_staff DECIMAL(6,2) NOT NULL,
    overload_ratio    DECIMAL(5,2) NOT NULL, -- 超過警戒值比例
    created_at        TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (period_month, region_code, staff_type)
);
```

**來源說明**:
*   指標由事件流 (Kafka) 與 CMS 統計匯入，採月彙整並保留歷史快照。

---

## 2.3 Redis 快取設計 (Cache Strategy)

| Key Pattern | Data Structure | TTL | Purpose |
| :--- | :--- | :--- | :--- |
| `sess:{user_token}` | Hash | 30m | 用戶登入會話與權限 (Role Scope) |
| `inv:device:{provider_id}:{sku}` | String (Int) | 5m | 供應商輔具即時庫存 (Real-time Inventory) |
| `rate:ip:{ip_addr}` | Counter | 1m | API Rate Limiting (防禦 DDoS) |
| `geo:providers:{category}` | GeoSet | - | 服務單位位置索引 (加速半徑搜尋) |

---

## 2.4 FHIR Storage Strategy (Hybrid Pattern)

為平衡 FHIR 標準的靈活性與 SQL 查詢的高效能，採用 **Hybrid Storage Model**。

### 2.4.1 Storage Rules
1.  **Raw Data**: 完整的 FHIR Resource (JSON) 儲存於 `fhir_data` (JSONB) 欄位，確保 100% 原始資料保真度。
2.  **Search Params**: 頻繁查詢欄位 (e.g., `identifier`, `gender`, `birthDate`) **必須** 提取至 Table 的獨立 Column，並建立 B-Tree 索引。
3.  **Complex Query**: 針對 JSONB 內部的巢狀查詢 (e.g., `Patient.contact.telecom`), 使用 **GIN Index** (`jsonb_path_ops`)。

### 2.4.2 Indexing Example
```sql
-- 1. 針對任意 JSON 路徑的高效查詢
CREATE INDEX idx_cases_fhir_gin ON core.cases USING GIN (fhir_data jsonb_path_ops);

-- 2. 針對特定 FHIR 路徑的提取索引 (Functional Index)
-- 查詢: WHERE fhir_data @> '{"contact": [{"relationship": [{"coding": [{"code": "C"}]}]}]}'
CREATE INDEX idx_cases_contact_rel ON core.cases 
USING GIN ((fhir_data -> 'contact') jsonb_path_ops);
```
