# 1. API 介面規格 (API Specifications) - Architect Edition

## 1.1 設計規範 (Design Guidelines)
*   **Protocol**: 
    *   External (Web/App/Partner) -> **REST over HTTP/2**
    *   Internal (Agent-to-Agent) -> **gRPC (Protobuf)**
*   **Auth**: JWT (RS256) in `Authorization` header.
*   **Rate Limit**: 基於 Redis Token Bucket (X-RateLimit-Limit).

---

## 1.2 Public REST API (For Provider App / Admin Web)

### 1.2.1 接案管理 (Provider Operations)

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
