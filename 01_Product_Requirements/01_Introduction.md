# 1. 簡介與願景 (Introduction & Vision)

## 1.1 政策背景與市場現況
### 1.1.1 長照 3.0 政策 (2026-2030)
為因應 2026 年台灣邁入超高齡社會，衛福部推動「長照 3.0」，其核心變革為本系統之設計依據：
*   **醫養合一 (Medical-Care Integration)**：強調醫療與照顧的無縫銜接，目標將「住院->長照」的等待期從平均 7 天縮短至 **2 天 (2026年目標)**，遠期目標為 **0 天 (2030年)**。
*   **智慧科技導入**：2026/7/1 正式上路「智慧科技輔具租賃補助」，提供每人 3 年 6 萬元額度 (採租賃制，與購置補助擇一)，涵蓋移位機、AI 監測床墊等高單價設備。
*   **服務對象擴大**：自 2026/1/1 起，納入全年齡失智症、中風等 PAC (急性後期照護) 對象。

### 1.1.2 當前痛點 (Pain Points)
1.  **需求與人口結構壓力**：台灣將於 2025 年進入超高齡社會，65 歲以上人口占比持續上升，2070 年預估達 46.5%，需求長期擴張，照顧體系需承擔更大服務量。[國發會人口推估](https://www.ndc.gov.tw/nc_27_38548)
2.  **服務量擴張但供給壓力上升**：112 年長照 2.0 服務人數達 505,020 人，特約服務單位 8,858 處，規模快速擴張但仍面臨量能與品質壓力。[衛福部衛福年報(113) Page 71](https://service.mohw.gov.tw/ebook/dopl/113/01/files/basic-html/page71.html)
3.  **人力短缺與留任困難**：照顧服務與護理人力面臨長期缺口、留任率低與情緒勞動壓力，形成結構性人力危機。[Frontiers 2025](https://www.frontiersin.org/journals/psychology/articles/10.3389/fpsyg.2025.1545955/full) [IJERPH 2020](https://mdpi-res.com/d_attachment/ijerph/ijerph-17-09413/article_deploy/ijerph-17-09413-v2.pdf) [PMC review](https://pmc.ncbi.nlm.nih.gov/articles/PMC12718986/)
4.  **資源不均與服務覆蓋缺口**：長照資源與服務涵蓋在縣市間存在落差，部分地區有較高未覆蓋比例，形成服務斷點與等待成本。[IJERPH 2021](https://www.mdpi.com/1660-4601/18/2/605)
5.  **醫照整合與治理斷裂**：醫療與長照在制度、流程與資訊系統上仍有明顯斷點，造成出院銜接困難與重複建檔。[Ageing & Society 2020](https://www.cambridge.org/core/journals/ageing-and-society/article/abs/longterm-care-system-in-taiwan-the-2017-major-reform-and-its-challenges/BBFFA7C6F73A3EBE2771C1FDA55589B2) [BMC Geriatrics 2019](https://pmc.ncbi.nlm.nih.gov/articles/PMC6555737/)
6.  **資訊落差與使用障礙**：民眾對服務資訊掌握不足，正式資訊來源與使用率呈正相關，顯示資訊不對稱影響服務可近性。[BMC Health Services Research 2025](https://link.springer.com/article/10.1186/s12913-025-12814-6)
7.  **外籍看護與制度平行**：外籍看護形成制度外的平行照顧系統，與公部門長照資源的整合不足，影響系統一致性。[Ageing & Society 2020](https://www.cambridge.org/core/journals/ageing-and-society/article/abs/longterm-care-system-in-taiwan-the-2017-major-reform-and-its-challenges/BBFFA7C6F73A3EBE2771C1FDA55589B2)

### 1.1.3 問題證據摘要 (Evidence Snapshot)
*   **人口與需求**：2024 年 65 歲以上人口占比 19.2%，2070 年預估 46.5%，超高齡成為長期壓力來源。[國發會人口推估](https://www.ndc.gov.tw/nc_27_38548)
*   **服務規模**：112 年長照服務人數 505,020 人，服務特約單位 8,858 處，體系規模快速成長。[衛福部衛福年報(113) Page 71](https://service.mohw.gov.tw/ebook/dopl/113/01/files/basic-html/page71.html)
*   **區域缺口**：多個縣市呈現未覆蓋比例偏高的結構性落差。[IJERPH 2021](https://www.mdpi.com/1660-4601/18/2/605)

## 1.2 產品願景 (Product Vision)
**LTC-Agent Network (長照協作網)** 是一個基於 **Multi-Agent System (MAS)** 的智慧協作中台。它不取代現有的 HIS 或 CMS，而是作為「連結層 (Connective Layer)」，利用 AI Agent 自動化處理繁瑣的聯繫、媒合與追蹤工作。

**我們的承諾**：讓每一位出院長者，在返抵家門的那一刻，服務與輔具都已經準備就緒 (Service Ready upon Arrival)。

## 1.3 商業目標 (Business Goals)
1.  **政策達標**：協助地方衛生局達成「出院 2 日銜接率 > 90%」之 KPI。
2.  **市場佔有**：成為全台 60% 以上 B 單位 (服務提供者) 首選的接案調度工具。
3.  **生態系整合**：串接前 5 大輔具租賃廠商，打造最大的「智慧輔具租賃入口」。

## 1.4 專案範疇 (Scope)
*   **In-Scope**:
    *   醫院出院準備轉介 API (FHIR)。
    *   長照 A 單位個案管理輔助工具。
    *   B 單位 (居服/日照/輔具) 接案 App。
    *   家屬端進度查詢 Line Bot。
    *   智慧媒合引擎 (人力+輔具)。
*   **Out-of-Scope**:
    *   醫院內部 HIS 系統開發。
    *   長照服務費用申報系統 (僅做資料拋轉，不取代正式申報)。
