# 7. 商業邏輯規則 (Business Logic Rules)

## 7.1 補助額度計算 (Subsidy Calculation)

### 7.1.1 照顧及專業服務 (Care Services)
依據衛福部 **長期照顧給付及支付基準** (2026版)。

| CMS 等級 | 每月給付額度 (元) | 部分負擔比率 (一般戶) | 部分負擔比率 (中低收) | 部分負擔比率 (低收) |
| :--- | :--- | :--- | :--- | :--- |
| 第 1 級 | 0 | N/A | N/A | N/A |
| 第 2 級 | 10,020 | 16% | 5% | 0% |
| 第 3 級 | 15,460 | 16% | 5% | 0% |
| 第 4 級 | 18,580 | 16% | 5% | 0% |
| 第 5 級 | 24,100 | 16% | 5% | 0% |
| 第 6 級 | 28,070 | 16% | 5% | 0% |
| 第 7 級 | 32,090 | 16% | 5% | 0% |
| 第 8 級 | 36,180 | 16% | 5% | 0% |

**邏輯規則**:
*   **CMS 1 處理**: 不納入長照給付，月額度視為 0，轉介至預防/健康促進服務。
*   **超額自付**: 若 `PlanTotalCost` > `MonthlyCap`, 超出部分由使用者全額自付。
*   **身分別判定**: 需透過 API 呼叫社會局資料庫驗證 `WelfareStatus` (一般/中低/低收)。

### 7.1.2 智慧科技輔具租賃 (Smart Device Rental) - **2026 New!**
依據「智慧科技輔具租賃補助計畫」 (2026/7/1 生效)。

*   **總額度**: 3 年內累計最高補助 **60,000 元**。
*   **計算週期**: 採「滾動式 3 年」計算 (Rolling 3-Year Window)。
*   **適用範疇**: 5 大類 (移位/移動/沐浴排泄/居家照顧床/安全看視)。
*   **排除對象 (Exclusion)**:
    *   全日住宿式機構之服務使用者。
    *   團體家屋服務之服務使用者。
*   **互斥規則 (Exclusion Rule)**:
    *   檢查 `HistoryClaims`。
    *   若過去 3 年內有 `Type = PURCHASE (購置)` 且 `Category` 屬於同類輔具 -> **不予補助** (Reject)。
*   **部分負擔 (Cost Sharing)**:
    *   一般戶：自付 30%。
    *   中低收：自付 10%。
    *   低收：自付 0%。

## 7.2 媒合權重參數 (Matching Weights)

媒合分數 `MatchScore` (0-100) 決定派案順序。

```text
MatchScore = (Distance_W * Distance_S) + 
             (Skill_W * Skill_S) + 
             (Time_W * Time_S) + 
             (Priority_Bonus)
```

| 參數 | 權重 (Weight) | 說明 |
| :--- | :--- | :--- |
| **Distance (距離)** | 30% | 基於 Google Maps API 駕駛時間。 < 10 mins = 100分。 |
| **Skill (技能)** | 20% | 證照匹配度 (e.g., AA01, BA02)。 必要證照缺失 = 直接剔除 (Score=0)。 |
| **Time (時間)** | 30% | 排班空檔匹配。 完全重疊 = 100分。 |
| **Rating (評價)** | 20% | 服務員歷史均分 (1-5星歸一化)。 |

**特殊加權 (Priority Bonus)**:
*   `CMS >= 7`: +40 分。
*   `Time = Night (20:00-08:00)`: +30 分。
*   `Time = Holiday`: +30 分。
*   `Location = Remote (偏鄉)`: +50 分 (政策保障)。

## 7.3 服務例外規則 (Exceptions)

*   **改期/取消窗口**:
    *   若 `Now <= StartTime - 24h`：允許自助改期/取消。
    *   若 `StartTime - 24h < Now <= StartTime - 2h`：允許改期，但需提示可能影響媒合與到位時效。
    *   若 `Now > StartTime - 2h`：改期/取消需人工介入 (避免頻繁爽約導致供給端損失)。
*   **爽約定義**:
    *   服務人員抵達後等待超過 15 分鐘，且無法聯繫案家 -> 視為 `NO_SHOW`。
*   **安全事件類型**:
    *   `FALL`, `VIOLENCE`, `HARASSMENT`, `SUSPECTED_ABUSE`, `MEDICAL_EMERGENCY`。
    *   高風險事件需觸發升級與稽核留存。

## 7.4 排班與共案約束 (Scheduling & Shared Care Constraints)

*   **時間窗 (Time Window)**:
    *   每筆 `ServiceOrder` 必須包含可服務時間窗 `EarliestStart` / `LatestStart`。
*   **路線成本 (Routing Cost)**:
    *   若同一居服員同日多案，系統需將移動時間納入 `Time_S`，避免不可履約排程。
*   **共案交接 (Handoff)**:
    *   若同一個案在 7 天內由 >=3 位居服員服務，需強制產生 `HandoffSummary`。
    *   下一位服務者需 `ACK` 後才可開始打卡。
