# Project Polaris: Taiwan LTC 3.0 Open Specifications

![Repo: ltc-project-polaris](https://img.shields.io/badge/Repo-ltc--project--polaris-blue.svg)

![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)
![Status: Deep Dive](https://img.shields.io/badge/Status-Deep_Dive-blue)
![PRs: Welcome](https://img.shields.io/badge/PRs-Welcome-brightgreen)

## 🌟 專案願景 (Vision)
本專案旨在為台灣「長照 3.0」政策打造一套**開源、標準化、可落地**的系統規格 (Open Specifications)。透過引入 **Multi-Agent Systems (MAS)** 與 **FHIR 國際標準**，我們致力於解決醫養轉介的斷層，實現「出院即長照」的無縫銜接願景。

This project defines the open standards and technical specifications for Taiwan's Long-Term Care (LTC) 3.0 ecosystem, featuring an AI-driven Multi-Agent architecture.

---

## 📂 文件導覽 (Documentation Map)

本儲存庫分為三個核心層次，分別對應不同的閱聽眾：

### 1. [產品需求文件 (PRD)](01_Product_Requirements.md)
> **對象：** 產品經理、政策制定者、領域專家
*   [01 簡介與願景](01_Product_Requirements/01_Introduction.md): 長照 3.0 政策背景、結構性痛點與商業目標。
*   [02 使用者角色](01_Product_Requirements/02_User_Roles.md): 人物誌與使用者旅程。
*   [06 使用者故事](01_Product_Requirements/06_User_Stories.md): 端到端的業務場景與驗收標準。
*   [07 商業邏輯](01_Product_Requirements/07_Business_Logic.md): 補助額度試算規則 (2026 新制)。

### 2. [系統分析 (System Analysis)](02_System_Analysis.md)
> **對象：** 系統架構師、資安專家、DevOps
*   [01 系統架構](02_System_Analysis/01_System_Architecture.md): Event-Driven 微服務架構與技術選型。
*   [02 資料庫設計](02_System_Analysis/02_Database_Schema.md): 高效能 DB Schema 與索引策略，含公平/人力監測指標表。
*   [03 安全性與合規](02_System_Analysis/03_Security_Compliance.md): 零信任架構、KMS Encryption、可近性與稽核。
*   [04 時序圖](02_System_Analysis/04_Sequence_Diagrams.md): 複雜流程 (Saga Pattern) 的視覺化拆解。

### 3. [程式規格書 (Program Specs)](03_Program_Specifications.md)
> **對象：** 後端工程師、前端工程師
*   [01 API 規格](03_Program_Specifications/01_API_Specifications.md): REST/gRPC 介面定義，含公平監測指標 API。
*   [02 核心邏輯](03_Program_Specifications/02_Agent_Logic_SLA.md): 演算法偽代碼與 Workflow 定義。
*   [03 資料字典](03_Program_Specifications/03_Data_Dictionary.md): Enums、DB Models 與 FHIR Mapping。

---

## ⚖️ 授權條款 (License)

為了讓本規格能被政府單位、醫院資訊室、長照系統廠商 (HIS/CMS vendors) 廣泛引用與實作，本專案採用 **[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)** 授權。

### 這代表您可以：
*   **分享**: 在任何媒介以任何形式複製、發行本素材。
*   **修改**: 重混、轉換本素材，並基於本素材進行創作 (例如：開發商業軟體、撰寫新的標案規格書)。
*   **商業使用**: 您可以用於商業目的 (例如：投標政府長照系統標案)。

### 唯一條件 (Condition)：
*   **姓名標示 (Attribution)**: 您必須給予適當的表彰，提供指向本授權條款的連結，並說明本素材是否已被修改。

---

## 🤝 如何貢獻 (Contribution)

我們歡迎所有領域專家 (Domain Experts) 與開發者參與貢獻：
1.  **Issue**: 發現規格漏洞或政策更新？請提交 Issue。
2.  **Pull Request**: 想要補充 Sequence Diagram 或優化 API 定義？歡迎直接 PR。

---
*Created by LTC-Agent Network Open Source Initiative.*
