# 1. API 介面規格 (API Specifications) - Architect Edition

## 1.1 設計規範 (Design Guidelines)
*   **Protocol**: 
    *   External (Web/App/Partner) -> **REST over HTTP/2**
    *   Internal (Agent-to-Agent) -> **gRPC (Protobuf)**
*   **Auth**: JWT (RS256) in `Authorization` header.
*   **Rate Limit**: 基於 Redis Token Bucket (X-RateLimit-Limit).
*   **Region Context**: 涉及公平性監測的 API 必須帶 `region_code` (或能由個案/機構推導)。

## 1.1.1 版本控管與相容性 (Versioning & Compatibility)
*   **REST**: 以 `/api/v1` 做主版號，minor 變更需保持向後相容，breaking change 必須升版 (`/api/v2`)。
*   **Deprecation**: 新舊版平行運行至少 6 個月，並提前 90 天公告停用時程。
*   **gRPC**: 僅新增 optional 欄位，不重用欄位編號；破壞性變更以新 package/version 發佈。
*   **Events**: 使用 Schema Registry 管控 `event_version`，僅允許 backward compatible 變更。

## 1.1.2 FHIR 版本鎖定與 Conformance
*   **FHIR Baseline**: 固定 FHIR R4.0.1 + TW Core IG (snapshot: 2026-01-16) 作為規格凍結基準。
*   **Conformance Artifacts**: 發佈 CapabilityStatement、StructureDefinition、ValueSet 供 HIS 端對接與驗證。
*   **升級策略**: 新版本以 migration window 推出，並提供 diff/validation 報告。

---

## 1.2 Public REST API (For Provider App / Admin Web)

### 1.2.1 接案管理 (Provider Operations)

#### `GET /api/v1/admin/metrics/coverage-gap`
查詢各縣市/鄉鎮覆蓋率與等待時間指標，用於公平性監測。

*   **Query Params**:
    *   `period`: 月份 (YYYY-MM)
    *   `region_code`: 縣市/鄉鎮代碼 (Optional)
*   **Response (200 OK)**:
    ```json
    {
      "items": [
        {
          "period": "2026-06",
          "region_code": "TPE",
          "coverage_rate": 0.82,
          "avg_wait_days": 3.4,
          "p90_wait_days": 7.2
        }
      ],
      "meta": { "total": 1, "page": 1 }
    }
    ```

#### `GET /api/v1/admin/metrics/workforce-load`
查詢個管/照專人力負荷與超載比例，用於人力壓力監測。

*   **Query Params**:
    *   `period`: 月份 (YYYY-MM)
    *   `region_code`: 縣市/鄉鎮代碼 (Optional)
    *   `staff_type`: `CARE_MANAGER` | `SUPERVISOR`
*   **Response (200 OK)**:
    ```json
    {
      "items": [
        {
          "period": "2026-06",
          "region_code": "TPE",
          "staff_type": "CARE_MANAGER",
          "avg_cases_per_staff": 128.5,
          "overload_ratio": 0.23
        }
      ],
      "meta": { "total": 1, "page": 1 }
    }
    ```

#### `GET /api/v1/provider/orders/available`
獲取當前可接案的列表 (基於 Geo-Fencing 與資格過濾)。

*   **Query Params**:
    *   `lat`, `lng`: 當前位置 (必填)
    *   `radius`: 搜尋半徑 (default: 5km)
    *   `category`: 篩選類別 (Optional: `HOME_CARE`, `RENTAL`)
*   **Response (200 OK)**:
    ```json
    {
      "items": [
        {
          "order_id": "ord_881923",
          "distance_km": 1.2,
          "cms_level": 7,
          "service_items": ["BA01", "BA02"],
          "subsidy_flags": ["URGENT", "REMOTE_BONUS"],
          "expiration_time": "2026-06-01T10:30:00Z"
        }
      ],
      "meta": { "total": 1, "page": 1 }
    }
    ```

#### `POST /api/v1/provider/orders/{order_id}/accept`
接單 (搶單模式)。

*   **Logic**: 使用 Redis `SETNX` 鎖定訂單，防止超賣。
*   **Response**:
    *   `200 OK`: 接單成功。
    *   `409 Conflict`: 訂單已被搶走。

### 1.2.2 輔具庫存管理 (Device Inventory)

#### `POST /api/v1/admin/metrics/coverage-gap/ingest`
匯入月度覆蓋率與等待時間指標 (由 CMS 統計或事件流彙整)。

*   **Payload**:
    ```json
    {
      "period": "2026-06",
      "region_code": "TPE",
      "eligible_count": 120000,
      "served_count": 98400,
      "coverage_rate": 0.82,
      "avg_wait_days": 3.4,
      "p90_wait_days": 7.2
    }
    ```
*   **Response**:
    *   `202 Accepted`: 已接收，進入批次校驗與入庫。

#### `POST /api/v1/admin/metrics/workforce-load/ingest`
匯入月度人力負荷指標。

*   **Payload**:
    ```json
    {
      "period": "2026-06",
      "region_code": "TPE",
      "staff_type": "CARE_MANAGER",
      "active_cases": 15420,
      "staff_count": 120,
      "avg_cases_per_staff": 128.5,
      "overload_ratio": 0.23
    }
    ```
*   **Response**:
    *   `202 Accepted`: 已接收，進入批次校驗與入庫。

#### `POST /api/v1/provider/inventory/sync`
同步輔具庫存狀態 (供系統做媒合判斷)。

*   **Payload**:
    ```json
    {
      "provider_id": "prov_7788",
      "items": [
        { "sku": "LIFT_01", "desc": "電動移位機(B款)", "qty_available": 3 },
        { "sku": "MAT_AI_02", "desc": "智慧感測床墊", "qty_available": 0 }
      ]
    }
    ```

---

## 1.3 Internal gRPC API (Microservices)

定義於 `.proto` 檔案中，用於 Agent 間的高效通訊。

### 1.3.1 Matching Service (`match.proto`)

```protobuf
syntax = "proto3";

package ltc.agent.match;

service MatchEngine {
  // 計算候選人分數
  rpc CalculateScore (ScoreRequest) returns (ScoreResponse);
  
  // 尋找最佳路徑 (GIS)
  rpc FindRoute (RouteRequest) returns (RouteResponse);
}

message ScoreRequest {
  string case_id = 1;
  string provider_id = 2;
  double distance_km = 3;
  repeated string required_skills = 4;
}

message ScoreResponse {
  string provider_id = 1;
  double total_score = 2; // 0-100
  map<string, double> breakdown = 3; // "dist": 30, "skill": 20
}
```

### 1.3.2 Resource Service (`resource.proto`)

```protobuf
syntax = "proto3";

package ltc.agent.resource;

service SubsidyCalculator {
  // 試算補助額度
  rpc CalculateGrant (GrantRequest) returns (GrantResponse);
}

message GrantRequest {
  string case_id = 1;
  string item_category = 2; // e.g. "RENTAL_DEVICE"
  double item_price = 3;
}

message GrantResponse {
  bool eligible = 1;
  double subsidy_amount = 2;
  double self_pay_amount = 3;
  string rejection_reason = 4; // if not eligible
}
```
