# 3. 功能需求 (Functional Requirements) - Deep Dive

## 3.1 需求與評估 Agent (Demand & Assessment Agent)

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
