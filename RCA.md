## 1. Summary of /query API 
fetches stream of eligible products on the basis of premiumRequest

```
1. CLIENT REQUEST
   ├─ POST /api/tm-life/v0/products/query
   ├─ PremiumRequest (age, terms, sum assured, categories)
   └─ BrokerConfig (determined from hostname)

2. CONTROLLER LAYER (LifeProductsQueryApi)
   ├─ Determine broker from hostname
   ├─ Set currency
   └─ Delegate to service

3. SERVICE LAYER (LifeProductsQueryServiceImpl)
   ├─ Pre-process request (reset filters)
   ├─ Extract categories
   └─ Get category-specific validator

4. VALIDATOR LAYER (e.g., TermRequestValidator)
   ├─ Transform request to LifeValidationRequestMapper
   ├─ Calculate missing fields (entry age, maturity age)
   ├─ DB QUERY: Fetch enabled LifeProductMaster rows
   ├─ DB QUERY: Fetch matching LifeRequestValidatorNM rows
   ├─ DB QUERY: Apply nearest-match algorithm if needed
   ├─ DB QUERY: Fetch eligible LifeRiderMeta
   ├─ DB QUERY: Fetch applicable LifeOfferMeta
   ├─ Combine all data into ValidProductRowsMapper
   └─ Return Mono<Map<String, List<ValidProductRowsMapper>>>

5. RESPONSE MAPPING (LifeProductsQueryServiceImpl)
   ├─ Extract basic product info from LifeProductMaster
   ├─ Extract plan features from PlanFeatureDetails
   ├─ Extract payout info from LifePayoutInfo
   ├─ Extract deferment info from LifeDefermentInfo
   ├─ Map to LifeProductQueryResponseDto
   └─ Convert List to Flux

6. API RESPONSE
   └─ Flux<LifeProductQueryResponseDto> (Stream of eligible products)
```

---


## 2. Sequence Diagram - Life Service quotes 
(/request call platform->life) - in platform service cookies are being set and then the call is redirected to service level for further validation

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

## 4. Life-Service /results API -> Here the call goes from platform->life service. 
From the request API we get the valid products keylist. That keylist is then passed on to results api payload which is then iterated over one by one and for each productCode the call comes to life-service.

```
CLIENT REQUEST
├─ productCode: "P88"
├─ PremiumRequest (age, PT, PPT, riders, offers)
└─ BrokerConfig (determined from hostname)

↓

CONTROLLER (LifeResultsAPI)
├─ Set default currency
└─ Delegate to service

↓

SERVICE (LifeResultsServiceImpl)
├─ Fetch valid product rows for product code
├─ Merge rider selections
├─ Merge offer details
├─ Merge ULIP fund allocation
└─ For each product option:
   ├─ Get appropriate results provider
   ├─ Call getResultsFromProviderV2()
   ├─ Save rider details (async)
   ├─ Create response object
   └─ Post-process response

↓

PROVIDER LAYER (Multiple Implementations)
├─ NVEST Provider → Call NVEST API
├─ Insurer Provider → Call Insurer API (MaxLife, Bajaj, etc.)
├─ CIS Provider → Call Integrated Service
└─ Offline Provider → Use DB factors

↓

RESPONSE PROCESSING
├─ Populate error categories
├─ Add result card info
├─ Generate BI timeline
├─ Save to database
└─ Sort and classify responses

↓

FINAL RESPONSE
└─ Mono<PremiumResponse> with quotes
```

---

# 5. Comparison: `/validate` vs `/result/{productCode}` API - Understanding the Overlap


There is **significant overlap** in logic between these two APIs. Here's a detailed breakdown of what's the same and what's different.

---

## Side-by-Side Comparison

### 1. **VALIDATE API** (`/api/tm-life/v0/premiums/request/validate`)
**Location**: `LifeResultsAPI.getRequest()` - Line 89

```
INPUT: PremiumRequest (customer profile + categories)
       
PROCESS:
├─ Extract categories
├─ Get category-specific validator
├─ Call validatePremiumRequest(request, brokerConfig, true)
│  ├─ Fetch all enabled products for category
│  ├─ Apply validation rules (age, terms, sum assured)
│  ├─ Run nearest-match algorithm
│  ├─ Fetch riders metadata (all available riders)
│  ├─ Fetch offers metadata (all available offers)
│  ├─ Combine into ValidProductRowsMapper
│  └─ Returns: Map<String, List<ValidProductRowsMapper>>
│     (Multiple products with ALL possible options)
│
└─ Return ValidationResponse containing:
   ├─ All eligible products
   ├─ All available riders per product
   ├─ All available offers
   └─ All plan features

OUTPUT: List of ALL products matching customer profile
        NO premium calculation
        NO actual quotes
```

---

### 2. **RESULTS API** (`/api/tm-life/v0/premiums/result/{productCode}`)
**Location**: `LifeResultsAPI.getResults()` - Line 152

```
INPUT: productCode (specific product selected)
       + PremiumRequest (customer profile + rider selections + offer selections)
       
PROCESS:
├─ Extract productCode
├─ Extract categories
├─ Get category-specific validator
├─ Call fetchProductRowsByRequestIdAndProductCode(productCode, request, brokerConfig)
│  └─ Returns: List<ValidProductRowsMapper> for ONLY that productCode
│
├─ Merge rider selections
│  └─ Link selected riders to product rows
│
├─ Merge offer selections
│  └─ Link selected offers to product rows
│
├─ Merge ULIP fund allocation (if applicable)
│  └─ Link fund allocation choices
│
├─ For each product row:
│  ├─ Get results provider (NVEST/INSURER/CIS/OFFLINE)
│  ├─ Call getResultsFromProviderV2() 
│  │  └─ CALLS EXTERNAL INSURER API ✓ (KEY DIFFERENCE)
│  ├─ Save rider details asynchronously
│  ├─ Create response object
│  └─ Post-process (error categories, BI timeline, save to DB)
│
├─ Sort responses
├─ Classify into options
└─ Return: List<PremiumResult> with actual quotes

OUTPUT: Actual premium quotes for selected product
        Multiple quote options (different terms/frequencies)
        With actual premium amounts
```

---

## Key Differences Explained

| Aspect | VALIDATE API | RESULTS API |
|--------|--------------|------------|
| **Purpose** | Product discovery & filtering | Premium calculation & quoting |
| **Input** | Customer profile only | Customer profile + Selected product + Rider selections |
| **Product Scope** | ALL eligible products | ONE specific product |
| **External API Call** | ❌ NO | ✅ YES (Insurer/NVEST API) |
| **Rider Handling** | Fetches ALL available riders | Uses SELECTED riders only |
| **Offer Handling** | Fetches ALL available offers | Uses SELECTED offers only |
| **Premium Calculation** | ❌ NO | ✅ YES |
| **Output Complexity** | Simple (metadata only) | Complex (quotes + options) |
| **Time Duration** | ~100-500ms (DB queries only) | ~500ms-2s (includes external API) |
| **Use Case** | Shopping list of products | Getting actual quote for checkout |

---

## What's SAME Between Both?

### Shared Logic:

```
BOTH APIs do:
├─ Extract categories
├─ Get category-specific validator
├─ Calculate validation parameters (entry age, maturity age, etc.)
├─ Apply age/term/sum assured constraints
├─ Run nearest-match algorithm (if needed)
├─ Fetch LifeRequestValidatorNM rows from DB
├─ Fetch LifeProductMaster rows from DB
├─ Fetch plan features
├─ Create ValidProductRowsMapper objects
└─ Apply broker-specific business rules
```

This shared logic is why **validatePremiumRequest()** exists - it's reusable.

---

## Why This Overlap Exists (Architecture Reasoning)

### The Flow is Intentional:

```
USER JOURNEY:
    
    1. BROWSE: "Show me all life insurance products"
       └─ CALLS: /request/validate API
          └─ Gets: List of eligible products + features + riders + offers

    2. SELECT: "Show cards on UI of each valid products"
       └─ CALLS: /result/{productCode} API
          └─ Uses validation to fetch correct product rows for P88
          └─ Merges user selections
          └─ Calls insurer to get ACTUAL premium
          └─ Returns: Quoted premium amount

```
