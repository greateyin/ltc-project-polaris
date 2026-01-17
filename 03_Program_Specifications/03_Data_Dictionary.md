# 3. 資料字典 (Data Dictionary) - Architect Edition

## 3.1 全域枚舉 (Global Enums)

所有微服務必須引用統一的 Enum 定義，嚴禁使用 Magic String。

### `CaseStatus`
| Code | Description | Next Allowed States |
| :--- | :--- | :--- |
| `NEW` | 新進案 (未處理) | `ASSESSING`, `VOID` |
| `ASSESSING` | 評估中 | `MATCHING`, `CLOSED` |
| `MATCHING` | 媒合中 | `ASSIGNED`, `EXPIRED` |
| `ASSIGNED` | 已派案 | `ACTIVE`, `CANCELLED` |
| `ACTIVE` | 服務中 (已簽約) | `CLOSED`, `SUSPENDED` |

### `ServiceCategory`
| Code | Name | FHIR Coding (TW Core) |
| :--- | :--- | :--- |
| `BA` | 居家照顧 | `https://pre-code.gov.tw/CodeSystem/LTC-Service-Type#BA` |
| `DA` | 日間照顧 | `https://pre-code.gov.tw/CodeSystem/LTC-Service-Type#DA` |
| `EF_R` | 輔具租賃 | `https://pre-code.gov.tw/CodeSystem/LTC-Service-Type#EF-R` |

### `MetricType`
| Code | Description |
| :--- | :--- |
| `COVERAGE_GAP` | 覆蓋率與等待時間指標 |
| `WORKFORCE_LOAD` | 人力負荷指標 |

## 3.2 資料庫對映 (DB Mapping)

### Model: `ServiceOrder` (服務單)

```typescript
// ORM Model Definition (TypeORM / Prisma style)
model ServiceOrder {
  id              String   @id @default(uuid())
  caseId          String   // FK to Case
  providerId      String?  // FK to Provider
  regionCode      String?  // 縣市/鄉鎮代碼 (公平性監測)
  
  status          OrderStatus @default(PENDING)
  
  // Financials
  totalCost       Decimal  @db.Decimal(10, 2)
  subsidyAmt      Decimal  @db.Decimal(10, 2)
  selfPayAmt      Decimal  @db.Decimal(10, 2)
  
  // Scheduling
  scheduledDate   DateTime
  durationMins    Int
  
  // Verification
  checkInTime     DateTime?
  checkOutTime    DateTime?
  gpsLat          Float?
  gpsLng          Float?
  
  // Audit
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  @@index([caseId])
  @@index([providerId, status])
  @@index([scheduledDate]) // For Scheduler Query
  @@index([regionCode])
}
```

### Model: `CoverageMetric` (公平性指標)

```typescript
model CoverageMetric {
  periodMonth     DateTime
  regionCode      String
  eligibleCount   Int
  servedCount     Int
  coverageRate    Decimal
  avgWaitDays     Decimal
  p90WaitDays     Decimal
  createdAt       DateTime @default(now())

  @@id([periodMonth, regionCode])
}
```

### Model: `WorkforceMetric` (人力負荷指標)

```typescript
model WorkforceMetric {
  periodMonth       DateTime
  regionCode        String
  staffType         String // CARE_MANAGER, SUPERVISOR
  activeCases       Int
  staffCount        Int
  avgCasesPerStaff  Decimal
  overloadRatio     Decimal
  createdAt         DateTime @default(now())

  @@id([periodMonth, regionCode, staffType])
}
```

## 3.3 FHIR Profile Mapping

本系統內部模型與 standard FHIR Resource 的對映表。

| System Field | FHIR Resource Path | Card. | Type | Note |
| :--- | :--- | :--- | :--- | :--- |
| `Case.nationalId` | `Patient.identifier[system='...nhi'].value` | 1..1 | String | 需 Hash 處理 |
| `Case.name` | `Patient.name.text` | 1..1 | String | |
| `Case.cmsLevel` | `Observation[code='CMS-Level'].valueInteger` | 0..1 | Int | |
| `Plan.instruction`| `CarePlan.description` | 0..1 | String | 出院醫囑 |
