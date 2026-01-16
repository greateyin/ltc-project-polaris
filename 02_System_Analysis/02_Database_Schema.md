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

---

## 2.3 Redis 快取設計 (Cache Strategy)

| Key Pattern | Data Structure | TTL | Purpose |
| :--- | :--- | :--- | :--- |
| `sess:{user_token}` | Hash | 30m | 用戶登入會話與權限 (Role Scope) |
| `inv:device:{provider_id}:{sku}` | String (Int) | 5m | 供應商輔具即時庫存 (Real-time Inventory) |
| `rate:ip:{ip_addr}` | Counter | 1m | API Rate Limiting (防禦 DDoS) |
| `geo:providers:{category}` | GeoSet | - | 服務單位位置索引 (加速半徑搜尋) |
