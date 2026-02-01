## 1. Sequence Diagram - Life Service - quotes (/request call platform->life)

```
Client → LifeResultsAPI.getRequest
           ↓
       LifeValidationServiceImpl.validatePremiumRequest
           ↓
       LIService.getLifeRequestValidator (by category)
           ↓
       AbstractLifeRequestValidator.validatePremiumRequest
           ↓
       ┌─── Check LifeABTestingConfig ───┐
       │                                  │
   isActive=true                    isActive=false
       ↓                                  ↓
   useNewImplementation            useOldImplementation
       ↓                                  ↓
   HTTP → Master Service V2         DB → lifeRequestValidatorNM
       ↓                                  ↓
       └────────── Merge Results ─────────┘
                        ↓
                Rider Validation (Rule Engine)
                        ↓
                Offer Validation (DB)
                        ↓
                LifeCacheService.cache
                        ↓
                Return ValidationResponse
```
---


## 2. Database Collections

| Collection | Purpose | Used By |
|:---|:---|:---|
| `lifeProductMaster` | Product definitions, options, status | Both Old & New |
| `lifeRequestValidatorNM` | Validation rules (age, term, maturity) | Old Impl |
| `lifeRequestValidator` | Legacy validation rules | Old Impl |
| `lifeRiderMaster` | Rider definitions | Rider Validation |
| `lifeOfferMaster` | Offer definitions | Offer Validation |
| `lifeAgeIncomeMultiplierValidator` | SA multiplier based on age/income | Old Impl |
| `lifeRecommondatedGrid` | Recommendation logic | Entry Point |
| `LifeABTestingConfig` | Feature flags (Old/New selection) | Strategy Selection |

---

## 3. Old vs New Implementation

### Selection Logic

```text
IF LifeABTestingConfig.isActive() == true
    → Use NEW Implementation (Product Studio)
ELSE
    → Use OLD Implementation (Local DB)
    → ALSO run NEW in parallel (for comparison/migration)
```

**Key Configuration:** `LifeABTestingConfig` with `featureName = "lifeValidator"`.

---

### New Implementation Flow

```
1. fetchEnabledProductMaster (from lifeProductMaster)
     ↓
2. abProductMasterHelper.productMasterFromStudio (filter by Studio conditions)
     ↓
3. Apply Defaults (PayoutType, Frequency)
     ↓
4. postFilterProductMaster (filter by PayoutMode)
     ↓
5. applyProductVisibilityFilter (broker-specific rules)
     ↓
6. planFeatureService.filterProductsForFeatures
     ↓
7. LifeRequestValidatorHelper.applyLifeValidator
     ↓
8. NearestMatchValidatorsImpl → HTTP POST to Master Service V2
     ↓
9. Rider & Offer Validation
     ↓
10. Return ValidProductRowsMapper list
```

**External Dependency:** Master Service V2 (`${master.service.v2.host}/api/v1/life-validator`)

---

### Old Implementation Flow

```
1. getProductMastersForCategory (from lifeProductMaster)
     ↓
2. InsurerConfigurationService.getEnabled (filter disabled insurers)
     ↓
3. abProductMasterHelper.productMasterFromDB
     ↓
4. Apply Defaults & Filters
     ↓
5. fetchEligibleProductRows → 4 PARALLEL DB QUERIES:
     ├── Main Criteria (PT_NVEST_FLAG=0, PPT_NVEST_FLAG=0)
     ├── PT Formula (PT_NVEST_FLAG=1)
     ├── PPT Formula (PPT_NVEST_FLAG=1)
     └── PT+PPT Formula (both=1)
     ↓
6. getCombinedRequestValidatorNMRows (merge & dedupe)
     ↓
7. filterLifeRequestValidatorRows (cross-check with ProductMaster)
     ↓
8. Rider & Offer Validation
     ↓
9. Return ValidProductRowsMapper list
```

**DB Dependency:** `lifeRequestValidatorNM` collection.

---

### The 4 Parallel Queries Explained

| Query | PT Flag | PPT Flag | Formula |
|:---|:---|:---|:---|
| Main | 0 | 0 | PT and PPT are exact values |
| PT Formula | 1 | 0 | `Actual_PT = DB_PT - EntryAge` |
| PPT Formula | 0 | 1 | `Actual_PPT = DB_PPT - EntryAge` |
| Both Formula | 1 | 1 | Both PT and PPT are calculated |

**Fields Validated:**
- `MinEntryAge` ≤ UserAge ≤ `MaxEntryAge`
- `MinMaturityAge` ≤ (UserAge + PT) ≤ `MaxMaturityAge`
- `Currency` match
- `Input_Policy_Term` and `Input_Premium_Payment_Term` match

---
