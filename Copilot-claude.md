# API Documentation: `/api/tm-life/v0/premiums/request/validate`

## 1. Overview

| Property | Value |
|:---|:---|
| **Endpoint** | `/api/tm-life/v0/premiums/request/validate` |
| **Method** | `POST` |
| **Controller** | `LifeResultsAPI` |
| **Handler Method** | `getRequest(PremiumRequest, ServerHttpRequest)` |
| **Returns** | `Mono<ValidationResponse>` |

**Purpose:** Validates a life insurance premium request and returns eligible products/plans based on user inputs (age, income, sum assured, policy term, etc.).

---

## 2. Complete Function Reference

### Entry Point

| Function | Class | Purpose |
|:---|:---|:---|
| `getRequest` | `LifeResultsAPI` | API entry point. Determines broker, sets defaults, triggers validation. |

### Service Layer

| Function | Class | Purpose |
|:---|:---|:---|
| `validatePremiumRequest` | `LifeValidationServiceImpl` | Orchestrates validation. Gets validator by category and calls it. Caches results. |
| `getLifeRequestValidator` | `LIService` | Returns the appropriate `ILifeRequestValidator` impl based on product category (TERM, ULIP, etc.). |
| `cacheValidProductRowsMapper` | `LifeCacheService` | Caches validated products for future requests. |

### Validator Layer (AbstractLifeRequestValidator)

| Function | Purpose |
|:---|:---|
| `validatePremiumRequest` | Main validation orchestrator. Decides Old vs New implementation. |
| `useNewImplementation` | Executes validation via external Product Studio service. |
| `useOldImplementation` | Executes validation via local DB queries. |
| `getProductMastersForCategory` | Fetches product masters from DB filtered by category. |
| `calculateDefaultParameters` | Calculates default premium, term based on product rules. |
| `postFilterProductMaster` | Filters products by Payout Mode (Income/Lumpsum). |
| `applyProductVisibilityFilter` | Filters products based on broker-specific visibility rules. |
| `fetchEligibleProductRows` | **(Old Impl)** Queries `lifeRequestValidatorNM` for valid products. |
| `createMainValidationQueryCriterias` | Builds MongoDB criteria for fixed PT/PPT validation. |
| `createPTFormulaValidationCriterias` | Builds criteria where PT is dynamic (PT - Age). |
| `createPPTFormulaValidationCriterias` | Builds criteria where PPT is dynamic (PPT - Age). |
| `createPTPPTFormulaValidationCriterias` | Builds criteria where both PT and PPT are dynamic. |
| `getCombinedRequestValidatorNMRows` | Merges results from 4 parallel queries, removes duplicates. |

### Helper Layer

| Function | Class | Purpose |
|:---|:---|:---|
| `applyLifeValidator` | `LifeRequestValidatorHelper` | **(New Impl)** Calls external service, then validates riders/offers. |
| `getNearestMatchRange` | `NearestMatchValidatorsImpl` | Calls Product Management Service for nearest match validation. |
| `fetchLifeValidator` | `ProductManagementServiceImpl` | HTTP POST to Master Service V2 (`/api/v1/life-validator`). |

### Rider & Offer Validation

| Function | Class | Purpose |
|:---|:---|:---|
| `fetchEligibleRiderRows` | `LifeRiderValidator` | Validates riders via Rule Engine. |
| `fetchEligibleOfferRows` | `LifeOfferValidator` | Validates offers from `lifeOfferMaster`. |

---

## 3. Database Collections

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

## 4. Old vs New Implementation

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

## 5. Error Handling & Validation Failures

### Scenario 1: Exception in Validation

```java
// In LifeResultsAPI.getRequest
.onErrorReturn(LifeResponseUtils.createValidationResponse(null, request))
```

**Result:** Returns empty `ValidationResponse` with original request. No products returned.

### Scenario 2: No Matching Products

**Result:** Returns `ValidationResponse` with empty product list. Frontend displays "No plans available".

### Scenario 3: External Service Failure (New Impl)

```java
// In NearestMatchValidatorsImpl
.onErrorResume(error -> Mono.just(new ArrayList<>()))
```

**Result:** Returns empty list. Validation continues but no Studio products are returned.

### Scenario 4: Rule Engine Failure (Rider Validation)

**Result:** Riders are skipped. Base products still returned without rider options.

### Scenario 5: AB Testing Config Missing

**Result:** Falls back to Old Implementation only.

---

## 6. Request/Response Format

### Request Body (PremiumRequest)

```json
{
  "_id": "unique-request-id",
  "lifePremiumRequest": {
    "entryAge": 30,
    "policyTerm": 20,
    "premiumPaymentTerm": 10,
    "sumAssured": 10000000,
    "annualIncome": 1000000,
    "paymentFrequency": "YEARLY",
    "categories": ["TERM"],
    "payoutType": "LUMPSUM",
    "planType": "TERM"
  },
  "clientId": "partner-id",
  "tenant": "B2C"
}
```

### Response (ValidationResponse)

```json
{
  "status": "SUCCESS",
  "validProducts": {
    "PRODUCT_CODE_1": [
      {
        "lifeRequestValidatorNM": { ... },
        "productMaster": { ... },
        "riderMeta": { ... },
        "offerMeta": { ... }
      }
    ]
  },
  "requestId": "unique-request-id"
}
```

---

## 7. Sequence Diagram

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
