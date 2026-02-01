# API Endpoint Documentation: `/api/tm-life/v0/products/query`

## Overview
The `/query` API is a **product query and filtering endpoint** that retrieves eligible life insurance products based on customer requirements and validates them against various criteria. It returns a curated list of products with supported features like payout types, deferment periods, income terms, and special plan features (Joint Life, Return of Premium).

---

## API Endpoint Details

### Route
```
POST /api/tm-life/v0/products/query
```

### Controller Class
- **File**: `LifeProductsQueryApi.java`
- **Package**: `com.turtlemint.life.controller`

### Request Format
```java
@PostMapping(value = "/query")
@ResponseBody
public Flux<LifeProductQueryResponseDto> getProducts(
    @RequestBody PremiumRequest request, 
    ServerHttpRequest serverRequest
)
```

### Request Parameters
- **PremiumRequest**: Contains customer details and product filtering criteria
  - `LifePremiumRequest`: Nested object with insurance-specific parameters
    - `entryAge`: Customer's age
    - `policyTerm`: Duration of the policy
    - `premiumPaymentTerm`: Duration for premium payment
    - `paymentFrequency`: Frequency of premium payment
    - `sumAssured` / `coverAmount`: Insured amount
    - `categories`: Product categories to search (e.g., TERM, ULIP, PAR, WOL)
    - `maturityAge`: Age at policy maturity
    - `minIncome` / `maxIncome`: Income range filters
    - `businessModel`: Customer business relationship model
    - `investmentGoals`: Customer's investment objectives
    - `riskAppetite`: Risk preference level

- **ServerHttpRequest**: HTTP request object to determine broker configuration

### Response Format
```java
Flux<LifeProductQueryResponseDto>
```

Returns a **Reactive Stream (Flux)** of products. Each product contains:
- `productCode`: Unique product identifier
- `productName`: Product name
- `insurerCode`: Insurance company code
- `insurerName`: Insurance company name
- `jointLife`: Boolean flag indicating Joint Life feature availability
- `returnOfPremium`: Boolean flag indicating ROP feature availability
- `incomeTerms`: Set of supported income/payout terms (years)
- `payoutTypes`: Set of supported payout types (e.g., ANNUITY, LUMPSUM)
- `defermentPeriods`: Set of supported deferment periods (years)
- `planType`: Type of plan (ULIP, PAR, NON_PAR)

---

## Function Flow and Architecture

### 1. **Controller Layer** (`LifeProductsQueryApi.getProducts`)
**Location**: `/src/main/java/com/turtlemint/life/controller/LifeProductsQueryApi.java`

**Responsibilities**:
- Receive HTTP request
- Determine broker configuration from request hostname using `BrokerUtils.determineBroker()`
- Set currency in request using `LifeUtils.setCurrency()`
- Delegate to service layer
- Handle errors gracefully with `onErrorResume()` and `onErrorReturn()`

**Key Code**:
```java
@PostMapping(value = "/query")
@ResponseBody
public Flux<LifeProductQueryResponseDto> getProducts(@RequestBody PremiumRequest request, ServerHttpRequest serverRequest) {
    BrokerConfig brokerConfig = BrokerUtils.determineBroker(serverRequest.getURI().getHost());
    LifeUtils.setCurrency(request);
    return lifeProductsQueryService.getLifeProducts(request, brokerConfig)
        .onErrorResume(throwable -> {
            TMLogger.info("Errored out request" + request);
            TMLogger.error("[LifeProductsQueryApi] Error occurred in getProducts method : " + throwable.getMessage(), throwable);
            return Flux.empty();
        })
        .onErrorReturn(new LifeProductQueryResponseDto());
}
```

---

### 2. **Service Layer** (`LifeProductsQueryServiceImpl`)
**Location**: `/src/main/java/com/turtlemint/life/service/impl/LifeProductsQueryServiceImpl.java`

**Main Method**: `getLifeProducts(PremiumRequest premiumRequest, BrokerConfig brokerConfig)`

#### Flow Steps:

**Step 2.1: Pre-process Request**
```java
preProcessPremiumRequest(premiumRequest);
```
- Clears/resets filter parameters that should not be applied during query phase:
  - `jointLifeDetails` → null
  - `returnOfPremiumDetails` → null
  - `insurers` → null
  - `payoutType` → null
  - `defermentPeriod` → null
- **Purpose**: Ensures query returns all eligible products without premature filtering

**Step 2.2: Extract Categories**
```java
LifePremiumRequest lifePremiumRequest = premiumRequest.getLifePremiumRequest();
Set<String> categories = lifePremiumRequest.getCategories();
```
- Extracts product categories from request (e.g., TERM, ULIP, PAR, WOL)
- Gets first category to determine which validator to use

**Step 2.3: Get Appropriate Validator**
```java
ILifeRequestValidator lifeRequestValidator = liService.getLifeRequestValidator(categories.iterator().next());
```
- Retrieves category-specific validator using service locator pattern
- Different validators for: Term, ULIP, PAR, Whole of Life, Guaranteed, Child, etc.

**Step 2.4: Validate and Fetch Products**
```java
return lifeRequestValidator.validatePremiumRequest(premiumRequest, brokerConfig, true)
    .map(validatedProductRows -> createLifeFiltersResponse(validatedProductRows))
    .onErrorReturn(createLifeFiltersResponse(null))
    .flatMapIterable(list -> list); // Mono<List<T>> ---> Flux<T>
```
- Calls validator's `validatePremiumRequest()` method (returns `Mono<Map<String, List<ValidProductRowsMapper>>>`)
- Maps result to response DTOs
- Handles errors by returning empty response
- Converts `Mono<List>` to `Flux` for streaming results

**Step 2.5: Response Mapping**
```java
createLifeFiltersResponse(Map<String, List<ValidProductRowsMapper>> validatedProductRowsMap)
```
- Iterates through validated product rows map
- For each product entry, extracts features and creates response DTOs
- Returns `List<LifeProductQueryResponseDto>`

**Step 2.6: Extract Product Features**

**a) Product Basic Info** (`validProductsRowsQueryResponseMapper`):
```java
lifeProductQueryResponseDto.setProductCode(product.getProductCode());
lifeProductQueryResponseDto.setProductName(product.getProductName());
lifeProductQueryResponseDto.setInsurerCode(product.getInsurerCode());
lifeProductQueryResponseDto.setInsurerName(product.getInsurerName());
```

**b) Plan Features** (`setProductRowParams`):
```java
List<PlanFeatureDetails> planFeatureDetailsList = validProductRowsMapper.getLifePlanFeatureDetailsInfo().getPlanFeatureDetailsList();
for (PlanFeatureDetails planFeatureDetails : planFeatureDetailsList) {
    if (planFeatureDetails.getCode().equals(PlanFeatureEnum.JOINT_LIFE.getCode())) {
        lifeProductQueryResponseDto.setJointLife(Boolean.TRUE);
    } else if (planFeatureDetails.getCode().equals(PlanFeatureEnum.RETURN_OF_PREMIUM.getCode())) {
        lifeProductQueryResponseDto.setReturnOfPremium(Boolean.TRUE);
    }
}
```

**c) Payout Parameters** (`setPayoutParams`):
```java
Set<Integer> supportedIncomeTerms = lifeProductQueryResponseDto.getIncomeTerms();
if(productMaster.getPayoutBenefitTerms() != null) {
    supportedIncomeTerms.addAll(productMaster.getPayoutBenefitTerms());
}

Set<LifePayoutType> supportedPayoutTypes = lifeProductQueryResponseDto.getPayoutTypes();
if(productMaster.getPayoutTypes() != null) {
    productMaster.getPayoutTypes().forEach(payoutType -> {
        supportedPayoutTypes.add(LifePayoutType.valueOf(payoutType));
    });
}
```

**d) Deferment Parameters** (`setDefermentParams`):
```java
Set<Integer> supportedDefermentPeriods = lifeProductQueryResponseDto.getDefermentPeriods();
if(Objects.nonNull(productMaster.getDefermentPeriods())) {
    supportedDefermentPeriods.addAll(productMaster.getDefermentPeriods());
}
```

---

### 3. **Validator Layer** (Category-Specific Implementations)
**Location**: `/src/main/java/com/turtlemint/life/validator/service/impl/`

**Interface**: `ILifeRequestValidator.java`

**Main Method**: `validatePremiumRequest(PremiumRequest, BrokerConfig, Boolean doNotApplyDefaulting)`

#### Implementations:
- `TermRequestValidator` → Handles TERM category products
- `ULIPRequestValidator` → Handles ULIP products
- `ParticipatingRequestValidator` → Handles PAR products
- `GuaranteedRequestValidator` → Handles GUARANTEED products
- `WholeOfLifeRequestValidator` → Handles WOL products
- `ChildRequestValidator` → Handles CHILD products

#### Common Validation Steps (from AbstractLifeRequestValidator):

**3.1: Request Transformation**
- Converts `PremiumRequest` to `LifeValidationRequestMapper`
- Calculates derived fields:
  - **Entry Age**: If DOB provided, calculate age from current date
  - **Maturity Age**: Derived from policy term + entry age
  - **Policy Term**: If not provided, calculated from maturity age - entry age

**3.2: Fetch Enabled Products**
```java
List<LifeProductMaster> enabledProductMasterRows = 
    liService.getProductMasterDao()
        .findAllNRx(LifeProductMaster.class, brokerConfig.getBroker());
```
- Retrieves all enabled products for the broker from DB

**3.3: Fetch Plan Features**
```java
List<PlanFeatureDetails> planFeatureDetailsList = 
    liService.getPlanFeatureValidator()
        .fetchPlanFeatureDetails(...)
```
- Gets supported plan features (Joint Life, ROP, etc.)

**3.4: Match Validation Rules**
- Fetches `LifeRequestValidatorNM` rows matching criteria:
  - Age range (min age ≤ entry age ≤ max age)
  - Policy term constraints
  - Premium payment term constraints
  - Sum assured range matching

**3.5: Nearest Match Algorithm**
- If exact matches not found, uses **nearest match validation**
- Finds closest matching products by:
  - Age proximity
  - Policy term proximity
  - Premium payment term proximity
  - Sum assured range overlap

**3.6: Fetch Riders**
```java
Map<String, LifeRiderMeta> lifeRiderMetaMap = 
    liService.getLifeRiderValidator()
        .fetchEligibleRiderRows(productValidRowsMap, validationRequest, ...)
```
- Retrieves eligible riders for each matched product

**3.7: Fetch Offers**
```java
Map<String, LifeOfferMeta> lifeOfferMetaMap = 
    liService.getLifeOfferValidator()
        .fetchEligibleOfferRows(productValidRowsMap, validationRequest, ...)
```
- Retrieves applicable offers/promotions

**3.8: Combine Results**
```java
Map<String, List<ValidProductRowsMapper>> validProductRowsMapperMap = 
    combineValidProductRiderRows(productValidRowsMap, lifeRiderMetaMap, lifeOfferMetaMap, ...)
```
- Combines products, riders, and offers into single mapper objects

---

## Database Collections Involved

### 1. **LifeProductMaster** (Primary Collection)
**Collection Name**: `lifeProductMaster`

**Purpose**: Stores master data for all insurance products

**Key Fields**:
- `_id`: MongoDB ObjectId
- `productCode`: Unique product identifier
- `productName`: Display name
- `insurerCode`: Insurance company code
- `insurerName`: Insurance company name
- `optionCode`: Product option variant
- `optionName`: Option display name
- `planType`: ULIP / PAR / NON_PAR
- `productUIN`: Unique Identification Number
- `provider`: NVEST / INSURER / NVEST-INTEGRATION
- `isActive`: Boolean activation flag
- `minAge`: Minimum customer age allowed
- `maxAge`: Maximum customer age allowed
- `minPolicyTerm`: Minimum policy duration (years)
- `maxPolicyTerm`: Maximum policy duration (years)
- `minSumAssured`: Minimum cover amount
- `maxSumAssured`: Maximum cover amount
- `payoutBenefitTerms`: Supported income/payout terms (array of integers)
- `payoutTypes`: Supported payout types (e.g., "ANNUITY", "LUMPSUM")
- `defermentPeriods`: Supported deferment periods (array of integers)

**Usage in Flow**: 
- Primary source for product details
- Provides supported features, limits, and payout options
- Filtered based on customer age, sum assured, and policy terms

---

### 2. **LifeRequestValidatorNM** (Validation Rules Collection)
**Collection Name**: `lifeRequestValidatorNM`

**Purpose**: Stores validation rules for matching customers to products

**Key Fields**:
- `_id`: MongoDB ObjectId
- `productCode`: Product this rule applies to
- `option`: Product option number
- `minAge`: Minimum age for this rule
- `maxAge`: Maximum age for this rule
- `minPolicyTerm`: Minimum policy term
- `maxPolicyTerm`: Maximum policy term
- `minPremiumPaymentTerm`: Minimum PPT
- `maxPremiumPaymentTerm`: Maximum PPT
- `minSumAssured`: Minimum cover amount
- `maxSumAssured`: Maximum cover amount
- `broker`: Broker code this rule applies to

**Usage in Flow**: 
- Matched against customer parameters (age, terms, sum assured)
- Returns eligible products for given customer profile
- Supports nearest-match algorithm if no exact matches found

---

### 3. **LifeRiderMeta** (Riders Configuration)
**Collection Name**: `lifeRiderMeta`

**Purpose**: Stores available riders and their applicability rules

**Key Fields**:
- `_id`: MongoDB ObjectId
- `productCode`: Product this rider applies to
- `riderCode`: Unique rider identifier
- `riderName`: Display name
- `isOptional`: Whether rider is mandatory or optional
- `minRiderAge`: Minimum age to add rider
- `maxRiderAge`: Maximum age to add rider
- `applicableProducts`: Products where this rider can be added

**Usage in Flow**: 
- Fetched after product validation
- Determines which riders can be added to matched products
- Checked for age and product applicability

---

### 4. **LifeOfferMeta** (Offers/Promotions)
**Collection Name**: `lifeOfferMeta`

**Purpose**: Stores promotional offers and discounts

**Key Fields**:
- `_id`: MongoDB ObjectId
- `offerId`: Unique offer identifier
- `offerName`: Display name
- `applicableProducts`: Products where offer applies
- `discountPercentage` / `discountAmount`: Offer value
- `validFrom` / `validTo`: Offer validity period
- `conditions`: Applicability conditions

**Usage in Flow**: 
- Fetched for matched products
- Provides promotional information in response
- Can be used for premium calculation

---

### 5. **PlanFeatureDetails** (Plan Features)
**Collection Name**: `planFeatureDetails`

**Purpose**: Stores available plan features like Joint Life, Return of Premium

**Key Fields**:
- `_id`: MongoDB ObjectId
- `code`: Feature code (e.g., "JOINT_LIFE", "ROP")
- `name`: Feature name
- `description`: Feature description

**Usage in Flow**: 
- Retrieved to check feature availability
- Used to populate `jointLife` and `returnOfPremium` flags in response

---

### 6. **LifePayoutInfo** (Payout Configuration)
**Collection Name**: `lifePayoutInfo`

**Purpose**: Stores payout structure for products

**Key Fields**:
- `_id`: MongoDB ObjectId
- `productCode`: Product code
- `payoutType`: Type of payout (ANNUITY, LUMPSUM, etc.)
- `supportedTerms`: Available payout terms
- `supportedFrequencies`: Payout frequencies

**Usage in Flow**: 
- Fetched to populate supported payout types and income terms
- Used to validate payout selections in subsequent APIs

---

### 7. **LifeDefermentInfo** (Deferment Configuration)
**Collection Name**: `lifeDefermentInfo`

**Purpose**: Stores deferment period configurations

**Key Fields**:
- `_id`: MongoDB ObjectId
- `productCode`: Product code
- `supportedPeriods`: Array of deferment periods (years)
- `conditions`: Applicability conditions

**Usage in Flow**: 
- Fetched to populate supported deferment periods
- Used to validate deferment selections in subsequent APIs

---

## Summary of Data Flow

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

## Key Validations and Conditions

### Age Validation
- Entry age must be between `minAge` and `maxAge` of validation rule
- If DOB provided, entry age is calculated from current date
- Maturity age calculated as entry age + policy term

### Policy Term Validation
- Policy term must be within `minPolicyTerm` to `maxPolicyTerm` range
- If maturity age provided, policy term is calculated automatically

### Premium Payment Term Validation
- Payment term must be within `minPremiumPaymentTerm` to `maxPremiumPaymentTerm`
- Payment term cannot exceed policy term

### Sum Assured Validation
- Cover amount must be within `minSumAssured` to `maxSumAssured` range
- Validated against product master constraints

### Nearest Match Algorithm
- If no exact rule matches customer profile, finds "closest" matches
- Matches by:
  1. Age proximity (finds rule with age range closest to entry age)
  2. Policy term proximity
  3. Premium payment term proximity
  4. Sum assured range overlap

### Product Eligibility
- Product must be `isActive = true` in LifeProductMaster
- Product must be enabled for the given broker
- All age/term/sum assured constraints must be satisfied

---

## Response Example

```json
[
  {
    "productCode": "P88",
    "productName": "BAJAJ Suraksha Plus",
    "insurerCode": "BAJAJLI",
    "insurerName": "Bajaj Allianz Life Insurance",
    "jointLife": true,
    "returnOfPremium": false,
    "incomeTerms": [5, 10, 15],
    "payoutTypes": ["ANNUITY", "LUMPSUM"],
    "defermentPeriods": [0, 2, 5],
    "planType": "ULIP"
  },
  {
    "productCode": "P92",
    "productName": "LIC Jeevan Anand",
    "insurerCode": "LIC",
    "insurerName": "Life Insurance Corporation",
    "jointLife": false,
    "returnOfPremium": true,
    "incomeTerms": [5, 7, 10],
    "payoutTypes": ["LUMPSUM"],
    "defermentPeriods": [0, 3],
    "planType": "PAR"
  }
]
```

---

## Error Handling

- **Invalid Request**: Returns empty Flux (via `onErrorResume`)
- **No Matching Products**: Returns Flux with empty list
- **Validation Exception**: Logs error and returns empty response
- **Database Connection Error**: Exception propagated with error logging

---

## Performance Considerations

1. **Reactive Streams**: Uses Project Reactor (Flux/Mono) for non-blocking I/O
2. **Lazy Evaluation**: Database queries executed only when subscribed
3. **Error Recovery**: Graceful fallback to empty results on errors
4. **Caching**: LifeProductMaster can be cached at application level for frequent access

---

## Related APIs

- **`POST /api/tm-life/v0/premiums/request/validate`**: Validates complete premium request with rider selections
- **`POST /api/tm-life/v0/results`**: Calculates actual premium quotes for selected product
- **`POST /api/tm-life/v0/riders`**: Fetches available riders for selected product


