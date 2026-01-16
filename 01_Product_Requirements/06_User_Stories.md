# 6. 使用者故事 (User Stories)

## 6.1 出院轉介與評估 (Referral & Assessment)

**Story 1: 無縫轉介 (Seamless Referral)**
*   **As a** 出院準備護理師
*   **I want to** 在 HIS 系統完成出院摘要後，點選「傳送長照評估」，系統自動將 `Patient` 與 `CarePlan` 資料轉換為 FHIR 對應格式並傳送至平台
*   **So that** 我不需要重複在衛福部系統登打資料，節省 50% 行政時間，並確保資料正確性。
*   **Acceptance Criteria**:
    *   系統能接收 JSON 格式的 FHIR Bundle。
    *   若必填欄位 (如身分證號、CMS 初評分數) 缺漏，API 應回傳明確錯誤代碼。
    *   傳送成功後，HIS 畫面應顯示「已受理，案件編號: CASE-2026-XXXX」。

**Story 2: 風險自動標記 (Risk Auto-Tagging)**
*   **As a** 長照中心照管專員
*   **I want to** 在接收新案件時，系統自動標記「高風險」標籤 (如：管路留置、獨居、主要照顧者 > 80歲)
*   **So that** 我可以優先處理這些急迫案件，避免個案返家後發生意外。
*   **Acceptance Criteria**:
    *   系統解析 fhIR `Observation` 資源。
    *   若 `Barthel Index` < 30 分，標記「重度失能」。
    *   若 `SocialHistory` 顯示獨居，標記「獨居」。

## 6.2 智慧媒合 (Smart Matching)

**Story 3: 智慧輔具推薦 (Smart Device Recommendation)**
*   **As a** A單位個管師
*   **I want to** 系統根據個案的 CMS 等級 (e.g., 第 7 級) 和身體狀況 (e.g., 下肢癱瘓)，自動推薦符合「2026 租賃新制」的輔具清單 (e.g., 移位機、減壓床墊)
*   **So that** 我能快速將輔具納入照顧計畫，並確保符合 3 年 6 萬的補助資格。
*   **Acceptance Criteria**:
    *   輸入 CMS Level 7 + 失能代碼 (下肢無力)。
    *   系統輸出：推薦 "電動移位機 (B款)"，預估租金 $1500/月，補助 $1500/月。
    *   系統需自動檢查該個案過去 3 年是否有「購置」紀錄，若有則顯示「不符資格」。

**Story 4: 優先權媒合 (Priority Matching)**
*   **As a** 重度失能者家屬
*   **I want to** 我的案件因為是重症 (CMS 8級)，能被系統加權優先推播給績優的服務單位
*   **So that** 我不會因為案子難顧而被踢皮球，導致出院後無人照顧。
*   **Acceptance Criteria**:
    *   媒合演算法針對 CMS 7-8 級案件，將 `PriorityWeight` 提升 40%。
    *   B 單位看到的接案介面上，此類案件應有「急件/加給」標示 (UI Highlight)。

## 6.3 追蹤與回饋 (Tracking & Feedback)

**Story 5: 物流式進度追蹤 (Logistic-style Tracking)**
*   **As a** 家庭照顧者
*   **I want to** 在 LINE 上收到通知，顯示「輔具廠商已出發，預計 14:00 抵達」，以及「居服員 王小明 將於明日 09:00 第一次與您見面」
*   **So that** 我可以安排家裡的人留守開門，減少不確定性。
*   **Acceptance Criteria**:
    *   當 B 單位更新 `ServiceOrder` 狀態為 `EN_ROUTE` (路途中)，系統即時推播 LINE 訊息。
    *   訊息需包含服務人員照片與聯絡電話 (保護隱私前提下，提供轉接分機)。
