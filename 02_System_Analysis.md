# 系統分析文件 (System Analysis) - 目錄

**專案名稱：** Project Polaris (LTC-Agent Network)  
**版本：** 2.0 (Deep Dive)  
**日期：** 2026-01-16  
**架構風格：** Event-Driven Microservices (with Multi-Agent)

---

## 📄 章節列表

1.  **[系統架構設計](02_System_Analysis/01_System_Architecture.md)**
    *   邏輯視圖 (Logical View): Multi-Agent 協作層
    *   部署視圖 (Deployment View): K8s & Hybrid Cloud
    *   技術決策 (Tech Stack Decision)

2.  **[資料庫綱要設計](02_System_Analysis/02_Database_Schema.md)**
    *   實體關係圖 (ERD) - High Detail
    *   實體規格 (Entity Specs): Partitioning & Indexing
    *   指標資料表 (Coverage Gap / Workforce Load)
    *   Redis 快取策略

3.  **[安全性與合規設計](02_System_Analysis/03_Security_Compliance.md)**
    *   零信任架構 (Zero Trust)
    *   資料加密 (KMS & Column-Level)
    *   公平與可近性合規 (Equity & Accessibility)
    *   FHIR TW Core 合規驗證

4.  **[關鍵流程時序圖](02_System_Analysis/04_Sequence_Diagrams.md)**
    *   SEQ-01: 出院轉介與自動分案 (Saga Pattern)
    *   SEQ-02: 智慧輔具租賃額度試算
    *   SEQ-03: 跨機構身分聯邦驗證 (FIdM)

5.  **[介面整合規格](02_System_Analysis/05_Interface_Specs.md)**
    *   IF-01: 衛福部 CMS 介接
    *   IF-02: 支付審核系統批次上傳
    *   IF-03: 醫院 HIS FHIR Gateway

6.  **[參考資源](02_System_Analysis/00_References.md)** (New!)
