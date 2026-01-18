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

    Note over Orch, Ag_M: Equity Metric Update
    Orch->>DB: Update metrics_coverage_gap (region, wait_days)
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

---

## SEQ-04: 改期/取消與升級通報 (Reschedule/Cancel & Escalation)

**情境**: 家屬在 LINE 上提出改期；若 B 單位未在 30 分鐘內確認，系統升級通知衛生局值勤窗口。

```mermaid
sequenceDiagram
    autonumber

    participant Fam as Family (LINE)
    participant Line as Line Bot Platform
    participant GW as API Gateway
    participant Orch as Orchestrator (Temporal)
    participant Ag_EE as Exception & Escalation Agent
    participant DB as Core DB
    participant Prov as Provider Supervisor App
    participant Gov as Health Bureau On-Call

    Fam->>Line: "改期" (RESCHEDULE_REQUEST)
    Line->>GW: POST /api/v1/service-orders/{id}/reschedule
    GW->>Orch: Signal Workflow (ServiceOrderSaga)

    Orch->>Ag_EE: Validate window & propose 3 slots
    Ag_EE->>DB: INSERT incident_tickets (type=RESCHEDULE)
    Ag_EE-->>Orch: ProposedSlots

    Orch->>Prov: Notify (need confirm)
    note right of Orch: Timer starts (30 minutes)

    alt Provider confirmed within 30m
        Prov-->>Orch: Confirm Slot
        Orch->>DB: UPDATE service_orders (status=RESCHEDULED)
        Orch->>Line: Push confirmation to family
    else No response in 30m
        Orch->>Gov: Escalation Notify (Level 2)
        Orch->>DB: UPDATE incident_tickets (status=ESCALATED)
        Orch->>Line: Push "已升級處理" to family
    end
```

---

## SEQ-05: 爽約/安全事件通報與稽核 (No-show & Safety Incident)

**情境**: 居服員到場遇到案家爽約或安全事件，系統建立 IncidentReport 並啟動升級與稽核留存。

```mermaid
sequenceDiagram
    autonumber

    participant CW as Care Worker App
    participant GW as API Gateway
    participant Orch as Orchestrator (Temporal)
    participant Ag_EE as Exception & Escalation Agent
    participant DB as Core DB
    participant S3 as WORM Audit Store
    participant CM as Care Manager Portal

    CW->>GW: POST /api/v1/incidents (NO_SHOW/SAFETY_INCIDENT)
    GW->>Orch: Start Workflow (IncidentWorkflow)

    Orch->>Ag_EE: Classify severity
    Ag_EE->>DB: INSERT incident_reports
    Ag_EE->>DB: INSERT incident_tickets (status=OPEN)
    Ag_EE->>S3: Append audit event (Object Lock)

    Orch->>CM: Notify incident (requires ACK)
    alt ACK within 30m
        CM-->>Orch: ACK
        Orch->>DB: UPDATE incident_tickets (status=ACKED)
    else No ACK
        Orch->>DB: UPDATE incident_tickets (status=ESCALATED)
    end
```

---

## SEQ-06: 共案交接摘要產生與確認 (Shared-care Handoff)

**情境**: 同一個案在 7 天內由多位居服員服務，系統在每次服務結束後更新交接摘要；新接手人員需 ACK。

```mermaid
sequenceDiagram
    autonumber

    participant CW1 as Care Worker A
    participant CW2 as Care Worker B
    participant GW as API Gateway
    participant Orch as Orchestrator (Temporal)
    participant Ag_CL as Case Lifecycle Agent
    participant DB as Core DB

    CW1->>GW: POST /api/v1/service-orders/{id}/checkout
    GW->>Orch: Signal (ServiceOrderCompleted)

    Orch->>Ag_CL: Build/Update HandoffSummary
    Ag_CL->>DB: UPSERT handoff_summaries (version++)
    Ag_CL-->>Orch: SummaryId

    Orch->>GW: Require ACK before next check-in

    CW2->>GW: GET /api/v1/cases/{id}/handoff-summary
    CW2->>GW: POST /api/v1/cases/{id}/handoff-ack
    GW->>Ag_CL: Record ACK
    Ag_CL->>DB: INSERT handoff_acks
```
