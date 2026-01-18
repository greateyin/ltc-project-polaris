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

**Story 4.5: 資源公平與等待時間透明 (Equity Transparency)**
*   **As a** 衛生局承辦人
*   **I want to** 系統提供各縣市的等待時間、覆蓋率與人力負荷指標
*   **So that** 我能及早識別高缺口區域並啟動補強資源。
*   **Acceptance Criteria**:
    *   儀表板顯示 Coverage Gap 與等待時間分布。
    *   可依縣市/鄉鎮切換。

**Story 5: 物流式進度追蹤 (Logistic-style Tracking)**
*   **As a** 家庭照顧者
*   **I want to** 在 LINE 上收到通知，顯示「輔具廠商已出發，預計 14:00 抵達」，以及「居服員 王小明 將於明日 09:00 第一次與您見面」
*   **So that** 我可以安排家裡的人留守開門，減少不確定性。
*   **Acceptance Criteria**:
    *   當 B 單位更新 `ServiceOrder` 狀態為 `EN_ROUTE` (路途中)，系統即時推播 LINE 訊息。
    *   訊息需包含服務人員照片與聯絡電話 (保護隱私前提下，提供轉接分機)。

## 6.4 身分與導入 (Onboarding & Identity)

**Story 6: B 單位快速導入 (Provider Onboarding)**
*   **As a** B 單位督導
*   **I want to** 用最少步驟完成單位資料、服務項目、服務區域、可接案時段與人員證照上傳
*   **So that** 我能在 1 天內完成上線，開始接收媒合案件。
*   **Acceptance Criteria**:
    *   系統支援建立 `ProviderProfile` (基本資料、服務類別、服務區域、聯絡窗口)。
    *   系統支援上傳並驗證居服員必要證照/技能標籤 (e.g., AA01/BA02)。
    *   系統提供「試接案模式」：可接收案件但不計入績效，期限 14 天。

**Story 7: 家屬低門檻開通 (Family Onboarding via LINE)**
*   **As a** 家庭照顧者
*   **I want to** 透過 LINE 一鍵綁定個案與主要聯絡人身分
*   **So that** 我不需要下載 App 或註冊帳號，也能查詢進度與接收通知。
*   **Acceptance Criteria**:
    *   綁定需至少 1 個驗證因子：`CaseID + OTP` 或 `ROC_ID_Last4 + DOB`。
    *   綁定成功後，家屬可檢視「案件狀態、預計服務時間、輔具配送狀態」。
    *   支援新增/移除次要照顧者 (Secondary Contacts)，並保留稽核紀錄。

## 6.5 例外與客服 (Exceptions & Support)

**Story 8: 改期與取消 (Reschedule / Cancel)**
*   **As a** 家庭照顧者
*   **I want to** 在服務前可提出改期或取消申請，並清楚知道是否會影響媒合順位或產生費用
*   **So that** 我能在病況變動或臨時外出時避免爽約與溝通成本。
*   **Acceptance Criteria**:
    *   服務前可在 LINE 送出 `RESCHEDULE_REQUEST` 或 `CANCEL_REQUEST`。
    *   系統需顯示可選時段 (至少 3 個候選時段或「需人工協助」)。
    *   取消/改期後，系統需通知 A 單位個管師與 B 單位督導。

**Story 9: 爽約與安全事件通報 (No-show & Safety Incident)**
*   **As a** 居服員
*   **I want to** 在遇到案家爽約、拒絕服務、或現場安全事件 (跌倒/暴力/性騷擾) 時，一鍵回報並啟動後續處理
*   **So that** 我能降低風險、保護自己，也讓後續銜接不中斷。
*   **Acceptance Criteria**:
    *   系統支援建立 `IncidentReport` (類型、時間、地點、描述、照片/錄音選填)。
    *   重大事件需觸發升級：通知 B 單位督導 + A 單位個管師；必要時通知衛生局。
    *   若 30 分鐘內未處理，系統觸發二次提醒。

## 6.6 共案協作 (Shared Care / Handoff)

**Story 10: 共案交接摘要 (Shared-care Handoff)**
*   **As a** B 單位督導
*   **I want to** 當同一個案由多位居服員輪流服務時，系統自動生成交接摘要 (偏好/注意事項/禁忌)
*   **So that** 共案照顧的溝通成本下降，照顧一致性提高。
*   **Acceptance Criteria**:
    *   每次服務結束後，系統更新 `HandoffSummary`：本次觀察、風險標記、下次注意事項。
    *   新接手的居服員在服務前需確認 (ack) 交接摘要。
    *   若交接摘要含高風險標記 (如褥瘡/管路)，需強制提示並要求回報確認。
