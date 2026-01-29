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
---

## 8. DETAILED: Rider Fetching & Validation

### Overview: What are Riders?

**Riders** are optional insurance covers that can be added to a base life insurance product:
- **Accidental Death Benefit (ADB)** - Extra payout on accidental death
- **Disability Waiver of Premium (DWP)** - Premium waived if insured becomes disabled
- **Critical Illness Rider (CI)** - Lump sum on critical illness diagnosis
- **Spouse Rider (SR)** - Extended coverage to spouse
- **Family Income Rider (FIR)** - Regular income to family on death

---

### 8.1 Rider Fetching Flow (Step-by-Step)

#### Step 1: Retrieve Eligible Riders for Product

**Function:** `LifeRiderValidator.fetchEligibleRiderRows(productCode, request)`

**Location:** `src/main/java/com/turtlemint/life/validator/service/impl/LifeRiderValidator.java`

```java
public Mono<List<LifeRiderMeta>> fetchEligibleRiderRows(
    String productCode,
    PremiumRequest request) {
    
    // Fetch all riders associated with this product from DB
    return lifeRiderMasterDao.findByProductCode(productCode)
        .flatMapMany(riders -> Flux.fromIterable(riders))
        .filterWhen(rider -> validateRiderEligibility(rider, request))
        .collectList();
}
```

**Query to lifeRiderMaster:**

```json
Query: { "Product_Code": "HDFC_CLICK_2_PROTECT" }

Sample Result:
[
  {
    "_id": "ObjectId(...)",
    "Product_Code": "HDFC_CLICK_2_PROTECT",
    "Rider_Code": "ADB_RIDER",
    "Rider_Name": "Accidental Death Benefit",
    "Rider_Type": "OPTIONAL",
    "Min_Age": 18,
    "Max_Age": 60,
    "Min_SA": 100000,
    "Max_SA": 5000000,
    "Min_Term": 5,
    "Max_Term": 40,
    "Premium_Rate": 2.50,          // Per 1000 SA
    "Required_Min_Base_SA": 1000000,
    "Is_Active": true
  },
  {
    "_id": "ObjectId(...)",
    "Product_Code": "HDFC_CLICK_2_PROTECT",
    "Rider_Code": "CI_RIDER",
    "Rider_Name": "Critical Illness",
    "Rider_Type": "OPTIONAL",
    ...
  }
]
```

---

#### Step 2: Validate Each Rider Against Request Parameters

**Function:** `validateRiderEligibility(rider, request)`

**Validation Conditions:**

| Condition | Check | Source |
|:---|:---|:---|
| **Age Eligibility** | `Min_Age ≤ UserAge ≤ Max_Age` | lifeRiderMaster |
| **Term Eligibility** | `Min_Term ≤ PolicyTerm ≤ Max_Term` | lifeRiderMaster |
| **Sum Assured Range** | `Min_SA ≤ RiderSA ≤ Max_SA` | lifeRiderMaster |
| **Base SA Requirement** | `BaseProductSA ≥ Required_Min_Base_SA` | lifeRiderMaster |
| **Rider Status** | `Is_Active = true` | lifeRiderMaster |
| **Gender Restrictions** | Optional gender checks | lifeRiderMaster (if applicable) |
| **Smoking Status** | Optional smoking restrictions | lifeRiderMaster (if applicable) |
| **Rule Engine Validation** | Custom business rules | Rule Engine Service |

**Code Example:**

```java
private boolean validateRiderEligibility(LifeRiderMeta rider, PremiumRequest request) {
    LifePremiumRequest lpr = request.getLifePremiumRequest();
    
    // Check 1: Rider is active
    if (!rider.isActive()) {
        return false;
    }
    
    // Check 2: Age within range
    if (lpr.getEntryAge() < rider.getMinAge() || 
        lpr.getEntryAge() > rider.getMaxAge()) {
        return false;
    }
    
    // Check 3: Policy term within range
    if (lpr.getPolicyTerm() < rider.getMinTerm() || 
        lpr.getPolicyTerm() > rider.getMaxTerm()) {
        return false;
    }
    
    // Check 4: Sum assured within rider limits
    Long riderSA = calculateRiderSA(lpr, rider);
    if (riderSA < rider.getMinSA() || riderSA > rider.getMaxSA()) {
        return false;
    }
    
    // Check 5: Base product SA meets minimum requirement
    if (lpr.getSumAssured() < rider.getRequiredMinBaseSA()) {
        return false;
    }
    
    return true;
}
```

---

#### Step 3: Apply Rule Engine Validation (If Configured)

**Function:** `ruleEngineService.validateRiderRules(rider, request)`

**Purpose:** Apply complex business rules beyond simple range checks

**Example Rules:**

```json
{
  "riderCode": "DWP_RIDER",
  "rules": [
    {
      "name": "Max Disability Waiver Age",
      "condition": "IF rider.code = 'DWP' AND age > 50 THEN reject",
      "severity": "HARD"
    },
    {
      "name": "Smoking Restriction",
      "condition": "IF rider.code = 'CI' AND smoking = 'HEAVY_SMOKER' THEN reject",
      "severity": "HARD"
    },
    {
      "name": "Income Multiple Check",
      "condition": "IF rider.sa > (income * 15) THEN warn",
      "severity": "SOFT"
    }
  ]
}
```

**Code:**

```java
public boolean applyRuleEngineValidation(LifeRiderMeta rider, PremiumRequest request) {
    try {
        return ruleEngineService.validateRider(
            rider.getRiderCode(),
            request
        );
    } catch (Exception e) {
        // If rule engine fails, skip this rider
        logger.error("Rule Engine validation failed for rider: " + rider.getRiderCode(), e);
        return false;  // Rider is excluded
    }
}
```

---

#### Step 4: Calculate Rider Premium

**Function:** `calculateRiderPremium(rider, request)`

**Formula:**

```
Rider Premium = (Rider SA / 1000) × Premium Rate × Duration Factor

Example:
Rider SA = 1,000,000
Premium Rate = 2.50 (per 1000)
Rider Premium = (1,000,000 / 1000) × 2.50 = 2,500 per annum
```

**Code:**

```java
private Long calculateRiderPremium(LifeRiderMeta rider, PremiumRequest request) {
    LifePremiumRequest lpr = request.getLifePremiumRequest();
    
    Long riderSA = calculateRiderSA(lpr, rider);
    Long baseRate = rider.getPremiumRate();  // Per 1000
    
    // Calculate annual premium
    long riderPremium = (riderSA / 1000) * baseRate;
    
    // Apply duration factor (longer terms = higher premium)
    if (lpr.getPolicyTerm() > 30) {
        riderPremium = (long)(riderPremium * 1.05);  // 5% increase
    }
    
    return riderPremium;
}
```

---

#### Step 5: Fetch Rider Metadata & Display Information

**Function:** `enrichRiderMetaData(eligibleRiders)`

**Enrichment Details:**

```java
public LifeRiderMetaDto enrichRiderMetaData(LifeRiderMeta rider) {
    return LifeRiderMetaDto.builder()
        .riderCode(rider.getRiderCode())
        .riderName(rider.getRiderName())
        .description(rider.getDescription())
        .benefits(rider.getBenefits())              // List of benefit descriptions
        .exclusions(rider.getExclusions())          // List of exclusions
        .maxAge(rider.getMaxAge())                  // UI: Show age restriction
        .minAge(rider.getMinAge())
        .minSA(rider.getMinSA())                    // UI: Show SA range
        .maxSA(rider.getMaxSA())
        .premiumRate(rider.getPremiumRate())        // UI: Show cost per 1000
        .riderType(rider.getRiderType())            // MANDATORY / OPTIONAL
        .faqUrl(rider.getFaqUrl())                  // Help link
        .isRecommended(rider.isRecommended())       // UI: Badge for recommended riders
        .build();
}
```

---

### 8.2 Adding Riders to ValidProductRowsMapper

**Function:** `ValidProductRowsMapper.setRiderMeta(List<LifeRiderMeta>)`

**Location in Response Building:**

```java
// In AbstractLifeRequestValidator.validatePremiumRequest

// After product validation completes
ValidProductRowsMapper productRow = new ValidProductRowsMapper();
productRow.setProductMaster(productMaster);
productRow.setLifeRequestValidatorNM(validator);

// Fetch and add eligible riders
List<LifeRiderMeta> eligibleRiders = lifeRiderValidator
    .fetchEligibleRiderRows(productCode, request)
    .block();  // Convert Mono to blocking call
    
productRow.setRiderMeta(eligibleRiders);

// Fetch and add offers
List<LifeOfferMeta> activeOffers = lifeOfferValidator
    .fetchEligibleOfferRows(productCode, request)
    .block();
    
productRow.setOfferMeta(activeOffers);

return productRow;
```

---

### 8.3 Riders in Response JSON

**Sample Response with Riders:**

```json
{
  "status": "SUCCESS",
  "validProducts": {
    "HDFC_CLICK_2_PROTECT": [
      {
        "productMaster": {
          "productCode": "HDFC_CLICK_2_PROTECT",
          "productName": "HDFC Life Click 2 Protect Plus",
          "insurerCode": "HDFC"
        },
        "riderMeta": [
          {
            "riderCode": "ADB_RIDER",
            "riderName": "Accidental Death Benefit",
            "riderType": "OPTIONAL",
            "minAge": 18,
            "maxAge": 60,
            "minSA": 100000,
            "maxSA": 5000000,
            "premiumRate": 2.50,
            "description": "Extra payout if death is accidental",
            "isRecommended": true,
            "faqUrl": "https://example.com/faq/adb"
          },
          {
            "riderCode": "CI_RIDER",
            "riderName": "Critical Illness",
            "riderType": "OPTIONAL",
            "minAge": 18,
            "maxAge": 65,
            "minSA": 250000,
            "maxSA": 2000000,
            "premiumRate": 5.75,
            "description": "Lump sum payout on critical illness",
            "isRecommended": false,
            "faqUrl": "https://example.com/faq/ci"
          }
        ],
        "offerMeta": [ ... ]
      }
    ]
  }
}
```

---

### 8.4 Rider Validation Conditions Summary

#### Mandatory Conditions (Hard Stop)

| Condition | If Failed | Action |
|:---|:---|:---|
| Rider Active Status | false | Exclude rider |
| Age Range | Outside range | Exclude rider |
| Policy Term Range | Outside range | Exclude rider |
| Sum Assured Range | Outside range | Exclude rider |
| Min Base SA Requirement | Not met | Exclude rider |
| Rule Engine Validation | Hard fail | Exclude rider |

#### Optional Conditions (Warnings)

| Condition | If Failed | Action |
|:---|:---|:---|
| Income Multiple Check | Exceeded | Include rider + warning |
| Recommendation Flag | Not set | Include rider (no special handling) |

---

### 8.5 Database Collections for Riders

#### lifeRiderMaster Collection

**Used for:** Defining which riders can be added to a product

```json
{
  "_id": "ObjectId(...)",
  "Product_Code": "HDFC_CLICK_2_PROTECT",
  "Rider_Code": "ADB_RIDER",
  "Rider_Name": "Accidental Death Benefit",
  "Description": "Extra payout if insured dies in accident",
  "Rider_Type": "OPTIONAL",           // OPTIONAL or MANDATORY
  "Min_Age": 18,
  "Max_Age": 60,
  "Min_SA": 100000,
  "Max_SA": 5000000,
  "Min_Term": 5,
  "Max_Term": 40,
  "Premium_Rate": 2.50,               // Per 1000 of rider SA
  "Required_Min_Base_SA": 1000000,
  "Is_Active": true,
  "Benefits": [
    "100% of Base Sum Assured on accidental death",
    "No waiting period"
  ],
  "Exclusions": [
    "Death by suicide within 12 months",
    "Death due to illegal activities"
  ],
  "Is_Recommended": true,
  "FAQ_URL": "https://example.com/faq/adb",
  "Created_Date": "2025-01-01",
  "Updated_Date": "2026-01-29"
}
```

#### Rule Engine Configuration (External)

**Used for:** Complex business rule validation

```json
{
  "riderCode": "DWP_RIDER",
  "rules": [
    {
      "ruleId": "DWP_001",
      "description": "DWP not available after age 55",
      "condition": "age > 55",
      "action": "REJECT",
      "severity": "HARD"
    },
    {
      "ruleId": "DWP_002",
      "description": "Smoking restriction for DWP",
      "condition": "smokingStatus = 'HEAVY_SMOKER'",
      "action": "REJECT",
      "severity": "HARD"
    }
  ]
}
```

---

### 8.6 Rider Validation Sequence

```
Request for Product Validation
        ↓
Product passes all checks ✓
        ↓
[RIDER STAGE]
        ↓
For each available rider in lifeRiderMaster:
        ↓
┌─── Check 1: Is Active? ───┐
│   YES → Continue          │
│   NO → Exclude rider      │
└───────────────────────────┘
        ↓
┌─── Check 2: Age Range? ───┐
│   YES → Continue          │
│   NO → Exclude rider      │
└───────────────────────────┘
        ↓
┌─ Check 3: Term Range? ────┐
│   YES → Continue          │
│   NO → Exclude rider      │
└───────────────────────────┘
        ↓
┌─ Check 4: SA Range? ──────┐
│   YES → Continue          │
│   NO → Exclude rider      │
└───────────────────────────┘
        ↓
┌─ Check 5: Min Base SA? ───┐
│   YES → Continue          │
│   NO → Exclude rider      │
└───────────────────────────┘
        ↓
┌─ Check 6: Rule Engine ────┐
│   PASS → Continue         │
│   FAIL → Exclude rider    │
│   ERROR → Log + Exclude   │
└───────────────────────────┘
        ↓
┌─ Calculate Premium ───────┐
│   Formula applied         │
│   Duration factors added  │
└───────────────────────────┘
        ↓
┌─ Enrich Metadata ─────────┐
│   Add descriptions        │
│   Add benefits/exclusions │
│   Add FAQ links           │
└───────────────────────────┘
        ↓
Add to ValidProductRowsMapper.riderMeta
        ↓
Continue to next rider
        ↓
All riders processed
        ↓
Return ValidProductRowsMapper with eligible riders
```

---

### 8.7 Error Handling in Rider Validation

| Scenario | Handling | Result |
|:---|:---|:---|
| **DB Query Fails** | Catch & log exception | Product returned WITHOUT riders |
| **Rule Engine Timeout** | Timeout handler | Product returned WITHOUT riders |
| **Empty Rider List** | No riders found in DB | Product returned without riderMeta |
| **Partial Rider Failure** | One rider fails validation | That rider excluded, others included |
| **Invalid Premium Calculation** | Calculation error | Rider excluded with error log |

**Code Example:**

```java
try {
    List<LifeRiderMeta> riders = lifeRiderValidator
        .fetchEligibleRiderRows(productCode, request)
        .timeout(Duration.ofSeconds(5))            // 5 sec timeout
        .block();
    
    productRow.setRiderMeta(riders);
} catch (TimeoutException e) {
    logger.warn("Rider validation timeout for product: " + productCode);
    productRow.setRiderMeta(new ArrayList<>());    // Empty riders
    // Product still returned, just without rider options
} catch (Exception e) {
    logger.error("Rider validation failed", e);
    productRow.setRiderMeta(new ArrayList<>());
    // Product still returned
}
```

---



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
