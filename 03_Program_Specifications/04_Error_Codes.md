# 4. 錯誤代碼與處理 (Error Codes & Handling) - Architect Edition

## 4.1 標準錯誤回應 (Standard Error Response)

所有 API (REST & Internal) 在發生錯誤時，必須回傳統一的 JSON 結構 (基於 RFC 7807 Problem Details)。

```json
{
  "type": "https://ltc-api.gov.tw/probs/insufficient-quota",
  "title": "額度不足",
  "status": 402,
  "detail": "個案 3 年租賃額度剩餘 $2000，不足以支付 $5000",
  "instance": "/api/v1/subsidy/calc",
  "trace_id": "req_88192001",
  "code": "E-402-001"
}
```

## 4.2 全域錯誤碼表 (Global Error Table)

### 400 Bad Request (Validation Errors)
| Code | Message | Scenario | Action |
| :--- | :--- | :--- | :--- |
| `E-400-001` | Invalid FHIR Bundle | 上傳的 JSON 不符合 TW Core IG 定義 | Client fix payload |
| `E-400-002` | Missing Mandatory Field | 缺少身分證字號或出生日期 | Client fix payload |

### 401/403 Auth Errors
| Code | Message | Scenario | Action |
| :--- | :--- | :--- | :--- |
| `E-401-001` | Token Expired | JWT 已過期 | Client refresh token |
| `E-403-001` | Insufficient Scope | B單位嘗試存取 A單位專屬 API | Check permissions |

### 402 Payment/Quota Errors (Business Logic)
| Code | Message | Scenario | Action |
| :--- | :--- | :--- | :--- |
| `E-402-001` | Insufficient Subsidy Quota | 額度試算餘額不足 | 提示改為自費或更換項目 |
| `E-402-002` | Exclusion Rule Violation | 已有購置紀錄，不可租賃同類產品 | 拒絕交易 |

### 500 System Errors
| Code | Message | Scenario | Actions |
| :--- | :--- | :--- | :--- |
| `E-500-001` | Matching Engine Timeout | GIS 計算超時 | Retry with exponential backoff |
| `E-500-002` | Upstream CMS Unavailable | 衛福部 API 連線失敗 | Queue request for later retry |

## 4.3 重試策略 (Retry Policy)

*   **Transient Errors (瞬間錯誤)**: 如 Network Timeout, HTTP 503.
    *   **Policy**: Exponential Backoff (Init: 100ms, Max: 5s, Attempt: 3).
*   **Permanent Errors (永久錯誤)**: 如 400 Bad Request, 401.
    *   **Policy**: No Retry. Fail Fast.
