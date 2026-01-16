# 程式規格書 (Program Specifications) - 目錄

**專案名稱：** Project Polaris (LTC-Agent Network)  
**版本：** 2.0 (Architect Edition)  
**日期：** 2026-01-16  
**設計原則：** API-First, Type-Safe, Fail-Safe

---

## 📄 章節列表

1.  **[API 介面規格](03_Program_Specifications/01_API_Specifications.md)**
    *   Public REST API (Mobile/Web)
    *   External Webhook (Line/SMS)
    *   Internal gRPC (Microservices Inter-com)

2.  **[核心邏輯與演算法](03_Program_Specifications/02_Agent_Logic_SLA.md)**
    *   Temporal Workflow 定義 (出院轉介 Saga)
    *   智慧媒合演算法實作 (Weighted Scoring)
    *   補助額度計算引擎 (Rule Engine)

3.  **[資料字典與常數](03_Program_Specifications/03_Data_Dictionary.md)**
    *   Global Enums (Service Types, CMS Levels)
    *   Database Mapping (ORM Models)
    *   FHIR Profile Mapping

4.  **[錯誤代碼與處理](03_Program_Specifications/04_Error_Codes.md)** (New!)
    *   Standard Response Format
    *   Global Error Code Table (E-001 ~ E-999)
    *   Retry Policy Definition
