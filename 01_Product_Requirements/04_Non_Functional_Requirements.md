# 4. 非功能性需求 (Non-Functional Requirements) - Deep Dive

## 4.1 安全性 (Security)

### 4.1.1 傳輸層安全 (Transport Security)
*   **TLS 1.3 強制**: 停用 TLS 1.0/1.1 等老舊協定.
*   **Cipher Suits**: 僅允許 `TLS_AES_256_GCM_SHA384` 等高強度加密套件.
*   **mTLS (雙向認證)**: 醫院 HIS 與 API Gateway 之間必須交換 X.509 客戶端憑證, 確保來源可信.

### 4.1.2 資料保存 (Data At Rest)
*   **KMS Integration**: 使用雲端 KMS (Key Management Service) 管理主金鑰 (Target: AWS KMS / Azure Key Vault).
*   **Envelope Encryption**: 個案 PII 資料使用 Data Key 加密, Data Key 再由 Master Key 加密.
*   **Key Rotation**: 每 90 天自動輪替 Master Key.

## 4.2 效能指標 (Performance SLAs)

| 指標 | 目標值 | 測試場景 |
| :--- | :--- | :--- |
| **API Latency** | P95 < 200ms | 一般查詢 (單筆個案) |
| **Search Latency** | P95 < 500ms | 複雜媒合查詢 (跨區/多條件) |
| **Throughput** | 1000 TPS | 尖峰時段 (早上 8:00-9:00 簽到潮) |
| **Concurrency** | 5000 Active Users | 同時在線個管師/居服員 |

## 4.3 合規性 (Compliance)

### 4.3.1 FHIR TW Core IG
*   **Validation**: 所有進出的 JSON Payload 必須通過 FHIR Validator 檢核 (基於 `StructureDefinition`).
*   **Version**: 支援 FHIR R4.0.1 及 TW Core IG CI-Build 版本.

### 4.3.2 稽核軌跡 (Audit Trail)
*   **Immutable**: 稽核日誌須寫入 WORM (Write Once Read Many) 儲存體.
*   **Retention**: 醫療相關紀錄保留至少 **7 年** (醫療法規); 財務核銷紀錄保留 **10 年** (會計法規).
*   **Content**: 紀錄必須包含 `UserIP`, `Timestamp (UTC)`, `ResourceID`, `Action`, `UserAgent`.

### 4.3.3 公平與可近性 (Equity & Accessibility)
*   **Coverage Gap Monitoring**: 系統需追蹤縣市覆蓋率差距與等待時間，避免資源集中導致服務斷點。
*   **Info Accessibility**: 需提供低數位門檻管道 (LINE、語音) 讓家庭照顧者可查詢進度與權益。
*   **Regional SLA**: 偏鄉或高缺口地區應納入差別化 SLA 與資源補強指標。

## 4.4 同意與資料治理 (Consent & Data Governance)

*   **Consent Lifecycle**: 系統需支援同意的建立、更新、撤回與稽核。
*   **Minimum Necessary**: 預設僅揭露完成媒合/服務所需欄位；敏感欄位需明確授權。
*   **Revocation Behavior**: 同意撤回後，需立刻停止後續資料分享；已產生之稽核紀錄不得刪除。
*   **Delegated Access**: 家屬可授權次要照顧者查詢進度，但不得預設取得完整醫療資料。

## 4.5 演算法公平與可稽核性 (Algorithmic Fairness & Auditability)

*   **Fairness Metrics**:
    *   需量測媒合結果對不同區域/不同規模 B 單位的分配差異 (e.g., win-rate、等待時間分布)。
    *   需特別追蹤偏鄉/高缺口區域的服務到位時效。
*   **Intervention Controls**:
    *   支援政策加權參數 (如偏鄉保底加分) 並保留版本紀錄。
    *   當偵測到分配失衡時，需有人工調參與回復機制。
*   **Explainability**:
    *   對個管師提供 Top-N 媒合理由 (距離/技能/時間/加權) 的可讀摘要。

## 4.6 營運可用性 (Operational Readiness)

*   **Runbook**: 需定義關鍵事件 (改期、取消、爽約、安全事件) 的處理流程與責任歸屬。
*   **Support SLA**: 針對高風險事件通報，需定義 30 分鐘內回應之值勤機制。
