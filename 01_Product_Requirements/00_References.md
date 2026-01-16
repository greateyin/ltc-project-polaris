# 參考資源列表 (References)

## 1. 政策與法規 (Policy & Regulations)
*   **[衛福部] 長照 3.0 政策藍圖 (2026-2030)**
    *   關鍵目標：出院銜接 < 2 天、智慧科技導入、醫養合一。
    *   來源：衛福部官方新聞稿 / 長照專區。
*   **[衛福部] 智慧科技輔具租賃補助計畫**
    *   實施日期：2026/07/01
    *   補助額度：每 3 年最高 60,000 元 (與身障/長照購置補助擇一)。
    *   適用範疇：移位腰帶、移位機、智慧床墊、與其他安全監測設備等 5 大類。
*   **[衛福部] 長期照顧服務申請及給付辦法**
    *   定義 CMS (Care Management System) 2-8 級之給付額度與部分負擔比率。

## 2. 技術標準 (Technical Standards)
*   **FHIR TW Core IG (臺灣核心實作指引)**
    *   版本：CI Build (持續更新)
    *   關鍵資源：`TW Core Patient`, `TW Core CarePlan`, `TW Core Practitioner`, `TW Core ServiceRequest`.
    *   來源：[FHIR TW Core IG 官網](https://twcore.mohw.gov.tw/ig/twcore/)
*   **長照自建資訊系統介接規範**
    *   包含「照顧管理資訊平台」與「長照服務費用支付審核系統」之 API 規格。
    *   來源：衛福部資訊處 / 1966 專區。

## 3. 領域知識 (Domain Knowledge)
*   **出院準備服務 (Discharge Planning)**
    *   流程：入院 24h 篩選 -> 出院前 3 天評估 -> 擬定計畫 -> 轉介長照。
    *   KPI：出院後 7 天內服務到位 (2025 目標)，逐步縮短至 2 天 (2026 目標)。
*   **長照服務項目 (Long-Term Care Services)**
    *   照顧及專業服務 (B碼/C碼)
    *   交通接送服務 (D碼)
    *   輔具及居家無障礙環境改善服務 (E碼/F碼)
    *   喘息服務 (G碼)

## 4. 系統架構參考 (System References)
*   **HL7 FHIR RESTful API**: 標準化醫療資料交換協定。
*   **Multi-Agent Systems (MAS)**: 應用於分散式資源調度與自動協商之架構模式。
