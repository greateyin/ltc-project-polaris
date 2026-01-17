# 3. 安全性與合規設計 (Security & Compliance) - Deep Dive

## 3.1 零信任架構 (Zero Trust Architecture)

本系統假設「內網亦不可信」，落實以下 ZTA 原則：

1.  **從不信任，驗證一切 (Never Trust, Always Verify)**:
    *   服務間通訊 (Service-to-Service) 強制使用 **mTLS** (由 Istio Sidecar 執行)。
    *   所有 API 請求 (包含內部 Agent 呼叫) 必須攜帶有效的 **JWT (Service Token)**。

2.  **最小權限原則 (Least Privilege)**:
    *   DB 存取帳號分離：`Agent_Demand` 只能 `SELECT/INSERT cases`，不能存取 `providers` 表。
    *   K8s SA (Service Account) 權限限縮：僅掛載必要的 Secret。

---

## 3.2 資料保護與加密 (Data Protection)

### 3.2.1 KMS金鑰管理架構 (Key Management)
採用 **Envelope Encryption (信封加密)** 機制：

*   **Master Key (CMK)**: 儲存於 AWS KMS / Azure Key Vault (FIPS 140-2 Level 2+)。永遠不離開 HSM。
*   **Data Encryption Key (DEK)**: 每個 Case 產生一把獨立的 DEK。
    *   DEK (Plaintext) 用於加密該已 Case 的資料。
    *   DEK (Ciphertext) = `Encrypt(CMK, DEK_Plain)`，儲存於 DB `cases_keys` 表中。

### 3.2.2 敏感資料遮罩 (Masking)
在非生產環境 (Dev/UAT) 或低權限人員檢視時：
*   姓名：`王O明`
*   身分證：`A1xxxx6789`
*   地址：`台北市信義區xxxxx` (隱藏街道號碼)

---

## 3.3 合規性驗證 (Compliance Validation)

### 3.3.1 FHIR TW Core IG 合規
*   **Validator Service**: 部署官方 HAPI FHIR Validator 實例。
*   **Validation Pipeline**:
    *   Ingress: 來自 HIS 的 Bundle 進入 Inbound Gateway 時，同步呼叫 Validator。
    *   Logic: 若驗證失敗 (e.g., 缺少 Identifier System)，回傳 `400 Bad Request` + `OperationOutcome` 錯誤詳情。

### 3.3.2 稽核日誌 (Audit Logging)
符合 ISO 27001 A.12.4 規範。

**日誌格式 (JSON)**:
```json
{
  "timestamp": "2026-06-01T08:00:00Z",
  "correlation_id": "req_12345uuid",
  "actor": {
    "user_id": "usr_999",
    "role": "CASE_MANAGER",
    "ip": "211.20.x.x"
  },
  "action": "VIEW_CASE_DETAILS",
  "resource": {
    "type": "Case",
    "id": "case_5566"
  },
  "outcome": "SUCCESS",
  "context": {
    "reason": "Annual Review"
  }
}
```
*   **WORM Storage**: 日誌即時串流至 S3 Object Lock (Compliance Mode) 儲存桶，防止刪除或竄改。

---

## 3.4 資料主權與同意管理 (Data Sovereignty & Consent)

*   **Residency**: PII 與醫療資料必須儲存在政府雲/機房，跨境存取需經授權與去識別化。
*   **Consent Record**: 個案同意需以可稽核方式保存 (含同意範圍、有效期限、撤回紀錄)。
*   **Minimization**: API 回傳採最小必要原則，避免不必要欄位外洩。
*   **Deletion/Revocation**: 同意撤回後，停止二次利用並啟動資料封存或刪除流程。

## 3.4A 公平與可近性合規 (Equity & Accessibility Controls)
*   **Coverage Monitoring**: 依縣市/鄉鎮追蹤覆蓋率與等待時間，做為政策監督與資源補強依據。
*   **Low-Digital Access**: 家屬與長輩可透過 LINE/語音查詢，不以 App 作為唯一通路。
*   **SLA Parity**: 偏鄉與高缺口區域需定義差別化 SLA，避免平均值掩蓋落差。

## 3.5 災難復原與營運持續 (DR/BCP)

*   **RPO/RTO**: 核心系統 RPO <= 15 分鐘，RTO <= 2 小時。
*   **Backup**: DB 每日全量、每 15 分鐘增量，備份加密並存放異地。
*   **Failover**: 需支援跨區域故障切換與演練 (至少每半年一次)。

## 3.6 事件通報與應變 (Incident Response)

*   **Detection**: SIEM 監控與異常行為偵測 (含 API 失敗率、異常存取)。
*   **Response**: 依資安等級分級通報，重大事件需於 72 小時內完成初步通報。
*   **Forensics**: 保存關鍵日誌與關聯 ID，支援事後鑑識與責任追溯。
