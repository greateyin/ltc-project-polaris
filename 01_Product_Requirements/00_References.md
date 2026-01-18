# 參考資源列表 (References)

## 1. 政策與法規 (Policy & Regulations)
*   **[國發會] 中華民國人口推估（2024年至2070年）**
    *   2025 年進入超高齡社會，2070 年 65 歲以上人口占比預估 46.5%。
    *   來源：[國發會人口推估](https://www.ndc.gov.tw/nc_27_38548)
*   **[衛福部] 臺灣高齡健康與長照服務年報**
    *   年度長照服務人數與特約單位統計 (含 112 年資料)。
    *   來源：[1966 年報列表](https://1966.gov.tw/LTC/lp-6487-207.html)
*   **[衛福部] 長期照顧統計年報 (113 年)**
    *   112 年長照服務人數 505,020 人；特約單位 8,858 處。
    *   來源：[衛福年報 Page 71](https://service.mohw.gov.tw/ebook/dopl/113/01/files/basic-html/page71.html)
*   **[衛福部] 長照十年計畫 3.0**
    *   八大目標/推動策略，含「完善出院準備」與「導入智慧照顧」。
    *   來源：https://1966.gov.tw/LTC/cp-6572-85008-207.html
*   **[行政院] 長照 3.0 推動重點**
    *   目標：2026 年出院後銜接平均 2 天，2030 年達 0 天接住。
    *   來源：https://www.ey.gov.tw/Page/5A8A0CB5B41DA11E/fcf4a012-8aca-4dae-b868-3a848fcaafe7
*   **[Focus Taiwan] Gov't plans expanded long-term care to meet super-aged society challenges**
    *   描述出院後銜接等待時間：目標 2026 年 2 天、2030 年無縫。
    *   來源：https://focustaiwan.tw/society/202511270023
*   **[衛福部] 智慧科技輔具租賃補助 (115/7/1 起)**
    *   額度：每 3 年最高 60,000 元；5 大類 (移位/移動/沐浴排泄/居家照顧床/安全看視)。
    *   來源：https://1966.gov.tw/LTC/cp-6440-82812-207.html
*   **[衛福部] 長期照顧服務申請及給付辦法**
    *   定義 CMS (Care Management System) 2-8 級之給付額度與部分負擔比率。

## 2. 研究與證據 (Research & Evidence)
*   **Workforce shortage and coverage gaps (Dynamic DEA)**
    *   計量分析各縣市服務缺口與照顧資源不足。
    *   來源：https://www.mdpi.com/1660-4601/18/2/605
*   **Long-term care reform challenges**
    *   指出醫照整合與治理斷裂等問題。
    *   來源：https://www.cambridge.org/core/journals/ageing-and-society/article/abs/longterm-care-system-in-taiwan-the-2017-major-reform-and-its-challenges/BBFFA7C6F73A3EBE2771C1FDA55589B2
*   **Service integration challenges**
    *   長照體系整合與跨單位協作困難。
    *   來源：https://pmc.ncbi.nlm.nih.gov/articles/PMC6555737/
*   **Workforce retention and emotional labor**
    *   照顧人力留任與情緒勞動負擔。
    *   來源：https://www.frontiersin.org/journals/psychology/articles/10.3389/fpsyg.2025.1545955/full
*   **Information sources and service use**
    *   正式資訊來源與服務使用率關聯。
    *   來源：https://link.springer.com/article/10.1186/s12913-025-12814-6

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
