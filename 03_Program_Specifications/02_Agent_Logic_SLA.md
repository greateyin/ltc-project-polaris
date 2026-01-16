# 2. 核心邏輯與演算法 (Core Logic & Algorithms) - Architect Edition

## 2.1 出院轉介流程 (Referral Saga Workflow)
使用 **Temporal.io** 實作 Saga Pattern。

### Workflow Definition (TypeScript/Pseudocode)

```typescript
// referral-workflow.ts
export async function dischargeReferralSaga(bundle: FhirBundle): Promise<void> {
  const caseId = await activities.saveCase(bundle);

  try {
    // Step 1: 自動評估 (補償交易: deleteCase)
    const assessment = await activities.runAutoAssessment(caseId);
    
    // Step 2: 額度試算 (補償交易: releaseQuota)
    const budget = await activities.calculateInitialBudget(caseId, assessment.cmsLevel);
    
    // Step 3: 媒合廣播 (無副作用，可重試)
    await activities.broadcastToProviders({
      caseId,
      geo: assessment.geo,
      radius: 5000 // 5km
    });
    
    // Step 4: 等待接單 (Human signal)
    // 設定 48 小時 SLA Timer
    const accepted = await condition(() => isAssigned(caseId), '48h');
    
    if (!accepted) {
      // SLA Breach: 升級為急件，通知人工介入
      await activities.escalateToSupervisor(caseId);
    }

  } catch (err) {
    // 執行補償交易 (Compensating Transactions)
    await activities.rollbackCaseCreation(caseId);
    throw err;
  }
}
```

## 2.2 智慧媒合評分引擎 (Weighted Scoring Engine)
實作位置: `Matching Agent` (Python + NumPy/Pandas).

### 演算法實作細節

```python
def calculate_match_score(provider, case_req):
    """
    Input:
      provider: Dict { location, rating, skills, shift_table }
      case_req: Dict { location, cms_level, required_skills, urgent }
    Output:
      float: final_score (0-100)
    """
    
    # 1. 距離分數 (Distance Score) - Logistic Decay
    dist = haversine(provider['location'], case_req['location'])
    # 距離 0km=100分, 10km=50分, >20km=0分
    s_dist = 100 / (1 + math.exp(0.5 * (dist - 10)))
    
    # 2. 技能匹配 (Skill Match) - Jaccard Index
    req_set = set(case_req['required_skills'])
    prov_set = set(provider['skills'])
    if not req_set.issubset(prov_set):
        return 0 # Hard Constraint: 缺必要證照直接剔除
    s_skill = 100 # 因為是 Subset, 否則為 0
    
    # 3. 優先權加權 (Priority Boost)
    bonus = 0
    if case_req['cms_level'] >= 7:
        bonus += 40
    if case_req['urgent']:
        bonus += 20
        
    # 加權總分
    final_score = (0.4 * s_dist) + (0.3 * provider['rating_norm']) + (0.3 * s_skill) + bonus
    
    return min(100, final_score) # Cap at 100
```

## 2.3 補助額度規則引擎 (Subsidy Rule Engine)
實作位置: `Resource Agent` (Go).

### 規則表 (Decision Table)

| CMS Level | Item Category | Condition (身分) | Subsidy Rules |
| :--- | :--- | :--- | :--- |
| Any | `RENTAL_SMART` | Low Income | Grant = 100% Rent; Max 60k/3yr |
| Any | `RENTAL_SMART` | Mid Income | Grant = 90% Rent; Max 60k/3yr |
| Any | `RENTAL_SMART` | General | Grant = 70% Rent; Max 60k/3yr |
| 7, 8 | `DATA_PLAN` | Remote Area | Grant = 500/mo (遠距通訊補助) |

### 實作邏輯 (Go)
```go
func (e *Engine) CheckRule(ctx context.Context, claim ClaimRequest) (Result, error) {
    // 1. Check Exclusion (購置 vs 租賃)
    if hasConflictTransaction(claim.CaseID, claim.ItemType) {
        return Result{Eligible: false, Reason: "CONFLICT_PURCHASE"}, nil
    }

    // 2. Check Rolling Balance
    used := getUsedBalance(claim.CaseID, "36M") // 過去 36 個月
    remaining := 60000 - used
    if remaining <= 0 {
         return Result{Eligible: true, Subsidy: 0, SelfPay: claim.Cost}, nil
    }

    // 3. Apply Co-Pay Ratio
    ratio := getCoPayRatio(claim.WelfareStatus) // e.g., 0.7
    grant := min(claim.Cost * ratio, remaining)

    return Result{Eligible: true, Subsidy: grant, SelfPay: claim.Cost - grant}, nil
}
```
