# API Flow Documentation: Premium Validation Service

**Endpoint**: `/api/tm-life/v0/premiums/request/validate`  
**Method**: `POST`  
**Controller**: `LifeResultsAPI`  
**Implementation**: Product Management Service Based

---

## Table of Contents

- [Overview](#overview)
- [Complete Sequential Flow](#complete-sequential-flow)
  - [Step 1: API Entry Point](#step-1-api-entry-point)
  - [Step 2: Validation Service Entry](#step-2-validation-service-entry)
  - [Step 3: Abstract Validator - Main Orchestration](#step-3-abstract-validator---main-orchestration)
  - [Step 4: New Implementation - Product Fetching](#step-4-new-implementation---product-fetching)
  - [Step 5: AB Testing & Studio Filtering](#step-5-ab-testing--studio-filtering)
  - [Step 11: Core Validation - Nearest Match](#step-11-core-validation---nearest-match)
  - [Step 14: Combine Results](#step-14-combine-results)
- [Data Transformation Summary](#data-transformation-summary)
- [Database/External Service Summary](#databaseexternal-service-summary)

---

## Overview

This document provides a comprehensive breakdown of the premium validation API flow, including:
- Complete sequential flow with input/output details
- Line-by-line code explanations
- Data transformation examples
- External API calls and database queries
- Cache operations and their effects

---

## Complete Sequential Flow

### Step 1: API Entry Point

**Function**: `LifeResultsAPI.getRequest`  
**File**: `LifeResultsAPI.java`

#### Input

```json
{
  "_id": "request_id_123",
  "currency": "INR",
  "lifePremiumRequest": {
    "dob": "1990-01-01",
    "policyTerm": 20,
    "premiumPaymentTerm": 15,
    "premium": 10000,
    "sumAssured": 5000000,
    "paymentFrequency": 12,
    "categories": ["TERM"],
    "planType": "TERM",
    "insurerCodes": ["HDFC", "ICICI"],
    "payoutType": "LUMPSUM"
  }
}
```

#### Code

```java
@RequestMapping(value = "/api/tm-life/v0/premiums/request/validate", 
                method = RequestMethod.POST)
@ResponseBody
public Mono<ValidationResponse> getRequest(
    @RequestBody PremiumRequest request, 
    ServerHttpRequest serverRequest) {
    
    // Determine broker from host
    BrokerConfig brokerConfig = BrokerUtils.determineBroker(
        serverRequest.getURI().getHost());
    
    // Set default currency if not provided
    Currency currency = request.getCurrency() != null 
        ? request.getCurrency() 
        : request.getLifePremiumRequest().getCurrency();
    
    if (currency == null) {
        request.setCurrency(Currency.INR);
        LifePremiumRequest lpr = request.getLifePremiumRequest();
        lpr.setCurrency(Currency.INR);
    }
    
    // Set result URL
    request.setResultURL(
        urlUtils.getQuotesUrl(request.get_id(), false, brokerConfig));
    
    // Find recommended products
    request.getLifePremiumRequest().setRecommendedProducts(
        recommendation.findRecommendationProducts(
            brokerConfig.getBroker(), 
            request.getLifePremiumRequest()));
    
    // Call validation service
    return liService.getValidationService()
        .validatePremiumRequest(request, brokerConfig);
}
```

#### Transformation Details

| Step | Processing | Input | Output |
|------|-----------|-------|--------|
| Line 22 | Determine broker from host | Host: `api.turtlemint.com` | `brokerConfig: "turtlemint"` |
| Lines 25-30 | Set default currency to INR | `currency: null` | `currency: INR` |
| Line 33 | Set result URL | `request_id_123` | `resultURL: https://...` |
| Lines 36-38 | Find recommended products | `LifePremiumRequest` | `recommendedProducts: [...]` |

#### Next Call

```
liService.getValidationService().validatePremiumRequest(request, brokerConfig)
```

---

### Step 2: Validation Service Entry

**Function**: `LifeValidationServiceImpl.validatePremiumRequest`  
**File**: `LifeValidationServiceImpl.java`

#### Input

- `PremiumRequest` + `BrokerConfig`

#### Code

```java
public Mono<ValidationResponse> validatePremiumRequest(
    LifePremiumRequest premiumRequest, 
    BrokerConfig brokerConfig, 
    boolean isFromQuote) throws LifeInsuranceException {
    
    // Extract categories from request
    List<String> categories = premiumRequest.getCategories();
    
    if (CollectionUtils.isEmpty(categories)) {
        throw new LifeInsuranceException("Categories cannot be empty");
    }
    
    // Get appropriate validator based on category
    ILifeRequestValidator lifeRequestValidator = 
        liService.getLifeRequestValidator(
            categories.iterator().next());
    
    // Delegate to category-specific validator
    return lifeRequestValidator.validatePremiumRequest(
        premiumRequest, brokerConfig, isFromQuote);
}
```

#### Transformation Details

| Step | Processing | Input | Output |
|------|-----------|-------|--------|
| Line 3 | Extract categories | `categories: ["TERM"]` | `categories: ["TERM"]` |
| Line 4-6 | Validate not empty | `categories: ["TERM"]` | Valid ✓ |
| Line 9 | Get first category | `"TERM"` | `TermRequestValidator` |
| Line 12 | Delegate to validator | Enrich object | Call to `TermRequestValidator` |

#### Next Call

```
termRequestValidator.validatePremiumRequest(premiumRequest, brokerConfig, false)
```

---

### Step 3: Abstract Validator - Main Orchestration

**Function**: `AbstractLifeRequestValidator.validatePremiumRequest`  
**File**: `AbstractLifeRequestValidator.java`

#### Input

- `PremiumRequest`, `BrokerConfig`, `doNotApplyDefaulting=false`

#### Code

```java
public Mono<ValidationResponse> validatePremiumRequest(
    PremiumRequest premiumRequest, 
    BrokerConfig brokerConfig, 
    Boolean doNotApplyDefaulting) throws LifeInsuranceException {
    
    LifePremiumRequest lifePremiumRequest = 
        premiumRequest.getLifePremiumRequest();
    
    // Calculate default parameters
    calculateDefaultParameters(lifePremiumRequest);
    // This sets: entryAge, maturityAge, policyTerm, premiumPaymentTerm
    
    // Trigger dialer events (if Turtlemint and no clientId)
    triggerDialerEvents(premiumRequest, brokerConfig);
    
    // Fetch partner details from MintPro API
    PartnerDetails partnerDetails = 
        partnerService.getMintProPartnerDetailsV2(
            brokerConfig.getClientId(), 
            brokerConfig.getBroker());
    
    // Get AB Testing configuration
    LifeABTestingConfig lifeABTestingConfig = 
        liService.getAbTestingService()
            .getLifeABTestingConfig(
                brokerConfig.getBroker(), 
                LIFE_VALIDATOR_FEATURE_NAME);
    
    // Decision: New vs Old implementation
    if (lifeABTestingConfig.isActive()) {
        return useNewImplementation(
            premiumRequest, lifePremiumRequest, productValidRowsMap, 
            lifeProductMasterList, partnerDetails, brokerConfig, 
            lifeABTestingConfig, doNotApplyDefaulting);
    } else {
        return useOldImplementation(...);
    }
}
```

#### Transformation Details

| Step | Processing | Input | Output |
|------|-----------|-------|--------|
| Line 5 | Calculate `entryAge` | DOB: `1990-01-01`, Year: `2026` | `entryAge: 36` |
| Line 5 | Calculate `maturityAge` | `entryAge: 36, policyTerm: 20` | `maturityAge: 56` |
| Line 9 | Trigger dialer events | Broker: `turtlemint`, clientId: `null` | Events queued |
| Lines 12-15 | **External API**: MintPro | `clientId, broker` | `PartnerDetails` |
| Lines 18-19 | AB testing config | `broker: "turtlemint"` | `Feature flag: ACTIVE` |
| Line 22 | Route to implementation | Flag: `ACTIVE` | `useNewImplementation()` |

#### Next Call

```
useNewImplementation(premiumRequest, lifePremiumRequest, ...)
```

---

### Step 4: New Implementation - Product Fetching

#### 4.1: Build Product Query Parameters

##### Code

```java
ProductParamBuilder productQueryParam = ProductParamBuilder.builder()
    .categoryTags(lifePremiumRequest.getCategories())
    .planType(lifePremiumRequest.getPlanType())
    .insurerCodes(lifePremiumRequest.getInsurerCodes())
    .paymentFrequency(lifePremiumRequest.getPaymentFrequency())
    .currency(lifePremiumRequest.getCurrency().name())
    .selectedPlans(lifePremiumRequest.getSelectedPlans())
    .broker(brokerConfig.getBroker())
    .build();
```

##### Transformation

```
INPUT (from request):
  categories: ["TERM"]
  planType: "TERM"
  insurerCodes: ["HDFC", "ICICI"]
  paymentFrequency: 12
  currency: INR
  selectedPlans: []
  broker: "turtlemint"

OUTPUT (ProductQueryParam):
{
  categoryTags: ["TERM"],
  planType: "TERM",
  insurerCodes: ["HDFC", "ICICI"],
  paymentFrequency: 12,
  currency: "INR",
  selectedPlans: [],
  broker: "turtlemint"
}
```

#### 4.2: Fetch Enabled Product Masters

##### Code

```java
List<LifeProductMaster> enabledProductMasters = 
    liService.getProductMasterDao()
        .fetchEnabledProductMaster(productQueryParam);
```

Calls → `ProductMasterQuery.fetchByFieldsRx()` → `ProductCatalogueService.fetchLifeProductMaster()`

##### Cache Logic

```java
// In ProductCatalogueService
String cacheKey = "LIFE_PRODUCT_CATALOGUE_" + broker;

List<LifeProductMaster> cachedProducts = 
    liService.getLifeCacheService()
        .getRx(cacheKey, LifeProductMaster.class);

if (cachedProducts != null) {
    return Mono.just(cachedProducts);  // Cache HIT
}

// Cache MISS - fetch from Product Management Service
return productManagementService.fetchProductCatalogue(request)
    .map(response -> convertToLifeProductMaster(response))
    .doOnNext(products -> {
        // Update cache with TTL
        liService.getLifeCacheService()
            .setRx(cacheKey, products, TTL);
    });
```

##### Transformation

```
INPUT: ProductQueryParam

CACHE CHECK:
  Key: "LIFE_PRODUCT_CATALOGUE_turtlemint"
  Result: MISS (first time) or HIT (subsequent calls)

IF CACHE MISS:
  External API Call: Product Management Service
  Request: {
    broker: "turtlemint",
    businessCategory: "LIFE",
    inclusions: ["PDP", "INTEGRATION_INFO"],
    productStatus: "LIVE"
  }
  Response: ProductDetailsResponse with ALL products
  Conversion: ProductDetailsResponse → List<LifeProductMaster>
  Cache Update: Store 150 products with TTL

OUTPUT: List<LifeProductMaster> with 150 products
```

#### 4.3: In-Memory Filtering

##### Code

```java
productMasterList = productMasterList.stream()
    .filter(Objects::nonNull)
    
    // FILTER 1: Currency
    .filter(productMaster -> Objects.isNull(productQueryParam.getCurrency())
        || (Objects.nonNull(productMaster.getCurrency())
        && productMaster.getCurrency()
            .equals(productQueryParam.getCurrency())))
    
    // FILTER 2: Plan Type
    .filter(productMaster -> StringUtils.isBlank(productQueryParam.getPlanType())
        || (StringUtils.isNotBlank(productMaster.getPlanType())
        && productMaster.getPlanType()
            .equalsIgnoreCase(productQueryParam.getPlanType())))
    
    // FILTER 3: Insurer Code
    .filter(productMaster -> CollectionUtils.isEmpty(productQueryParam.getInsurerCodes())
        || (StringUtils.isNotBlank(productMaster.getInsurerCode())
        && productQueryParam.getInsurerCodes()
            .contains(productMaster.getInsurerCode())))
    
    // FILTER 4: Product Code (Selected Plans)
    .filter(productMaster -> CollectionUtils.isEmpty(productQueryParam.getSelectedPlans())
        || (StringUtils.isNotBlank(productMaster.getProductCode())
        && productQueryParam.getSelectedPlans()
            .contains(productMaster.getProductCode())))
    
    // FILTER 5: Category Tags
    .filter(productMaster -> CollectionUtils.isEmpty(productQueryParam.getCategoryTags())
        || (!CollectionUtils.isEmpty(productMaster.getCategoryTags())
        && productMaster.getCategoryTags()
            .containsAll(productQueryParam.getCategoryTags())))
    
    // FILTER 6: Payment Frequency
    .filter(productMaster -> Objects.isNull(productQueryParam.getPaymentFrequency())
        || (!CollectionUtils.isEmpty(productMaster.getPaymentFrequencyModes())
        && productMaster.getPaymentFrequencyModes()
            .contains(productQueryParam.getPaymentFrequency())))
    
    .collect(Collectors.toList());
```

##### Transformation Flow

```
150 products (from cache/API)
  ↓
FILTER 1: currency == "INR"
  Removed: 5 USD/AED products
  Remaining: 145 products
  ↓
FILTER 2: planType == "TERM"
  Removed: ULIP, PENSION, ENDOWMENT products
  Remaining: 80 products
  ↓
FILTER 3: insurerCode IN ["HDFC", "ICICI"]
  Removed: SBI, LIC, other insurers
  Remaining: 25 products
  ↓
FILTER 4: productCode IN selectedPlans
  No filtering (selectedPlans is empty)
  Remaining: 25 products
  ↓
FILTER 5: categoryTags contains all ["TERM"]
  All products have TERM tag
  Remaining: 25 products
  ↓
FILTER 6: paymentFrequencyModes contains 12
  Removed: 3 products without monthly payment
  Remaining: 22 products
```

**Output**: `List<LifeProductMaster>` with 22 products

---

### Step 5: AB Testing & Studio Filtering

#### Code

```java
enabledProductMasters = abProductMasterHelper
    .productMasterFromStudio(
        brokerConfig.getBroker(), 
        enabledProductMasters);
```

##### Internal Logic

```java
// Get AB testing conditions from config
Map<String, Object> conditions = 
    lifeABTestingConfig.getConditions();

List<LifeProductMaster> whitelistedProducts = 
    new ArrayList<>();

for (LifeProductMaster product : enabledProductMasters) {
    // Create key: internalProductCode_optionCode
    String key = product.getInternalProductCode() + "_" 
        + product.getOptionCode();
    
    // Check if this product-option is whitelisted
    if (conditions.containsKey(key)) {
        whitelistedProducts.add(product);
    }
}

return whitelistedProducts;
```

#### Transformation

```
INPUT: 22 products

AB Testing Conditions Map:
{
  "HDFC_CLICK_2_PROTECT_LIFE_1": true,
  "ICICI_IPROTECT_SMART_1": true,
  "HDFC_CLICK_2_PROTECT_SUPER_2": true
}

PROCESSING:
  For each product:
    Product 1: "HDFC_CLICK_2_PROTECT_LIFE_1" → IN whitelist → KEEP
    Product 2: "HDFC_TERM_INSURANCE_1" → NOT in whitelist → REMOVE
    Product 3: "ICICI_IPROTECT_SMART_1" → IN whitelist → KEEP
    ... (19 more products removed)

OUTPUT: 3 products (only whitelisted ones)
```

**Note**: If conditions map is empty or no products match → Returns empty list → **Flow ends here with no products**

---

### Step 11: Core Validation - Nearest Match

#### 11.1: Transform to Validator NM Objects

##### Code

```java
public List<LifeRequestValidatorNM> getLifeValidatorNMFromLifeProductMaster(
    List<LifeProductMaster> enabledProductMasterRows, 
    LifeValidationRequestMapper validationRequest) {
    
    return enabledProductMasterRows.stream()
        .map(enabledProductMaster -> {
            LifeRequestValidatorNM lifeRequestValidatorNM = 
                new LifeRequestValidatorNM();
            
            // Copy all properties from LifeProductMaster
            BeanUtils.copyProperties(
                enabledProductMaster, 
                lifeRequestValidatorNM);
            
            // Convert optionCode String to Integer
            if (Objects.nonNull(enabledProductMaster.getOptionCode())
                && !enabledProductMaster.getOptionCode()
                    .equalsIgnoreCase("-1")
                && !enabledProductMaster.getOptionCode()
                    .equalsIgnoreCase("")) {
                lifeRequestValidatorNM.setOption(
                    Integer.parseInt(
                        enabledProductMaster.getOptionCode()));
            }
            
            // Set nvest flags (legacy fields, always 0)
            lifeRequestValidatorNM.setPtNvestFlag(0);
            lifeRequestValidatorNM.setPptNvestFlag(0);
            
            // Set nvest product category = planType
            lifeRequestValidatorNM.setNvestProductCategory(
                enabledProductMaster.getPlanType());
            
            // Extract matching category from tags
            String requestedCategory = 
                validationRequest.getCategories().stream()
                    .findFirst()
                    .orElseThrow(() -> new LifeServiceException(
                        "Categories is empty"));
            
            Optional<String> category = 
                enabledProductMaster.getCategoryTags().stream()
                    .filter(categoryValue -> 
                        categoryValue.equals(requestedCategory))
                    .findFirst();
            lifeRequestValidatorNM.setCategory(
                category.orElse(null));
            
            return lifeRequestValidatorNM;
        })
        .collect(Collectors.toList());
}
```

##### Transformation Example

```java
// BEFORE (LifeProductMaster)
{
  productCode: "HDFC_CLICK_2_PROTECT_LIFE",
  optionCode: "1",                      // ← String
  internalProductCode: "HDFC_CLICK_2_PROTECT_LIFE",
  planType: "TERM",
  categoryTags: ["TERM", "PROTECTION"],
  insurerCode: "HDFC",
  insurerName: "HDFC Life",
  productName: "Click 2 Protect Life",
  minAge: 18,
  maxAge: 65,
  // ... 50+ other fields
}

// PROCESSING:
  Line 10: BeanUtils.copyProperties() → Copies ALL fields
  Line 17: optionCode "1" → Integer.parseInt("1") → 1
  Line 20-21: Set ptNvestFlag = 0, pptNvestFlag = 0
  Line 24: Set nvestProductCategory = "TERM"
  Line 27-33: Extract "TERM" from categoryTags

// AFTER (LifeRequestValidatorNM)
{
  productCode: "HDFC_CLICK_2_PROTECT_LIFE",
  option: 1,                            // ← Integer (converted)
  internalProductCode: "HDFC_CLICK_2_PROTECT_LIFE",
  nvestProductCategory: "TERM",
  category: "TERM",
  ptNvestFlag: 0,
  pptNvestFlag: 0,
  insurerCode: "HDFC",
  insurerName: "HDFC Life",
  productName: "Click 2 Protect Life",
  minAge: 18,
  maxAge: 65,
  // ... all other fields copied
  // pt: null        ← NOT SET YET
  // ppt: null       ← NOT SET YET
  // score: null     ← NOT SET YET
}
```

**Output**: `List<LifeRequestValidatorNM>` with 2 objects (pt/ppt/score not set yet)

---

### Step 14: Combine Results

**Function**: `lifeRequestValidatorDataHelper.combineValidProductRiderRows`

This is the **MOST CRITICAL** function that combines all validated data into the final response structure.

#### Complete Code with Explanation

```java
public Map<String, List<ValidProductRowsMapper>> 
    combineValidProductRiderRows(
    Map<String, List<LifeRequestValidatorNM>> productValidRowsMap,
    Map<String, LifeRiderMeta> lifeRiderMetaMap,
    Map<String, LifeOfferMeta> lifeOfferMetaMap,
    Map<String, LifeProductMaster> enabledProductValidRowsMap,
    List<PlanFeatureDetails> planFeatureDetailsList,
    LifePremiumRequest lifePremiumRequest) {
    
    // Initialize result map: productCode -> List of ValidProductRowsMapper
    Map<String, List<ValidProductRowsMapper>> 
        validProductRowsMapperMap = new HashMap<>();
    
    // Iterate through each product in the validated products map
    for (Map.Entry<String, List<LifeRequestValidatorNM>> entry : 
         productValidRowsMap.entrySet()) {
        
        // Get product code (e.g., "HDFC_CLICK_2_PROTECT_LIFE")
        String productCode = entry.getKey();
        
        // For each option of this product (usually just 1)
        entry.getValue().forEach(lifeRequestValidatorNM -> {
            
            // Create unique key for product-option combination
            String key = lifeRequestValidatorNM.getProductCode() + "_" 
                + lifeRequestValidatorNM.getOption().toString();
            // Example: "HDFC_CLICK_2_PROTECT_LIFE_1"
            
            // Create ValidProductRowsMapper (container for all data)
            ValidProductRowsMapper validProductRowsMapper = 
                new ValidProductRowsMapper();
            
            // Set the validated product (with PT, PPT, score)
            validProductRowsMapper.setLifeRequestValidatorNM(
                lifeRequestValidatorNM);
            
            // Attach full product master object
            if (enabledProductValidRowsMap != null 
                && enabledProductValidRowsMap.get(key) != null) {
                validProductRowsMapper.setProductMaster(
                    enabledProductValidRowsMap.get(key));
            }
            
            // Attach plan features
            LifePlanFeatureDetailsInfo lifePlanFeatureDetailsInfo = 
                new LifePlanFeatureDetailsInfo();
            lifePlanFeatureDetailsInfo.setPlanFeatureDetailsList(
                planFeatureDetailsList);
            validProductRowsMapper.setLifePlanFeatureDetailsInfo(
                lifePlanFeatureDetailsInfo);
            
            // Populate payout information
            populatePayoutInfo(
                lifePremiumRequest, 
                validProductRowsMapper);
            
            // Populate deferment information
            populateDefermentInfo(
                lifePremiumRequest, 
                validProductRowsMapper);
            
            // Attach riders (if available for this product-option)
            if (lifeRiderMetaMap != null 
                && lifeRiderMetaMap.get(key) != null) {
                validProductRowsMapper.setRiderMeta(
                    lifeRiderMetaMap.get(key));
            }
            
            // Attach offers (if available for this product-option)
            if (lifeOfferMetaMap != null 
                && lifeOfferMetaMap.get(key) != null) {
                validProductRowsMapper.setOfferMeta(
                    lifeOfferMetaMap.get(key));
            }
            
            // Add to result map, grouped by product code
            validProductRowsMapperMap.computeIfAbsent(
                productCode, 
                k -> new ArrayList<>())
                .add(validProductRowsMapper);
        });
    }
    
    return validProductRowsMapperMap;
}
```

#### 14.1: Populate Payout Info

##### Code

```java
public void populatePayoutInfo(
    LifePremiumRequest lifePremiumRequest, 
    ValidProductRowsMapper validProductRowsMapper) {
    
    LifePayoutInfo lifePayoutInfo = new LifePayoutInfo();
    LifeProductMaster productMaster = 
        validProductRowsMapper.getProductMaster();
    
    // Set payout type from request
    lifePayoutInfo.setPayoutType(
        lifePremiumRequest.getPayoutType());
    
    // Set payout term (from request or product default)
    lifePayoutInfo.setPayoutTerm(
        populatePayoutBenefitTerm(
            lifePremiumRequest, 
            productMaster));
    
    // Set payout frequency (from request or product default)
    lifePayoutInfo.setPayoutFrequency(
        populatePayoutBenefitFrequency(
            lifePremiumRequest, 
            productMaster));
    
    validProductRowsMapper.setLifePayoutInfo(lifePayoutInfo);
}

public Integer populatePayoutBenefitTerm(
    LifePremiumRequest lifePremiumRequest, 
    LifeProductMaster lifeProductMaster) {
    
    if (lifePremiumRequest.getPayoutTerm() != null) {
        return lifePremiumRequest.getPayoutTerm();
    } else {
        return lifeProductMaster
            .getDefaultPayoutBenefitTerm();
    }
}

public LifeIncomeFrequency populatePayoutBenefitFrequency(
    LifePremiumRequest lifePremiumRequest, 
    LifeProductMaster lifeProductMaster) {
    
    if (Objects.nonNull(
        lifePremiumRequest.getPayoutFrequency())) {
        return lifePremiumRequest.getPayoutFrequency();
    } else {
        return lifeProductMaster
            .getDefaultPayoutBenefitFrequency();
    }
}
```

##### Transformation Example

```
INPUT:
  lifePremiumRequest.payoutType = "LUMPSUM"
  lifePremiumRequest.payoutTerm = null            ← Not provided
  lifePremiumRequest.payoutFrequency = null       ← Not provided
  productMaster.defaultPayoutBenefitTerm = 10
  productMaster.defaultPayoutBenefitFrequency = "YEARLY"

PROCESSING:
  Set payoutType = "LUMPSUM" (from request)
  Call populatePayoutBenefitTerm()
    → request.payoutTerm is null
    → Return productMaster.defaultPayoutBenefitTerm = 10
  Call populatePayoutBenefitFrequency()
    → request.payoutFrequency is null
    → Return productMaster.defaultPayoutBenefitFrequency = "YEARLY"

OUTPUT:
  LifePayoutInfo {
    payoutType: "LUMPSUM",          // From request
    payoutTerm: 10,                 // From product default
    payoutFrequency: "YEARLY"       // From product default
  }
```

#### 14.2: Populate Deferment Info

##### Code

```java
public void populateDefermentInfo(
    LifePremiumRequest lifePremiumRequest, 
    ValidProductRowsMapper validProductRowsMapper) {
    
    // Set default deferment period if not provided
    if (Objects.isNull(
        lifePremiumRequest.getDefermentPeriod())) {
        lifePremiumRequest.setDefermentPeriod(0);
    }
    
    LifeDefermentInfo defermentInfo = 
        new LifeDefermentInfo();
    int defermentPeriod = 
        lifePremiumRequest.getDefermentPeriod();
    defermentInfo.setDefermentPeriod(defermentPeriod);
    
    // Get current year
    int currentYear = Year.now().getValue();  // 2026
    
    // Calculate income benefit start year
    if (lifePremiumRequest.getPlanType()
        .equalsIgnoreCase(LifeConstants.PLAN_TYPE_PENSION)) {
        // For PENSION: Start year = Current year + Deferment
        defermentInfo.setIncomeBenefitStartYear(
            currentYear + defermentPeriod);
    } else if (lifePremiumRequest.getPremiumPaymentTerm() != null) {
        // For other plans: 
        // Start year = Current year + PPT + Deferment
        defermentInfo.setIncomeBenefitStartYear(
            currentYear 
            + lifePremiumRequest.getPremiumPaymentTerm() 
            + defermentPeriod);
    }
    
    validProductRowsMapper.setDefermentInfo(defermentInfo);
}
```

##### Transformation Example 1 (TERM Plan)

```
INPUT:
  planType = "TERM"
  premiumPaymentTerm = 15
  defermentPeriod = null         ← Not provided
  currentYear = 2026

PROCESSING:
  defermentPeriod is null → Set to 0
  currentYear = 2026
  planType is "TERM" (not "PENSION") → Skip PENSION logic
  premiumPaymentTerm is 15 (not null) → Execute else-if
  Calculate: 2026 + 15 + 0 = 2041

OUTPUT:
  LifeDefermentInfo {
    defermentPeriod: 0,
    incomeBenefitStartYear: 2041
  }
```

##### Transformation Example 2 (PENSION Plan)

```
INPUT:
  planType = "PENSION"
  defermentPeriod = 5
  currentYear = 2026

PROCESSING:
  defermentPeriod is 5 (not null) → No change
  currentYear = 2026
  planType is "PENSION" → Execute if block
  Calculate: 2026 + 5 = 2031

OUTPUT:
  LifeDefermentInfo {
    defermentPeriod: 5,
    incomeBenefitStartYear: 2031
  }
```

#### 14.3: Complete Transformation Output

##### Final Data Structure

```java
Map<String, List<ValidProductRowsMapper>> {
  "HDFC_CLICK_2_PROTECT_LIFE": [
    ValidProductRowsMapper {
      
      // 1. Validated product with PT/PPT/score
      lifeRequestValidatorNM: {
        productCode: "HDFC_CLICK_2_PROTECT_LIFE",
        option: 1,
        pt: 20,                    // ← Validated by nearest match
        ppt: 15,                   // ← Validated by nearest match
        score: 0,                  // ← Perfect match
        insurerCode: "HDFC",
        insurerName: "HDFC Life",
        // ... all other fields from product master
      },
      
      // 2. Full product master with all details
      productMaster: {
        productCode: "HDFC_CLICK_2_PROTECT_LIFE",
        productName: "Click 2 Protect Life",
        insurerCode: "HDFC",
        insurerName: "HDFC Life",
        planType: "TERM",
        minAge: 18,
        maxAge: 65,
        minSumAssured: 1000000,
        maxSumAssured: 10000000,
        // ... 50+ other fields
      },
      
      // 3. Plan features
      lifePlanFeatureDetailsInfo: {
        planFeatureDetailsList: []
      },
      
      // 4. Payout information
      lifePayoutInfo: {
        payoutType: "LUMPSUM",
        payoutTerm: 10,
        payoutFrequency: "YEARLY"
      },
      
      // 5. Deferment information
      defermentInfo: {
        defermentPeriod: 0,
        incomeBenefitStartYear: 2041
      },
      
      // 6. Riders (validated and enriched)
      riderMeta: {
        productCode: "HDFC_CLICK_2_PROTECT_LIFE",
        optionCode: "1",
        riderInfoList: [
          {
            riderCode: "HDFC_CRITICAL_ILLNESS",
            riderName: "Critical Illness Rider",
            riderCategory: "HEALTH",
            riderSumAssured: 2500000,  // ← By rule engine
            riderPolicyTerm: 20,
            riderPremiumPaymentTerm: 15,
            isValid: true,             // ← Validated by engine
            isInBuilt: false,
            isSelected: false
          }
        ]
      },
      
      // 7. Offers
      offerMeta: {
        productCode: "HDFC_CLICK_2_PROTECT_LIFE",
        optionCode: "1",
        offerInfoList: [
          {
            offerCode: "HDFC_PREMIUM_WAIVER",
            offerName: "Premium Waiver Benefit",
            offerCategory: "WAIVER",
            offerDisplayName: "Premium Waiver on CI",
            isActive: true,
            inBuilt: false,
            isSelected: false
          },
          {
            offerCode: "HDFC_LOYALTY_BONUS",
            offerName: "Loyalty Bonus",
            offerCategory: "BONUS",
            offerDisplayName: "5% Loyalty Bonus",
            isActive: true,
            inBuilt: false,
            isSelected: false
          }
        ]
      }
    }
  ],
  "ICICI_IPROTECT_SMART": [
    ValidProductRowsMapper {
      // ... similar structure with 2 riders and 1 offer
    }
  ]
}
```

---

## Data Transformation Summary

### Complete Flow with Counts

```
API Request (Incoming)
  ↓
STEP 1: Entry Point - Enrich request
  ↓
STEP 2: Validation Service - Route to validator
  ↓
STEP 3: Calculate defaults (entryAge=36, maturityAge=56)
  ↓
STEP 4.2: Fetch from Product Management Service
  150 products
  ↓
STEP 4.3: In-memory filtering (6 sequential filters)
  Filter 1 (Currency: INR):           150 → 145 products
  Filter 2 (Plan Type: TERM):         145 → 80 products
  Filter 3 (Insurer: HDFC/ICICI):     80 → 25 products
  Filter 4 (Product Code):            25 → 25 products (no change)
  Filter 5 (Category Tags: TERM):     25 → 25 products (no change)
  Filter 6 (Payment Frequency: 12):   25 → 22 products
  ↓
STEP 5: AB Testing whitelist
  22 products → 3 products (only whitelisted)
  ↓
STEP 6: Payout filtering
  3 products → 2 products
  ↓
STEP 11.1: Transform to LifeRequestValidatorNM
  2 LifeRequestValidatorNM objects (pt/ppt/score not set)
  ↓
STEP 11.3: Nearest Match Validation (external API)
  2 LifeRequestValidatorNM objects (pt=20, ppt=15, score=0)
  ↓
STEP 12: Rider Enrichment (DB + Rule Engine)
  Fetch: 4 riders from DB
  Validate: Via Rule Engine
  Result: 3 valid riders (1 rejected)
  ↓
STEP 13: Offer Enrichment (DB)
  Fetch: 3 offers from DB
  ↓
STEP 14: combineValidProductRiderRows
  Combine all 7 components:
    1. Validated products (2)
    2. Product masters (2)
    3. Plan features (0)
    4. Payout info (2)
    5. Deferment info (2)
    6. Riders (3 total)
    7. Offers (3 total)
  ↓
  2 ValidProductRowsMapper objects (complete with all data)
  ↓
Final Response (Outgoing)
```

### Key Transformations

| Transformation | From | To | Process |
|---|---|---|---|
| **Product Master to Validator NM** | `LifeProductMaster` | `LifeRequestValidatorNM` | Copies all fields via BeanUtils, converts optionCode String→Integer, sets nvest flags to 0, extracts category from tags |
| **Add Validation Results** | `LifeRequestValidatorNM` | `LifeRequestValidatorNM` (with pt/ppt/score) | External API call to nearest match validator adds pt, ppt, score |
| **Rider Master to Rider Info** | `LifeRiderMaster` | `LifeRiderInfo` (validated) | Fetches from DB, calculates sum assured via Rule Engine, validates via Rule Engine, filters invalid riders |
| **Offer Master to Offer Info** | `LifeOfferMaster` | `LifeOfferInfo` | Fetches from DB, creates offer info objects |
| **All Components to Combined Mapper** | 7 different sources | `ValidProductRowsMapper` | Combines all data sources into single response object per product |

---

## Database/External Service Summary

### External API Calls: 7 total

| Call # | Service | Function | Step | Frequency |
|--------|---------|----------|------|-----------|
| 1 | MintPro API | Get Partner Details | Step 3 | Once per request |
| 2 | Product Management Service | Fetch Product Catalogue | Step 4.2 | Once per request (cached) |
| 3 | Product Management Service | Nearest Match Validators | Step 11.3 | Once per request |
| 4 | Rule Engine | Calculate Rider Slabs | Step 12.3 | 2 calls (1 per product) |
| 5 | Rule Engine | Validate Riders | Step 12.4 | 2 calls (1 per product) |

### MongoDB Queries: 4 total

| Query # | Collection | Function | Step | Frequency |
|---------|-----------|----------|------|-----------|
| 1 | LifeRiderMaster | Fetch riders | Step 12.1 | 2 queries (1 per product) |
| 2 | LifeOfferMaster | Fetch offers | Step 13.1 | 2 queries (1 per product) |

### Cache Operations: 5-11 total

| Operation | Cache Key | Read/Write | Frequency | Notes |
|-----------|-----------|-----------|-----------|-------|
| Product Catalogue | `LIFE_PRODUCT_CATALOGUE_turtlemint` | Read/Write | 1 per request | TTL based, 150 products |
| Riders | Per product-option | Read/Write | 2 per request | TTL based |
| Offers | Per product-option | Read/Write | 2 per request | TTL based |
| Validated Results | Per request ID | Write | 1 per request | Final response cache |

---

## Appendix

### Request/Response Models

#### Sample Input Request

```json
{
  "_id": "req_12345",
  "currency": "INR",
  "lifePremiumRequest": {
    "dob": "1990-01-01",
    "policyTerm": 20,
    "premiumPaymentTerm": 15,
    "premium": 10000,
    "sumAssured": 5000000,
    "paymentFrequency": 12,
    "categories": ["TERM"],
    "planType": "TERM",
    "insurerCodes": ["HDFC", "ICICI"],
    "payoutType": "LUMPSUM",
    "defermentPeriod": 0
  }
}
```

#### Sample Output Response

```json
{
  "status": "SUCCESS",
  "requestId": "req_12345",
  "validatedProducts": [
    {
      "productCode": "HDFC_CLICK_2_PROTECT_LIFE",
      "productName": "Click 2 Protect Life",
      "insurerCode": "HDFC",
      "insurerName": "HDFC Life",
      "planType": "TERM",
      "entryAge": 36,
      "maturityAge": 56,
      "policyTerm": 20,
      "premiumPaymentTerm": 15,
      "payout": {
        "type": "LUMPSUM",
        "term": 10,
        "frequency": "YEARLY"
      },
      "deferment": {
        "period": 0,
        "incomeBenefitStartYear": 2041
      },
      "riders": [
        {
          "riderCode": "HDFC_CRITICAL_ILLNESS",
          "riderName": "Critical Illness Rider",
          "isValid": true,
          "riderSumAssured": 2500000
        }
      ],
      "offers": [
        {
          "offerCode": "HDFC_PREMIUM_WAIVER",
          "offerName": "Premium Waiver Benefit"
        }
      ]
    }
  ],
  "timestamp": "2026-02-05T10:30:00Z"
}
```

---

## Notes

- All external API calls are made asynchronously using Project Reactor Mono
- Caching significantly improves performance for repeated requests with same parameters
- AB testing feature flag controls product filtering - if inactive, uses old implementation
- Rule Engine is called per product for rider validation - expensive operation
- MongoDB queries are cached to avoid repeated DB hits
- Empty category tags result in no products returned

---

*Last Updated: February 2026*  
*API Version: v0*
