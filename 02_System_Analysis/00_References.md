# 系統分析參考資源 (SA References)

## 1. 架構與設計模式 (Architecture & Design Patterns)
*   **Microservices Patterns**: Saga Pattern (Orchestration-based) 用於處理長期運行的分散式交易。
    *   參考: [Microservices.io - Saga Pattern](https://microservices.io/patterns/data/saga.html)
*   **Event-Driven Architecture**: 使用 Apache Kafka 作為事件骨幹。
    *   參考: [Confluent - Event Streaming Platform](https://www.confluent.io/)
*   **Temporal.io**: Workflow as Code 的協調器引擎。
    *   參考: [Temporal Documentation](https://docs.temporal.io/)

## 2. 技術標準與協議 (Technical Standards)
*   **HL7 FHIR R4**: 醫療資料交換標準。
    *   Spec: [HL7 FHIR Release 4](http://hl7.org/fhir/R4/)
    *   Validator: [HAPI FHIR Validator](https://hapifhir.io/hapi-fhir/docs/validation/validation_introduction.html)
*   **TW Core IG**: 台灣核心實作指引 (CI Build)。
    *   Source: [TW Core IG 官網](https://twcore.mohw.gov.tw/ig/twcore/)
*   **OIDC / OAuth 2.0**: 身分認證授權標準。
    *   RFC 6749 (OAuth 2.0), OpenID Connect Core 1.0.

## 3. 資料庫與儲存 (Database & Storage)
*   **PostgreSQL Partitioning**: 處理大量交易資料的分割策略。
    *   Docs: [PostgreSQL 16 Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html)
*   **PostGIS**: 地理資訊系統擴充 (用於 10km 照顧圈媒合)。
    *   Docs: [PostGIS Geometry](https://postgis.net/docs/using_postgis_dbmanagement.html)
*   **Qdrant**: 向量資料庫 (用於可能的語意化媒合推薦)。
    *   Docs: [Qdrant Documentation](https://qdrant.tech/documentation/)

## 4. 安全與合規 (Security & Compliance)
*   **ISO 27001**: 資訊安全管理系統 (ISMS) 標準。
    *   重點: A.12.3 (加密), A.12.4 (日誌紀錄).
*   **NIST SP 800-207**: 零信任架構 (Zero Trust Architecture) 指引。
    *   Source: [NIST ZTA](https://csrc.nist.gov/publications/detail/sp/800-207/final)
