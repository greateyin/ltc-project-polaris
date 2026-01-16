# 4. 關鍵流程時序圖 (Sequence Diagrams)

本章節使用 Mermaid 描述系統關鍵的互動流程，特別著重於 **Saga Transaction** 與 **Multi-Agent 協商**。

## SEQ-01: 出院轉介與自動分案 (Discharge Referral & Auto-Dispatch)

**情境**: 醫院 HIS 發送出院轉介，系統自動建立案件並嘗試首次媒合。

```mermaid
sequenceDiagram
    autonumber
    
    participant HIS as 醫院 HIS
    participant GW as API Gateway (Kong)
    participant Orch as Orchestrator (Temporal)
    participant Ag_D as Demand Agent
    participant Ag_M as Matching Agent
    participant DB as Core DB
    participant Line as Line Bot Platform

    Note over HIS, GW: mTLS Encrypted Channel

    HIS->>GW: POST /api/v1/referral (FHIR Bundle)
    GW->>GW: Validate JWT & mTLS
    GW->>Orch: Start Workflow: "ReferralSaga"
    
    rect rgb(240, 248, 255)
        note right of Orch: SAGA Step 1: Ingestion
        Orch->>Ag_D: Parse & Verify (Bundle)
        Ag_D->>Ag_D: NLP Extraction (Risk Factors)
        Ag_D->>DB: INSERT Cases (New)
        Ag_D-->>Orch: Case Created (UUID)
    end
    
    rect rgb(255, 250, 240)
        note right of Orch: SAGA Step 2: Assessment
        Orch->>Ag_D: Calc Initial CMS Level
        Ag_D->>Ag_D: Rule Engine (ADL/IADL)
        Ag_D-->>Orch: Est. CMS Level = 7
    end

    rect rgb(240, 255, 240)
        note right of Orch: SAGA Step 3: Pre-Matching
        Orch->>Ag_M: Find Candidates (CMS=7, Loc=...)
        Ag_M->>DB: GIS Query (Providers in 5km)
        Ag_M->>Ag_M: Calculate Weighted Score
        Ag_M-->>Orch: Top 3 Providers List
    end

    Orch->>Line: Push Notification (To Family)
    Line-->>HIS: 201 Created (Case ID)
```

---

## SEQ-02: 智慧輔具租賃額度試算 (Smart Rental Subsidy)

**情境**: 個管師為案主申請「電動移位機」，系統檢查 3 年 6 萬額度。

```mermaid
sequenceDiagram
    autonumber

    participant CM as A單位 Portal
    participant GW as API Gateway
    participant Ag_R as Resource Agent
    participant Ledger as Subsidy Ledger (DB)
    participant Ext_AT as 輔具租賃平台 (External)

    CM->>GW: POST /subsidy/calculate (Item=LIFT_01)
    GW->>Ag_R: Calc Subsidy
    
    Ag_R->>Ledger: Query History (Last 3 Years)
    Ledger-->>Ag_R: [Txn1: Purchase Wheelchair, Txn2: Rental Bed]
    
    Ag_R->>Ag_R: Check Exclusion Rule
    note right of Ag_R: Rule: 購置輪椅不排斥移位機 -> Pass
    
    Ag_R->>Ag_R: Calc Remaining Balance
    note right of Ag_R: 60,000 - Used(12,000) = 48,000
    
    Ag_R->>Ext_AT: Query Inventory (LIFT_01)
    Ext_AT-->>Ag_R: Stock=2, Price=3000/mo
    
    Ag_R->>Ag_R: Optimization
    note right of Ag_R: Subsidize 100% (Low Income)
    
    Ag_R-->>CM: Eligible (Sub=3000, Self=0)
    
    CM->>GW: Confirm Order
    GW->>Ag_R: Reserve Quota
    Ag_R->>Ledger: INSERT (Type=RESERVE, Amt=3000)
```

---

## SEQ-03: 跨機構身分聯邦驗證 (FIdM Authentication)

**情境**: A 單位個管師使用「醫事人員卡 (HCA)」登入系統。

```mermaid
sequenceDiagram
    autonumber
    
    participant User as 個管師 (Browser)
    participant App as Web App
    participant Auth as Auth Service (OIDC)
    participant HCA as 衛福部 HCA IdP

    User->>App: Click "Login with HCA"
    App->>Auth: OIDC Authorize Request
    Auth->>User: Redirect to HCA Gateway
    
    User->>HCA: Insert Card & Enter PIN
    HCA->>HCA: Verify Certificate (PKI)
    HCA->>Auth: ID Token (Signed JWT)
    
    Auth->>Auth: Validate Signature
    Auth->>Auth: Map HCA ID to System Role
    note right of Auth: ROC ID -> UserId -> Role:CASE_MANAGER
    
    Auth->>App: Access Token + Refresh Token
    App->>User: Login Success (Dashboard)
```
