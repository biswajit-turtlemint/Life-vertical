# API Endpoint Documentation: `/api/tm-life/v0/premiums/result/{productCode}`

## Overview
The `/result/{productCode}` API is a **premium calculation and quote generation endpoint** that calculates the actual insurance premium for a specific product based on customer details, rider selections, offer details, and fund allocations (for ULIP products). It returns multiple quote options with different policy terms and payment frequencies.

---

## API Endpoint Details

### Route
```
POST /api/tm-life/v0/premiums/result/{productCode}
```

### Controller Class
- **File**: `LifeResultsAPI.java`
- **Package**: `com.turtlemint.life.controller`

### Request Format
```java
@PostMapping(value = "/api/tm-life/v0/premiums/result/{productCode}")
@ResponseBody
public Mono<PremiumResponse> getResults(
    @PathVariable String productCode, 
    @RequestBody PremiumRequest request, 
    ServerHttpRequest serverRequest
)
```

### Request Parameters

#### Path Parameter
- **productCode**: Unique product identifier (e.g., "P88", "P92")

#### Request Body (`PremiumRequest`)
Contains customer details and additional selections:
- `_id`: Unique request identifier
- `customerName`: Customer name
- `currency`: Currency (defaults to INR)
- **LifePremiumRequest** (nested):
  - `entryAge`: Customer's age
  - `dateOfBirth`: Customer's DOB
  - `policyTerm`: Duration of policy (years)
  - `premiumPaymentTerm`: Duration of premium payment (years)
  - `paymentFrequency`: Frequency (Monthly, Quarterly, Annual, etc.)
  - `sumAssured`: Cover amount
  - `categories`: Product category (TERM, ULIP, PAR, etc.)
  - `gender`: Customer gender
  - `riderMeta`: Selected riders with configuration
  - `offerMeta`: Selected offers/promotions
  - `ulipFundAllocationInfos`: ULIP fund allocation details (for investment-linked products)
  - Other parameters for validation

#### HTTP Headers (implicit)
- **Host**: Determines broker configuration

### Response Format
```java
Mono<PremiumResponse>
```

Returns wrapper containing:
```json
{
  "premiumRequest": { /* echoed request */ },
  "premiumResults": [
    {
      "lifePremiumResponse": {
        "productCode": "P88",
        "productName": "BAJAJ Suraksha Plus",
        "insurerCode": "BAJAJLI",
        "policyTerm": 20,
        "premiumPaymentTerm": 5,
        "paymentFrequency": "MONTHLY",
        "premium": 2500.50,
        "installmentAmount": 2500.50,
        "resultId": "unique_result_id_1",
        "status": "SUCCESS",
        "insurerStatus": "SUCCESS",
        "insurerMessage": "Quote generated successfully",
        "planType": "ULIP",
        "responseOptions": [ /* alternative quotes */ ]
      }
    }
  ]
}
```

---

## Function Flow and Architecture

### 1. **Controller Layer** (`LifeResultsAPI.getResults`)
**Location**: `/src/main/java/com/turtlemint/life/controller/LifeResultsAPI.java` (Line 152-170)

**Responsibilities**:
- Receive POST request with product code and premium request
- Determine broker configuration from hostname
- Set default currency to INR if not provided
- Delegate to service layer
- Wrap response in `PremiumResponse` object

**Key Code**:
```java
@PostMapping(value = "/api/tm-life/v0/premiums/result/{productCode}")
@ResponseBody
public Mono<PremiumResponse> getResults(@PathVariable String productCode, 
                                        @RequestBody PremiumRequest request, 
                                        ServerHttpRequest serverRequest) {
    BrokerConfig brokerConfig = BrokerUtils.determineBroker(serverRequest.getURI().getHost());
    
    // Set default currency
    Currency currency = request.getCurrency() != null ? request.getCurrency() : 
                        request.getLifePremiumRequest().getCurrency();
    if (currency == null) {
        request.setCurrency(Currency.INR);
        request.getLifePremiumRequest().setCurrency(Currency.INR);
    }
    
    return liService.getResultsService()
        .getResults(productCode, request, brokerConfig)
        .map(premiumResults -> {
            PremiumResponse premiumResponse = new PremiumResponse();
            premiumResponse.setPremiumResults(premiumResults);
            premiumResponse.setPremiumRequest(request);
            return premiumResponse;
        });
}
```

---

### 2. **Service Layer** (`LifeResultsServiceImpl.getResults`)
**Location**: `/src/main/java/com/turtlemint/life/service/impl/LifeResultsServiceImpl.java`

#### Flow Overview

```
1. Fetch valid product rows for given productCode
2. Update rider meta information if riders selected
3. Update offer meta information if offers selected
4. Update ULIP fund allocation if applicable
5. Call results provider (Insurer/NVEST/CIS API)
6. Generate multiple response options
7. Sort and classify responses
8. Post-process responses (add error categories, BI timelines)
9. Save flattened responses to DB
10. Return PremiumResult list
```

#### Step 2.1: Fetch Valid Product Rows
```java
String category = lifePremiumRequest.getCategories().iterator().next();
List<ValidProductRowsMapper> validProductRows = 
    liService.getLifeRequestValidator(category)
        .fetchProductRowsByRequestIdAndProductCode(productCode, premiumRequest, brokerConfig);
```
- Gets category-specific validator (TERM, ULIP, PAR, etc.)
- Fetches product rows matching the product code from validation rules
- Returns list of `ValidProductRowsMapper` objects containing:
  - Product master details
  - Validation rules
  - Plan features
  - Rider meta info
  - Offer meta info

#### Step 2.2: Update Rider Metadata
```java
if (premiumRequest.getLifePremiumRequest().getRiderMeta() != null) {
    List<LifeRiderMeta> riderMeta = premiumRequest.getLifePremiumRequest().getRiderMeta();
    if(riderMeta != null && riderMeta.size() > 0) {
        validProductRows = updateRiderMetaForOption(validProductRows, riderMeta);
    }
}
```
- Updates each product row with rider information matching the option code
- Links selected riders to corresponding product options
- Validates rider compatibility

#### Step 2.3: Update Offer Metadata
```java
if(Objects.nonNull(premiumRequest.getLifePremiumRequest().getOfferMeta())) {
    List<LifeOfferMeta> offerMeta = premiumRequest.getLifePremiumRequest().getOfferMeta();
    if(!CollectionUtils.isEmpty(offerMeta)) {
        validProductRows = updateOfferMetaForOption(validProductRows, offerMeta);
    }
}
```
- Associates applicable offers with product options
- Updates promotional details for premium calculation

#### Step 2.4: Update ULIP Fund Allocation (if applicable)
```java
if (premiumRequest.getLifePremiumRequest().getUlipFundAllocationInfos() != null) {
    List<LifeUlipFundAllocationInfo> fundAllocationInfo = 
        premiumRequest.getLifePremiumRequest().getUlipFundAllocationInfos();
    if (fundAllocationInfo != null && fundAllocationInfo.size() > 0) {
        validProductRows = updateFundAllocationInfoForOption(validProductRows, fundAllocationInfo);
    }
}
```
- For ULIP products: associates customer's fund allocation choices
- Example: 50% in Equity Fund, 30% in Debt Fund, 20% in Money Market Fund
- Used by ULIP providers for premium calculation

#### Step 2.5: Call Results Provider (Premium Calculation)
```java
Flux<ValidProductRowsMapper> validProductRowsMapperFlux = 
    Flux.just(validProductRows.toArray(...));

return validProductRowsMapperFlux.flatMap(validProductRowsMapper -> {
    String optionCode = validProductRowsMapper
        .getLifeRequestValidatorNM()
        .getOption()
        .toString();
    LifeProductMaster product = validProductRowsMapper.getProductMaster();
    
    InsurerConfig insurerConfig = brokerConfig
        .getInsurerConfig()
        .get(product.getInsurerCode())
        .get(VerticalEnum.LIFE.name());
    
    IResultsProvider resultsProviderService = 
        liService.getResultsProviderService(product.getProvider());
    
    return resultsProviderService
        .getResultsFromProviderV2(premiumRequest, lifePremiumRequest, 
                                  validProductRowsMapper, insurerConfig, 
                                  brokerConfig.getBroker());
});
```

**Provider Selection** (based on product provider):
- **NVEST**: Calls NVEST APIs for premium calculation
- **INSURER**: Calls insurer-specific APIs (MaxLife, Bajaj, etc.)
- **NVEST-INTEGRATION**: Calls integrated insurer framework
- **CIS**: Calls Central Integration Service
- **OFFLINE**: Uses offline calculation (premium factors)

#### Step 2.6: Generate Rider Details (Asynchronous)
```java
.doOnNext(response -> 
    lifeRidersService.generateAndSaveRidersDetailsInDb(
        premiumRequest, lifePremiumRequest, validProductRowsMapper, 
        insurerConfig, brokerConfig, resultsProviderService)
    .subscribeOn(Schedulers.parallel())
    .subscribe(...)
)
```
- Runs in parallel (non-blocking) using Schedulers.parallel()
- Generates rider premium details
- Saves rider information to database for future reference

#### Step 2.7: Create Response Object
```java
.map(lifePremiumResponse -> {
    return createResponse(lifePremiumResponse, 
        validProductRowsMapper.getLifeRequestValidatorNM(), 
        product, lifePremiumRequest, brokerConfig,
        validProductRowsMapper
            .getLifePlanFeatureDetailsInfo()
            .getPlanFeatureDetailsList());
})
```
- Maps provider response to internal `LifePremiumResponse` format
- Enriches with plan features, pricing details, etc.

#### Step 2.8: Post-Process Response
```java
.flatMap(lifePremiumResponse -> 
    postProcessLifePremiumResponse(lifePremiumRequest, lifePremiumResponse, brokerConfig)
)
```

**Post-Processing Steps**:

**a) Populate Error Category**:
```java
populateErrorCategory(lifePremiumResponse, brokerConfig)
```
- Uses attribution rules to classify error types
- Categorizes insurer errors into business-level error categories
- Example: "AGE_BELOW_MIN", "SUM_ASSURED_EXCEEDS_MAX", "PRODUCT_NOT_AVAILABLE"

**b) Add Result Card Info** (UI Display Data):
```java
addResultCardInfo(lifePremiumRequest, response, brokerConfig)
```
- Adds formatted data for UI card display
- Calculates additional metrics (IRR, returns, etc.)

**c) Set BI Timeline** (Benefit Illustration):
```java
setBiTimelineForNonTerm(response, brokerConfig)
```
- For non-TERM products (ULIP, PAR): generates benefit illustration timeline
- Creates visual representation of policy benefits over time
- Only for JK Bank broker and successful quotes

**d) Save Flattened Response** (Database Storage):
```java
saveFlattenedResponse(response, brokerConfig.getBroker())
```
- Stores quote response in database
- Creates denormalized structure for quick retrieval
- Used for rider premium lookup later

#### Step 2.9: Sort and Classify Responses
```java
.map(lifeResponses -> {
    if (!BrokerUtils.checkForTurtlemintBroker(brokerConfig)) {
        return liService.getLifeRequestValidator(category)
            .sortResponses(lifeResponses);
    }
    return lifeResponses;
})
.map(lifeResponses -> 
    liService.getLifeRequestValidator(category)
        .classifyIntoOptions(lifeResponses)
)
```
- **Sort Responses**: Orders quotes by premium amount or suitability
- **Classify Into Options**: Groups similar quotes and marks primary option
- **Response Structure**:
  - Main response (primary option)
  - Alternative responses (responseOptions list)

#### Step 2.10: Convert to PremiumResult
```java
.map(responses -> LifeResponseUtils.mapToPremiumResult(responses, premiumRequest))
```
- Transforms `LifePremiumResponse` list to `PremiumResult` list
- Final format expected by controller/client

---

## 3. **Results Provider Layer** (IResultsProvider)
**Location**: `/src/main/java/com/turtlemint/life/service/IResultsProvider.java`

### Provider Implementations

#### a) **NVEST Provider** (Investment-linked products)
- Calls NVEST web services for ULIP premium calculation
- Handles fund allocation details
- Receives response with fund-wise premium split

#### b) **MaxLife Provider**
- Calls MaxLife Insurance API
- Handles variable product quotes
- May return multiple option combinations

#### c) **Offline Provider**
- Uses pre-calculated premium factors from database
- Calculates premium offline using formulas
- Faster response (no external API call)
- Structure:
  ```
  productCode = LIFE_P88_RP  (for regular pay)
  productCode = LIFE_P88_LP  (for limited pay)
  Query premium factor from DB
  Premium = SumAssured × Factor × Adjustments
  ```

#### d) **CIS Provider** (Central Integration Service)
- Integrated insurer framework
- Makes API calls to multiple insurers
- Aggregates responses

### Provider Response Format
```java
Mono<LifePremiumResponse> getResultsFromProviderV2(
    PremiumRequest pr,
    LifePremiumRequest lifePremiumRequest,
    ValidProductRowsMapper validProductRowsMapper,
    InsurerConfig insurerConfig,
    String broker
) throws LifeInsuranceException
```

**Response Object** (`LifePremiumResponse`):
```java
{
    productCode: "P88",
    productName: "BAJAJ Suraksha Plus",
    insurerCode: "BAJAJLI",
    policyTerm: 20,
    premiumPaymentTerm: 5,
    paymentFrequency: "MONTHLY",
    premium: 2500.50,           // Annual premium
    installmentAmount: 208.37,   // Monthly installment
    resultId: "unique_id",
    status: "SUCCESS" / "ERROR",
    insurerStatus: "response from insurer",
    insurerMessage: "error message from insurer",
    planType: "ULIP",
    responseOptions: [ /* alternative quotes */ ],
    biTimeLine: [ /* visual timeline */ ],
    errorCategory: "PRODUCT_NOT_AVAILABLE"
}
```

---

## Database Collections Involved

### 1. **LifeRequestValidatorNM** (Validation Rules)
**Purpose**: Stores pre-processed validation rules for product matching

**Usage**: 
- Fetches rows matching customer profile
- Returns list of valid product-option combinations

### 2. **LifeProductMaster** (Product Master)
**Purpose**: Core product information

**Fields Used**:
- `productCode`: Product identifier
- `insurerCode`: Insurer identifier
- `provider`: Provider type (NVEST, INSURER, CIS, OFFLINE)
- `planType`: ULIP, PAR, NON_PAR

### 3. **LifeRiderMeta** (Rider Information)
**Purpose**: Rider details and premium factors

**Usage**:
- Linked to customer selections
- Rider premiums calculated separately

### 4. **LifeOfferMeta** (Promotional Offers)
**Purpose**: Discount and offer information

**Usage**:
- Applied to base premium
- May affect installment calculation

### 5. **Premium Factor Tables** (for Offline Provider)
**Purpose**: Pre-calculated factors for offline premium calculation

**Structure**:
```
{
    productCode: "LIFE_P88_RP",
    ageFrom: 25,
    ageTo: 30,
    pt: 20,
    ppt: 5,
    factor: 0.0125,  // Premium factor per 1000 SA
    carrier: "premium amount per 1000"
}
```

### 6. **LifePremiumResults** (Quote Storage)
**Purpose**: Stores calculated quotes for later retrieval

**Retention**: 
- Used for rider price lookup
- Referenced in checkout process

---

## Key Business Logic

### Premium Calculation Flow

```
1. BASE PREMIUM = SumAssured × AgeFactor × TermFactor
2. RIDER PREMIUM = Sum of all selected rider premiums
3. TOTAL ANNUAL PREMIUM = BASE + RIDERS + TAXES
4. INSTALLMENT = TOTAL / PaymentFrequencyDivisor
5. APPLY OFFERS = TOTAL - (DISCOUNT or DISCOUNT_AMOUNT)
6. FINAL PREMIUM = After offers & taxes
```

### Response Classification

Different responses generated for:
- **Different Policy Terms**: 10yr, 15yr, 20yr (if available)
- **Different Payment Frequencies**: Monthly, Quarterly, Annual
- **Different Investment Options** (ULIP): Growth, Balanced, Conservative
- **Rider Combinations**: With/without specific riders

### Error Handling

Errors handled at multiple levels:

1. **Insurer Level**: Insurer API returns error
   - Example: "Age below minimum" → Stored in `insurerMessage`
   
2. **Provider Level**: Provider couldn't call insurer API
   - Example: Connection timeout → Generic error response

3. **Service Level**: Invalid product/customer combination
   - Example: No valid product rows → Exception thrown

---

## Response Example

### Success Response
```json
{
  "premiumRequest": {
    "_id": "req_12345",
    "customerName": "John Doe",
    "lifePremiumRequest": {
      "entryAge": 30,
      "policyTerm": 20,
      "premiumPaymentTerm": 5,
      "paymentFrequency": "MONTHLY",
      "sumAssured": 1000000
    }
  },
  "premiumResults": [
    {
      "lifePremiumResponse": {
        "productCode": "P88",
        "productName": "BAJAJ Suraksha Plus",
        "insurerCode": "BAJAJLI",
        "policyTerm": 20,
        "premiumPaymentTerm": 5,
        "paymentFrequency": "MONTHLY",
        "premium": 30006.00,
        "installmentAmount": 500.10,
        "resultId": "res_98765",
        "status": "SUCCESS",
        "insurerStatus": "OK",
        "planType": "ULIP",
        "responseOptions": [
          {
            "policyTerm": 15,
            "premiumPaymentTerm": 3,
            "premium": 25000.00,
            "installmentAmount": 694.44,
            "resultId": "res_98766"
          }
        ]
      }
    }
  ]
}
```

### Error Response
```json
{
  "premiumResults": [
    {
      "lifePremiumResponse": {
        "productCode": "P88",
        "status": "ERROR",
        "insurerMessage": "Age below minimum coverage age",
        "insurerStatus": "VALIDATION_FAILED",
        "errorCategory": "AGE_BELOW_MIN"
      }
    }
  ]
}
```

---

## Workflow Summary

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

## Performance Considerations

1. **Reactive Streams**: Uses Flux for non-blocking I/O
2. **Async Rider Processing**: Rider details saved in parallel
3. **Error Resilience**: Graceful fallback for provider failures
4. **Caching**: Product master can be cached
5. **Lazy Evaluation**: DB queries execute only on subscription

---

## Related APIs

- **`POST /api/tm-life/v0/premiums/request/validate`**: Validates complete request before result calculation
- **`GET /api/tm-life/v0/rider/prices`**: Fetches saved rider premium details
- **`POST /api/tm-life/v0/resultselection`**: Selects specific result from options
- **`POST /api/tm-life/v0/products/query`**: Queries available products


