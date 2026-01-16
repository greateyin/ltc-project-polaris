# 5. 外部介面規格 (Interface Specifications)

本章節定義與外部政府系統介接的詳細規格。

## IF-01: 衛生福利部照顧管理資訊平台 (CMS)

*   **性質**: Inbound/Outbound (雙向同步).
*   **連線方式**: Site-to-Site VPN or Gov-Cloud Direct Connect.
*   **認證**: API Key + Whitelist IP.

### 1. 查詢個案 CMS 等級
*   **Method**: `GET`
*   **URL**: `https://cms-gateway.mohw.gov.tw/api/v2/citizen/cms-level`
*   **Query Params**:
    *   `pid`: 身分證字號 (Plaintext, over TLS)
*   **Response**:
    ```json
    {
      "pid": "A123456789",
      "level": 7,
      "disability_category": ["01", "05"],
      "last_assessed_at": "2026-01-10",
      "care_manager_id": "CM_9988"
    }
    ```

### 2. 回報服務計畫 (Sync Plan)
*   **Method**: `POST`
*   **URL**: `https://cms-gateway.mohw.gov.tw/api/v2/care-plan/sync`
*   **Body**:
    ```json
    {
      "pid": "A123456789",
      "plan_type": "DISCHARGE_TRANSITION",
      "items": [
        { "code": "BA01", "frequency": "DAILY" },
        { "code": "EF_R02", "frequency": "ONE_TIME" }
      ]
    }
    ```

---

## IF-02: 長照服務費用支付審核系統 (支審系統)

*   **性質**: Outbound (單向申報).
*   **連線方式**: https over Internet.
*   **認證**: 雙向 TLS (mTLS) + 權杖.

### 1. 批次上傳申報檔 (Batch Upload)
*   **Method**: `POST`
*   **URL**: `https://payment.mohw.gov.tw/api/claim/upload`
*   **Content-Type**: `application/xml` (依據官方舊制規範) 或 `application/json` (若新制支援).
*   **Body**:
    ```xml
    <ClaimPackage>
        <OrgCode>1234567890</OrgCode>
        <BatchDate>2026-06-01</BatchDate>
        <Records>
            <Record>
                <CaseID>A123456789</CaseID>
                <ServiceCode>BA01</ServiceCode>
                <ExecDate>2026-05-31</ExecDate>
                <ExecTimeStart>09:00</ExecTimeStart>
                <ExecTimeEnd>10:00</ExecTimeEnd>
                <StaffID>S_001</StaffID>
            </Record>
        </Records>
    </ClaimPackage>
    ```

---

## IF-03: 醫院 HIS FHIR Gateway

*   **性質**: Inbound (接收轉介).
*   **標準**: FHIR R4 TW Core IG.

### 1. 提交 Bundle (交易模式)
*   **Method**: `POST`
*   **URL**: `/api/v1/referral`
*   **Body**: 見 `03_Program_Specifications/01_API_Specifications.md`.
*   **Error Codes**:
    *   `400 Bad Request`: FHIR Validation Failed. (回傳 `OperationOutcome` 資源).
    *   `422 Unprocessable Entity`: 缺必要的識別欄位 (如身分證號).
    *   `409 Conflict`: 該個案已存在且正在評估中 (防止重複轉介).
