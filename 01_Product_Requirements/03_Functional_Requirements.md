# 3. 功能需求 (Functional Requirements) - Deep Dive

## 3.1 需求與評估 Agent (Demand & Assessment Agent)

### 問題對應 (Pain Points Mapping)
*   **轉介黑洞** -> `FR-DA-01` 自動解析與資料拋轉，減少重複登打。
*   **資訊落差** -> `FR-DA-02` 48 小時 SLA，確保案件不被遺漏。
*   **高風險案件漏接** -> 風險標記與等級推估作為優先處理依據。

### FR-DA-01: 智慧出院摘要解析 (Intelligent Discharge Parsing)
*   **Trigger**: 接收到 HIS 傳來的 `FHIR Bundle (DocumentType)`。
*   **Input**:
    *   `ClinicalImpression`: 臨床診斷文字 (e.g., "Left MCA infarction with right hemiplegia").
    *   `Observation`: Barthel Index (總分, 各項分數).
*   **Process (Logic)**:
    1.  NLP 關鍵字提取: 偵測 "NG tube" (鼻胃管), "Foley" (導尿管), "Pressure sore" (褥瘡).
    2.  風險加權計算: 若有管路 +20分, 褥瘡 +30分.
    3.  CMS 等級預估: 基於 ADL/IADL 分數對照表進行推算。
*   **Output**: 
    *   `ReferralFlag`: High/Medium/Low Risk.
    *   `TagList`: ["管路留置", "高跌倒風險"].

### FR-DA-02: 48小時黃金評估 SLA (48h Assessment SLA)
*   **Trigger**: `Referral` 建立成功 (Status=NEW).
*   **Logic**:
    *   `T0` = 出院通知時間.
    *   `Deadline` = `T0` + 48 hours.
    *   監控迴圈: 每小時檢查 Status 是否轉為 `ASSIGNED` (已分派個管師).
*   **Output**:
    *   若 `Now > Deadline - 4h`: 發送 Level 1 Alert 給衛生局督導.

## 3.2 智慧媒合 Agent (Smart Matching Agent)

### FR-MA-01: 跨區動態半徑媒合 (Dynamic Radius Matching)
*   **Trigger**: 個管師核定照顧計畫.
*   **Input**: 
    *   `ServiceLocation`: 案主經緯度.
    *   `CMS_Level`: 7 (重度).
*   **Process**:
    *   Step 1: 搜尋半徑 5km 內的 B 單位 (Score 100).
    *   Step 2: 若結果數 < 3, 自動擴大半徑至 10km (Score 80).
    *   Step 3: 若 CMS >= 7, 搜尋半徑無條件擴展至 15km, 並標記 "高額跨區津貼".
*   **Output**: Top-3 Provider List (包含預估交通時間).

### FR-MA-02: 輔具庫存即時查詢 (Device Inventory Check)
*   **Input**: 需求項目 `DEVICE_WHEELCHAIR_B` (B款輪椅).
*   **Process**:
    *   平行呼叫 (Parallel Call) 已串接之輔具商 API (`/inventory/check`).
    *   過濾 `Quantity > 0` 且 `Spec_Match = True` 的廠商.
*   **Output**: 可租賃清單 (包含: 廠商評分, 最快配送日).

## 3.3 資源與預算 Agent (Resource & Budget Agent)

### FR-RB-01: 2026 租賃新制額度試算
*   **Input**: `CaseID`, `RentalItems`=[床, 氣墊床].
*   **Process**:
    *   呼叫 `SubsidyCalculationModule` (詳見 7.1).
    *   鎖定 (Reserve) 預估額度 30 分鐘, 避免並發申請導致超額.
*   **Output**: 
    *   `SubsidyAmount`: $3000.
    *   `SelfPay`: $500.
    *   `QuotaRemaining`: $57000.

## 3.4 家庭支持 Agent (Family Support Agent)

### FR-FS-01: 照顧日誌數位簽名 (Digital Signature)
*   **Trigger**: 居服員完成服務打卡 (Clock-out).
*   **Process**:
    *   推播 Line Message 給家屬: "今日服務已完成 (沐浴, 備餐)，請確認".
    *   家屬點擊連結 -> 瀏覽服務照片 -> 手指簽名.
*   **Output**: 產生數位簽名檔, 存入 `ServiceRecord` 作為核銷憑證.

## 3.5 例外處理與升級 Agent (Exception & Escalation Agent)

### FR-EE-01: 改期/取消事件編排 (Reschedule/Cancel Orchestration)
*   **Trigger**: 收到 `RESCHEDULE_REQUEST` 或 `CANCEL_REQUEST`。
*   **Input**:
    *   `ServiceOrderID`, `RequesterRole` (FAM/CM/PROV), `ReasonCode`.
*   **Process (Logic)**:
    1.  檢查距離服務開始時間的剩餘時間 (e.g., >24h, 24h-2h, <2h).
    2.  若為改期：生成 3 個候選時段 (基於 B 單位可用性)；若不足則標記 `NEED_HUMAN_ASSIST`。
    3.  產生事件並通知相關角色：家屬、A 單位個管師、B 單位督導。
*   **Output**:
    *   `ServiceOrder` 狀態更新: `RESCHEDULE_PENDING` / `CANCELLED`。

### FR-EE-02: 爽約與安全事件通報 (No-show & Safety Incident Reporting)
*   **Trigger**: 居服員或督導回報 `NO_SHOW` / `REFUSED` / `SAFETY_INCIDENT`。
*   **Process (Logic)**:
    1.  生成 `IncidentReport` (含時間、地點、類型、描述、證據連結選填)。
    2.  若事件類型為高風險 (跌倒/暴力/性騷擾/疑似虐待) -> 觸發 Level 1 Escalation。
    3.  若 30 分鐘內未被 `ACK` -> 觸發 Level 2 Escalation (含衛生局值勤窗口)。
*   **Output**:
    *   `IncidentTicket` (可追蹤)，並寫入 `AuditTrail`。

## 3.6 生命週期與結案 Agent (Case Lifecycle Agent)

### FR-CL-01: 個案生命週期狀態機 (Case Lifecycle State Machine)
*   **Scope**: 定義個案從轉介到結案的狀態轉移。
*   **States**: `NEW` -> `ASSESSED` -> `PLAN_APPROVED` -> `SERVICE_ACTIVE` -> `SERVICE_STABLE` -> `CLOSED`。
*   **Rules**:
    *   再入院/死亡/轉機構等事件可將狀態轉為 `CLOSED` (附 `CloseReason`)。
    *   `CLOSED` 後 90 天內若再次啟動服務，需建立新 `CaseEpisode` 並保留關聯。

### FR-CL-02: 共案交接摘要 (Shared-care Handoff Summary)
*   **Trigger**: 服務結束或換人接案。
*   **Process**:
    *   從最近 N 次 `ServiceRecord` 擷取: 偏好、風險、禁忌、重要備註。
    *   產生 `HandoffSummary` 給下一位服務人員確認。
*   **Output**:
    *   `HandoffSummary` (版本化)，並要求下一位服務者 `ACK`。
