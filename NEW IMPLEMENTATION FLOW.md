# NEW IMPLEMENTATION FLOW - DETAILED IN-DEPTH ANALYSIS

## 🎯 Overview

The NEW implementation uses **Product Studio** (Master Service V2) for validation instead of querying local MongoDB collections. It's a **hybrid approach** that combines database queries with external API calls.

---

## 📊 Complete Flow Diagram (NEW Implementation)

```
HTTP POST /api/tm-life/v0/premiums/request/validate
                    ↓
         LifeResultsAPI.getRequest()
                    ↓
         LifeValidationServiceImpl.validatePremiumRequest()
                    ↓
         AbstractLifeRequestValidator.validatePremiumRequest()
                    ↓
    [STEP 1] Check AB Testing Config
    ┌──────────────────────────────────┐
    │ isActive = true (NEW IMPL)        │
    └──────────────────────────────────┘
                    ↓
    [STEP 2] useNewImplementation()
                    ↓
    ┌─────────────────────────────────────────────┐
    │ A. Fetch Enabled Product Masters from DB    │
    │    (lifeProductMaster collection)           │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ B. Filter by AB Testing Conditions          │
    │    (abProductMasterHelper.productMaster     │
    │     FromStudio)                             │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ C. Apply Payout Mode Filter                 │
    │    (postFilterProductMaster)                │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ D. Apply Product Visibility Filter          │
    │    (Broker-specific visibility rules)       │
    └─────────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────────┐
    │ E. Filter by Plan Features                  │
    │    (planFeatureService.filterProducts       │
    │     ForFeatures)                            │
    └─────────────────────────────────────────────┘
                    ↓
    [STEP 3] LifeRequestValidatorHelper
             .applyLifeValidator()
                    ↓
    ┌────────────────────────────────────────┐
    │ A. Create Validation Request           │
    │    (LifeValidationRequestMapper)       │
    │    - Age, Premium, Term, SA, Income    │
    └────────────────────────────────────────┘
                    ↓
    ┌────────────────────────────────────────┐
    │ B. Call Product Management Service V2  │
    │    [EXTERNAL API CALL]                 │
    │    URL: ${master.service.v2.host}      │
    │    /api/v1/life-validator              │
    └────────────────────────────────────────┘
                    ↓
    [STEP 4] NearestMatchValidators
             .getNearestMatchRange()
                    ↓
    ┌────────────────────────────────────────┐
    │ A. productManagementService.            │
    │    fetchLifeValidator(request)          │
    │    [EXTERNAL API - Reactive Call]       │
    │    Returns: LifeNearestMatchValidators  │
    │    with scoring information             │
    └────────────────────────────────────────┘
                    ↓
    ┌────────────────────────────────────────┐
    │ B. Parse Response & Compute Nearest     │
    │    Match (NearestMatchValidatorsImpl)    │
    │    - Iterate through validators         │
    │    - Check LifeValidatorOptions         │
    │    - Check ProductVariants              │
    │    - Filter by Score (non-null)         │
    │    - Rank by nearest match              │
    └────────────────────────────────────────┘
                    ↓
    ┌────────────────────────────────────────┐
    │ C. Update LifeRequestValidatorNM        │
    │    Objects with matching scores         │
    └────────────────────────────────────────┘
                    ↓
    [STEP 5] Fetch Riders & Offers
                    ↓
    ┌────────────────────────────────────────┐
    │ A. LifeRiderValidator.                  │
    │    fetchEligibleRiderRows()             │
    │    [DB Query: lifeRiderMaster]          │
    │    Returns: Map<String, LifeRiderMeta>  │
    └────────────────────────────────────────┘
                    ↓
    ┌────────────────────────────────────────┐
    │ B. LifeOfferValidator.                  │
    │    fetchEligibleOfferRows()             │
    │    [DB Query: lifeOfferMaster]          │
    │    Returns: Map<String, LifeOfferMeta>  │
    └────────────────────────────────────────┘
                    ↓
    [STEP 6] Combine & Build Response
                    ↓
    ┌────────────────────────────────────────┐
    │ LifeRequestValidatorDataHelper.         │
    │ combineValidProductRiderRows()          │
    │ - Product Master + Validators +         │
    │   Riders + Offers + Features            │
    │ Returns: ValidProductRowsMapper         │
    └────────────────────────────────────────┘
                    ↓
    ┌────────────────────────────────────────┐
    │ LifeResponseUtils.                      │
    │ createValidationResponse()              │
    │ Returns: Mono<ValidationResponse>       │
    └────────────────────────────────────────┘
                    ↓
         HTTP 200 OK (JSON Response)
```

---

## 🔍 DETAILED STEP-BY-STEP BREAKDOWN

### STEP 1: Check AB Testing Config

```java
// AbstractLifeRequestValidator.validatePremiumRequest()

return liService.getAbTestingService()
    .fetchData(LIFE_VALIDATOR_FEATURE_NAME, brokerConfig.getBroker())
    .flatMap(configData -> {
        if (configData.isActive()) {
            // NEW Implementation
            return useNewImplementation(...)
        } else {
            // OLD Implementation (or both)
        }
    });
```

**What Happens:**
- Queries `LifeABTestingConfig` collection
- Checks if feature `"lifeValidator"` is active for the broker
- If active → Use NEW implementation

---

### STEP 2: useNewImplementation()

```java
return liService.getProductMasterDao()
    .fetchEnabledProductMaster(
        buildProductQueryParam(...)
    )
    .flatMap(enabledProductMasters -> {
        // ... filtering and validation
    });
```

#### STEP 2.A: Fetch Enabled Product Masters (DB Query)

**Database Query:**
```
Collection: lifeProductMaster
Query: buildProductQueryParam(request, lifePremiumRequest, insurers, selectedPlans, broker)
Returns: List<LifeProductMaster>
```

**Query Criteria:**
- Product code
- Broker
- Active status
- Insurer code
- Selected plans (if any)

**Sample Query:**
```mongodb
db.lifeProductMaster.find({
    "Product_Code": { $in: [...] },
    "Insurer_Code": { $in: [...] },
    "Is_Active": true,
    "Provider": { $in: ["NVEST", "NVEST-INTEGRATION"] }
})
```

**Returns:** `List<LifeProductMaster>` with product details

---

#### STEP 2.B: Apply AB Testing Conditions Filter

```java
enabledProductMasters = abProductMasterHelper
    .productMasterFromStudio(
        enabledProductMasters, 
        lifeABTestingConfig.getConditions()
    );
```

**What it does:**
- Takes whitelisted products from AB Testing config
- Filters the product masters to only include whitelisted ones
- Example: Only HDFC, ICICI products for testing group

---

#### STEP 2.C: Apply Payout Mode Filter

```java
enabledProductMasters = postFilterProductMaster(
    lifePremiumRequest, 
    enabledProductMasters
);
```

**Filters by:**
- Payout type (LUMPSUM, INCOME, etc.)
- Payment frequency (ANNUAL, MONTHLY, etc.)
- Broker-specific defaults

---

#### STEP 2.D: Apply Product Visibility Filter

```java
enabledProductMasters = applyProductVisibilityFilter(
    validationRequest, 
    enabledProductMasters
);
```

**Filters by:**
- Broker visibility rules
- Partner-specific visibility
- Geographic restrictions
- Segment-specific visibility (B2C, B2B, etc.)

---

#### STEP 2.E: Filter by Plan Features

```java
List<PlanFeatureDetails> planFeatureDetailsList = 
    planFeatureService.createPlanFeatureDetailsFromRequest(lifePremiumRequest);

enabledProductMasters = planFeatureService.filterProductsForFeatures(
    enabledProductMasters, 
    planFeatureDetailsList
);
```

**Filters by:**
- Selected plan features
- Feature compatibility with products
- Tax-saving features, rider availability, etc.

---

### STEP 3: LifeRequestValidatorHelper.applyLifeValidator()

```java
public Mono<Map<String, List<ValidProductRowsMapper>>> applyLifeValidator(
    LifeValidationRequestMapper validationRequest,
    String id, 
    LifePremiumRequest lifePremiumRequest,
    List<LifeProductMaster> enabledProductMasterRows,
    List<PlanFeatureDetails> planFeatureDetailsList,
    String broker)
```

#### STEP 3.A: Create Request Validator NM Objects

```java
List<LifeRequestValidatorNM> lifeRequestValidatorNMList = 
    lifeRequestValidatorDataHelper
        .getLifeValidatorNMFromLifeProductMaster(
            enabledProductMasterRows, 
            validationRequest
        );
```

**What it does:**
- Converts LifeProductMaster objects to LifeRequestValidatorNM objects
- Extracts validation-relevant fields
- Prepares data for external API call

**Example LifeRequestValidatorNM:**
```json
{
  "Product_Code": "HDFC_CLICK_2_PROTECT",
  "Insurer_Code": "HDFC",
  "Min_Entry_Age": 18,
  "Max_Entry_Age": 60,
  "Policy_Term": 20,
  "Premium_Payment_Term": 20,
  "Min_Premium": 12000,
  "Max_Premium": 10000000
}
```

---

#### STEP 3.B: Create External API Request

```java
ProductManagementServiceRequest request = 
    createLifeNearestMatchValidatorRequest(
        lifeRequestValidatorNMList, 
        validationRequest
    );
```

**Builds:**
```java
LifeNearestMatchValidatorScope scope = new LifeNearestMatchValidatorScope();
scope.setAge(validationRequest.getEntryAge());
scope.setPolicyTerm(validationRequest.getPolicyTerm());
scope.setPremiumPaymentTerm(validationRequest.getPremiumPaymentTerm());
scope.setPremium(validationRequest.getPremium());
scope.setPremiumPaymentMode(validationRequest.getPaymentFrequency());
scope.setSumAssuredOnDeath(validationRequest.getSumAssured());
scope.setIncome(validationRequest.getMinIncome());

ProductManagementServiceRequest request = new ProductManagementServiceRequest();
request.setProductCode(getInternalProductCodeListFromNMList(lifeRequestValidatorNMList));
request.setScope(scope);
```

**Request Payload Example:**
```json
{
  "productCode": ["HDFC_CLICK_2_PROTECT", "ICICI_ISMART_TERM"],
  "scope": {
    "age": 30,
    "policyTerm": 20,
    "premiumPaymentTerm": 10,
    "premium": 25000,
    "premiumPaymentMode": "ANNUAL",
    "sumAssuredOnDeath": 1000000,
    "income": 600000
  }
}
```

---

### STEP 4: External API Call to Master Service V2

#### STEP 4.A: Make HTTP Call

**File:** `ProductManagementServiceImpl.java`

```java
@Override
public Mono<ProductManagementServiceResponse> fetchLifeValidator(
    ProductManagementServiceRequest request) {

    String url = UriComponentsBuilder.fromHttpUrl(baseUrl)
        .path(API_PATH_V1)
        .path(LIFE_VALIDATOR_URL)
        .build()
        .toString();
    
    // URL: ${master.service.v2.host}/api/v1/life-validator
    
    Map<String, String> headersMap = new HashMap<>();
    headersMap.put(HttpHeaders.CONTENT_TYPE, "application/json");
    headersMap.put("X-API-KEY", apiKey);
    
    return rxWebClient.postAPI(
        url, 
        null, 
        headersMap, 
        request, 
        ProductManagementServiceResponse.class
    );
}
```

**External API Details:**

| Property | Value |
|----------|-------|
| **Endpoint** | `${master.service.v2.host}/api/v1/life-validator` |
| **Method** | POST |
| **Content-Type** | application/json |
| **Authentication** | X-API-KEY header |
| **Timeout** | Configurable (RxWebClient) |
| **Pattern** | Non-blocking/Reactive (Mono) |

**Configuration (application.properties):**
```properties
master.service.v2.host=https://master-service.turtlemint.com
hubIntegrationsApikey=<api-key>
```

**Request:**
```json
{
  "productCode": ["HDFC_CLICK_2_PROTECT", "ICICI_ISMART_TERM"],
  "scope": {
    "age": 30,
    "policyTerm": 20,
    "premiumPaymentTerm": 10,
    "premium": 25000,
    "premiumPaymentMode": "ANNUAL",
    "sumAssuredOnDeath": 1000000,
    "income": 600000
  }
}
```

**Response Structure:**
```json
{
  "data": [
    {
      "productCode": "HDFC_CLICK_2_PROTECT",
      "lifeValidatorOptions": [
        {
          "score": 95,
          "ptNvestFlag": 0,
          "pptNvestFlag": 0,
          "minMaturityAge": 38,
          "maxMaturityAge": 80
        }
      ],
      "productVariants": [
        {
          "variantCode": "1",
          "variantName": "Option 1",
          "lifeValidatorOptions": [...]
        }
      ]
    },
    {
      "productCode": "ICICI_ISMART_TERM",
      "lifeValidatorOptions": [...]
    }
  ],
  "status": "SUCCESS"
}
```

**What API Does:**
- Validates products against provided scope (age, term, premium, etc.)
- Returns scores for each product
- Identifies variants (options) available
- Uses nearest-match algorithm to find suitable products
- Returns empty if product is out of scope

---

#### STEP 4.B: Error Handling

```java
.onErrorResume(throwable -> {
    log.error("Error occurred in fetching life validator: {}", 
        throwable.getMessage(), throwable);
    return Mono.error(new LifeInsuranceException(throwable));
})
```

**Error Scenarios:**
1. **API Timeout** → Exception logged, empty result returned
2. **Network Error** → Exception caught, flows to error handler
3. **Invalid Request** → 400 Bad Request from external API
4. **Unauthorized** → 401 Unauthorized (API key issue)
5. **Service Down** → 503 Service Unavailable

---

### STEP 5: Process External API Response

**File:** `NearestMatchValidatorsImpl.java`

```java
return productManagementService.fetchLifeValidator(request)
    .map(response -> {
        try {
            List<LifeNearestMatchValidators> nearestMatchValidatorList = 
                mapper.convertValue(
                    response.getData(), 
                    new TypeReference<List<LifeNearestMatchValidators>>() {}
                );
            
            Map<String, NearestMatch> nearestMatchMap = 
                computeNearestMatch(nearestMatchValidatorList, validationRequest);
            
            return updateLifeRequestValidatorNM(
                lifeRequestValidatorNMList, 
                nearestMatchMap
            );
        } catch (Exception exception) {
            log.error("Exception computing nearest match: {}", 
                exception.getMessage());
            return new ArrayList<LifeRequestValidatorNM>();
        }
    })
```

#### STEP 5.A: Parse Response

```
Response → Deserialize to List<LifeNearestMatchValidators>
         → Extract validator options and product variants
```

#### STEP 5.B: Compute Nearest Match

```java
private Map<String, NearestMatch> computeNearestMatch(
    List<LifeNearestMatchValidators> nearestMatchValidatorList,
    LifeValidationRequestMapper validationRequest)
```

**Algorithm:**

1. **Iterate through each validator**
   ```
   For each LifeNearestMatchValidators object:
   ```

2. **Check if has LifeValidatorOptions (no variants)**
   ```java
   if (!CollectionUtils.isEmpty(lifeValidatorOptions)) {
       validateLifeValidatorOptions(...)
           .ifPresent(nearestMatch -> {
               nearestMatch.setInternalProductCode(...);
               createNearestMatchMap(...);
           });
   }
   ```

3. **Check if has ProductVariants (multiple options)**
   ```java
   else if (!CollectionUtils.isEmpty(lifeProductVariants)) {
       for (LifeProductVariants variant : variants) {
           validateLifeValidatorOptions(
               variant.getLifeValidatorOptions(), 
               ...
           ).ifPresent(...)
       }
   }
   ```

4. **Filter by Score (must be non-null)**
   ```java
   List<LifeValidatorOptions> applicableValidatorOption = 
       lifeValidatorOptionsList.stream()
           .filter(list -> Objects.nonNull(list.getScore()))
           .collect(Collectors.toList());
   ```

5. **Rank by Score (higher is better)**
   ```
   Sort by score descending
   Select the one with highest score
   Create NearestMatch object
   ```

#### STEP 5.C: Update LifeRequestValidatorNM

```
Map<String, NearestMatch> → Update LifeRequestValidatorNM objects
                          → Add score information
                          → Filter out non-matching products
```

---

### STEP 6: Fetch Riders & Offers (DB Queries)

#### Riders - Database Query

```java
Map<String, LifeRiderMeta> lifeRiderMetaMap = 
    liService.getLifeRiderValidator()
        .fetchEligibleRiderRows(
            productValidRowsMap, 
            validationRequest, 
            id, 
            lifePremiumRequest,
            enabledProductValidRowsMap, 
            broker
        );
```

**Database Query:**
```
Collection: lifeRiderMaster
Query: {
    "Product_Code": { $in: [validated product codes] },
    "Min_Age": { $lte: userAge },
    "Max_Age": { $gte: userAge },
    "Is_Active": true
}
```

**Returns:** `Map<String, LifeRiderMeta>` where:
- Key: Product code
- Value: List of eligible riders

---

#### Offers - Database Query

```java
Map<String, LifeOfferMeta> lifeOfferMetaMap = 
    liService.getLifeOfferValidator()
        .fetchEligibleOfferRows(
            productValidRowsMap, 
            validationRequest, 
            id, 
            lifePremiumRequest,
            enabledProductValidRowsMap, 
            broker
        );
```

**Database Query:**
```
Collection: lifeOfferMaster
Query: {
    "Product_Code": { $in: [validated product codes] },
    "Is_Active": true,
    "Offer_Valid_From": { $lte: currentDate },
    "Offer_Valid_Till": { $gte: currentDate }
}
```

**Returns:** `Map<String, LifeOfferMeta>` with active offers

---

### STEP 7: Combine & Build Response

```java
Map<String, List<ValidProductRowsMapper>> validProductRowsMapperMap = 
    lifeRequestValidatorDataHelper.combineValidProductRiderRows(
        productValidRowsMap,           // From API
        lifeRiderMetaMap,              // From DB
        lifeOfferMetaMap,              // From DB
        enabledProductValidRowsMap,    // From DB
        planFeatureDetailsList,        // From request
        lifePremiumRequest
    );
```

**Combines:**
1. Validated products from external API
2. Riders from DB
3. Offers from DB
4. Product master details from DB
5. Plan features from request

**Creates ValidProductRowsMapper:**
```json
{
  "lifeRequestValidatorNM": { ... },
  "productMaster": { ... },
  "riderMeta": [ ... ],
  "offerMeta": { ... },
  "lifePlanFeatureDetailsInfo": { ... }
}
```

---

## 📊 Database vs External API Usage

| Component | Source | Type | Purpose |
|-----------|--------|------|---------|
| **Product Masters** | DB (lifeProductMaster) | Local Query | Product details, options, features |
| **Validation Rules** | **External API** | HTTP POST | Age, term, premium eligibility checks |
| **Riders** | DB (lifeRiderMaster) | Local Query | Available riders for product |
| **Offers** | DB (lifeOfferMaster) | Local Query | Active promotional offers |
| **AB Testing Config** | DB (LifeABTestingConfig) | Local Query | Feature flags and conditions |
| **Plan Features** | DB (lifeProductMaster) | Local Query | Feature availability |

---

## 🔗 External API Details

### Master Service V2 - Life Validator Endpoint

**Purpose:** Validates product eligibility based on customer scope

**Request:**
```
POST ${master.service.v2.host}/api/v1/life-validator
Content-Type: application/json
X-API-KEY: ${hubIntegrationsApikey}

{
  "productCode": ["HDFC_001", "ICICI_001"],
  "scope": {
    "age": 30,
    "policyTerm": 20,
    "premiumPaymentTerm": 10,
    "premium": 25000,
    "premiumPaymentMode": "ANNUAL",
    "sumAssuredOnDeath": 1000000,
    "income": 600000
  }
}
```

**Response:**
```
{
  "data": [
    {
      "productCode": "HDFC_001",
      "lifeValidatorOptions": [
        {
          "score": 95,
          "ptNvestFlag": 0,
          "pptNvestFlag": 0,
          "minMaturityAge": 38,
          "maxMaturityAge": 80
        }
      ],
      "productVariants": []
    }
  ],
  "status": "SUCCESS",
  "message": "Validation successful"
}
```

**What It Returns:**
- Score for each product (0-100)
- Policy term formula flags (if PT is calculated from age)
- Premium payment term formula flags (if PPT is calculated)
- Min/max maturity age constraints
- Product variant information (if multiple options)

---

## ⚠️ Error Scenarios

### 1. External API Failure

```
NEW Implementation Exception
    ↓
onErrorResume(error -> {
    log.error("Exception in new impl: {}", error.getMessage());
    return Mono.just(new HashMap<>());  // Empty results
})
    ↓
No products returned
```

### 2. External API Returns Empty

```
API Response: { "data": [], "status": "SUCCESS" }
    ↓
nearestMatchValidatorList is empty
    ↓
computeNearestMatch returns empty map
    ↓
No products matched
```

### 3. Score is NULL (Out of Range)

```
API Response includes validators with score: null
    ↓
Filter: .filter(list -> Objects.nonNull(list.getScore()))
    ↓
Those validators excluded
    ↓
Only in-range products returned
```

---

## 🎯 Summary

### NEW Implementation Uses:

| Type | Count | Examples |
|------|-------|----------|
| **DB Queries** | 4+ | productMaster, riders, offers, AB testing |
| **External API Calls** | 1 | Master Service V2 - Life Validator |
| **Local Filters** | 5+ | Payout mode, visibility, features, etc. |
| **Transformations** | 3+ | Request mapping, response parsing, combining |

### Is It Faster Than OLD?

**NEW (with API):**
- 1 DB query (product master)
- 1 External API call (validation)
- 2-3 additional DB queries (riders, offers)
- Time: ~500-1500ms (includes network latency)

**OLD (all DB):**
- 4 parallel DB queries (validation)
- 2-3 additional DB queries (riders, offers)
- Time: ~200-500ms (all local DB)

**Trade-off:** NEW is slower but more flexible and centralized.
