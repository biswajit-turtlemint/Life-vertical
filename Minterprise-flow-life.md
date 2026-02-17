# Life Quote Flow in Minterprise (Detailed API + Logic + DB Documentation)

This document describes the life quote journey implemented in minterprise under transactional-flows.

It covers:
- API contracts for FE
- request validation and "new implementation" flow
- validationMap creation and result execution
- pendingKeyList generation (from IH response)
- DB persistence for request/result
- mandatory collections/dependencies
- final response structures for all scenarios

## Source code references
- Controller: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/controllers/SachetController.java`
- Quote service: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java`
- Life aggregator: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java`
- Response model: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/beans/QuotationResponse.java`
- Request model: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/beans/QuotationRequest.java`
- Result model: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/beans/QuotationResult.java`
- Collection constants: `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/constant/DBColConstants.java`

## 1) FE API specification

Base path: `/api/minterprise/v2`

Common headers:
- `x-tenant` (optional, default applied by controller)
- `x-broker` (optional, default applied by controller)
- `x-partner-id` (optional)
- `x-provider` (optional)
- `Content-Type: application/json` for POST

---

### 1.1 Create quotes

- Method: `POST`
- URL: `/products/life/quotes`
- Controller method: `addQuotation(...)`

#### Request payload (combined structure: minterprise + life-service fields)

```json
{
  "data": {
    "referenceId": "",
    "quoteId": "",
    "policyType": "TRADITIONAL",
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-16T04:54:35.749Z",
      "utmParams": {
        "utmSource": "https://centinsure.cboi.bank.in/",
        "utmMedium": "referral",
        "utmUrl": "https://pro.centinsure.cboi.bank.in/life-insurance/results/AHUTQCG6NEY"
      },
      "initialReqFlag": true,
      "customerName": "UPENDRA SHRIVASTAVA",
      "userMobile": "9993679251",
      "userEmail": "upendasmarty@gmail.com",
      "currency": "INR",
      "riskInsured": {
        "dateOfBirth": "1987-10-11T00:00:00+05:30"
      },
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["investment"],
        "maritalStatus": "SINGLE",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "8 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 800000,
        "minIncome": 800000,
        "paymentFrequency": 12,
        "policyTerm": 10,
        "premiumPaymentTerm": 10,
        "premium": 70000,
        "profileType": "investment-planning",
        "insuredFullName": "UPENDRA SHRIVASTAVA",
        "dateOfBirth": "1987-10-11T00:00:00+05:30",
        "gender": "M",
        "entryAge": null,
        "isNonSelfJourney": false,
        "planFeatures": [],
        "riderMeta": [],
        "offerMeta": [],
        "ulipFundAllocationInfos": [],
        "includeCategory": false
      }
    }
  }
}
```

#### Mandatory/validated request points
- `data.premiumRequest.lifePremiumRequest` is mandatory.
- `data.premiumRequest.lifePremiumRequest.categories` must be non-empty.
- `paymentFrequency`, if present, must be one of `-1, 1, 3, 6, 12`.
- currency defaults to `INR` if missing.

#### Response envelope

```json
{
  "data": { "...QuotationResponse...": "..." },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

#### `data` schema (POST /quotes)
- `productCode`
- `referenceId`
- `quoteId`
- `status`
- `error`
- `pendingKeyList` (top-level)
- `quotes` (array of quote rows)
- `transactionSource`
- `transactionMode`

#### `data.quotes[*]` schema
Each row is `QuotationResult` style and includes:
- `_id`, `referenceId`, `provider`, `productCode`, `planCode`, `quoteId`
- `userId`, `partnerId`, `tenant`, `broker`
- `status` (`success|pending|failed`)
- `premium`, `tax`, `totalPremium`, `netPremium`, `currency`
- `planName`, `policyTerm`, `riskInsured`
- `fetchQuoteLinks`, `transactionSource`, `transactionMode`
- flattened life response fields at quote-row top level:
  - `lifePremiumResponse` (full life payload)
  - `validationMap`
  - `pendingKeyList`
  - `errorMessage`

#### `lifePremiumResponse` base fields added by minterprise (plus IH passthrough fields)
From `createLifePremiumResponse(...)`, these base fields are set and then merged with IH response:
- status/business metadata:
  - `resultId`, `status`, `insurerStatus`, `insurerBusinessFlowType`, `insurerCode`
- product/plan metadata:
  - `productCode`, `internalProductCode`, `productName`, `productUIN`, `tmPlanId`
  - `option`, `optionCode`, `planType`, `policyType`, `category`, `currency`
- term/age/frequency:
  - `policyTerm`, `premiumPaymentTerm`, `paymentFrequency`, `age`, `sumAssured`
- premium/tax:
  - `premium`, `tax`, `taxRate`, `premiumWithTax`, `premiumWithTaxWithRider`
- scoring and riders/offers:
  - `score`, `showRider`, `showRiderPremium`, `totalRiderPremium`, `riderList`, `offerList`
- enrichment:
  - `addons`, `payoutType`, `defermentPeriod`, `defaultOption`, `bqpRedirectionEnabled`
  - `paymentFirstJourneyCheck`, `stayInvestedValue`, `specialBenefits`
  - `planFeatureDetailsList`, `planFeatureList`, `maturityBenefits`, `planNote`, `tags`
  - `taxSavingAmount`, `inputCoverAmount`, `inputPremium`
  - `companyDetails`
  - `optionalBenefitAndExtraPremium` (when applicable)
  - `responseOptions` (other variants/options)

#### Variant and `responseOptions` behavior (parent vs options array)

This parent/child variant split is done by minterprise logic at our end, not directly returned in final shape by IH.

How it works:
1. We call IH for each valid row/variant (`createLifePremiumResponse(...)`), so each variant gets its own life response.
2. We collect all variant responses for a product key in `responses`.
3. We choose one response as parent (`mainResponse`) using `pickMainResponse(...)`:
   - first preference: `status=SUCCESS` and `defaultOption=true`
   - second preference: any `status=SUCCESS`
   - else: first available row
4. Parent response keeps its own `optionCode` at top level.
5. Remaining variant responses are placed in parent `responseOptions`.

Code snippet:

```java
responses.sort(Comparator.comparingDouble(response -> getDouble(response, "premiumWithTax", Double.MAX_VALUE)));

Map<String, Object> mainResponse = pickMainResponse(responses);
List<Map<String, Object>> responseOptions = new ArrayList<>(responses);
responseOptions.remove(mainResponse);
mainResponse.put("responseOptions", responseOptions);
```

So:
- `lifePremiumResponse.optionCode` at parent level = selected/main variant
- `lifePremiumResponse.responseOptions[*].optionCode` = other variants

IH still influences this because each variant payload is from IH and then merged, but final parent-vs-options grouping is performed by our aggregator logic.

`companyDetails` support is included and populated from `CompanyDetails` collection with fallback.

Important behavior for `companyDetails`:
- structure is dynamic (not fixed/minimal schema in code)
- full object fetched from DB is passed through (after minimal normalization for insurer code/name)
- so any additional keys present in `CompanyDetails` are preserved in response

Code path:

```java
Map<String, Object> companyDetails = resolveCompanyDetails(productMaster, request.getBroker());
response.put("companyDetails", companyDetails);
```

`resolveCompanyDetails(...)` lookup order:
1. by insurer code + broker
2. by insurer code + `Broker`
3. by insurer code without broker
4. fallback from product master (`insurerCode`, `insurerName`)

Normalization keeps DB fields and only ensures insurer key compatibility:

```java
Map<String, Object> normalized = new LinkedHashMap<>(companyDetails);
normalized.put("insurerCode", normalizedInsurerCode);
normalized.putIfAbsent("InsurerCode", normalizedInsurerCode);
normalized.putIfAbsent("Insurer_Code", normalizedInsurerCode);
fallback.forEach(normalized::putIfAbsent);
```

So if DB row has keys like `logoUrl`, `shortName`, `claimSettlementRatio`, `rating`, `website`, they will flow in response as part of `companyDetails`.

---

### 1.2 Get quotes (by referenceId)

- Method: `GET`
- URL: `/products/{productCode}/quotes`
- Query params:
  - `referenceId` required
  - `includeRequest` optional (`true|false`, default `false`)
- Headers:
  - `x-provider` optional filter

#### Response behavior
- Response shape is the same as `POST /quotes`.
- If `includeRequest=true`, one additional field is included: `data.premiumRequest`.
- If `includeRequest=false`, `data.premiumRequest` is not included.

#### includeRequest=true response example

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF123",
    "quoteId": "QID123",
    "status": "success",
    "error": null,
    "pendingKeyList": ["P31"],
    "quotes": [
      {
        "_id": "RID-P-1",
        "provider": "HDFCLI",
        "productCode": "life",
        "planCode": "P31",
        "quoteId": "QID123",
        "status": "pending",
        "lifePremiumResponse": {
          "status": "PENDING",
          "insurerStatus": "PENDING",
          "insurerCode": "HDFCLI",
          "productCode": "P31",
          "companyDetails": {
            "insurerCode": "HDFCLI",
            "insurerName": "HDFC Life"
          }
        },
        "pendingKeyList": ["P31"]
      }
    ],
    "premiumRequest": {
      "referenceId": "REF123",
      "quoteId": "QID123",
      "productCode": "life",
      "premiumRequest": {
        "lifePremiumRequest": {
          "categories": ["investment"]
        }
      }
    },
    "transactionSource": "API",
    "transactionMode": "ONLINE"
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

---

### 1.3 Poll pending quotes

- Method: `GET`
- URL: `/products/{productCode}/quotes/poll`
- Query params:
  - `referenceId` required
  - `resultIds` required (multi-value)

`resultIds` can be passed in either format (both are accepted by Spring `@RequestParam List<String>`):
- repeated param style:
  - `.../quotes/poll?referenceId=REF123&resultIds=RID1&resultIds=RID2`
- comma-separated style:
  - `.../quotes/poll?referenceId=REF123&resultIds=RID1,RID2`

Example:

```http
GET /api/minterprise/v2/products/life/quotes/poll?referenceId=REF123&resultIds=RID-P-1&resultIds=RID-S-1
```

#### Response behavior
- Response shape matches `POST /quotes` (`productCode`, `referenceId`, `quoteId`, `pendingKeyList`, `quotes`, etc.).
- `premiumRequest` is not returned by poll endpoint.
- `pendingKeyList` is recomputed from the selected result rows in DB.
- for `life`, poll first triggers pending-result refresh (IH/result flow) and then returns updated DB rows.

---

## 2) End-to-end flow: request -> validationMap -> result

### 2.1 Controller entry

```java
@PostMapping("/products/{productCode}/quotes")
public Mono<ResponseEntity<?>> addQuotation(...) {
    final QuotationRequest premiumRequest = JsonUtils.fromJsonObject(request.getData(), QuotationRequest.class);
    premiumRequest.setProductCode(productCode);
    premiumRequest.setTenant(tenant);
    premiumRequest.setBroker(broker);
    premiumRequest.setPartnerId(partnerId);
    premiumRequest.setProvider(provider);
    return quotationService.generateQuote(premiumRequest) ...
}
```

### 2.2 Quote service routes life to life aggregator

```java
if ("life".equalsIgnoreCase(request.getProductCode())) {
    return Flux.merge(validatePremium(request)
        .map(pr -> gridPremiumAggregatorFactory
            .getPremiumAggregator(request.getProductCode())
            .generateQuote(pr)));
}
```

### 2.3 Request persistence (before aggregation)
- request is stored in `sachetPremiumRequest`.
- history copy is stored in `sachetPremiumRequestHistory`.

Code path:
- `validatePremium(...)`
- `performValidation(...)`
- `saveAPIPremiumRequest(...)`
- `savePremiumRequestHistory(...)`

### 2.4 Life "new implementation" request flow (`processLifeFlow`)

Primary method: `processLifeFlow(...)` in `SachetLifeAggregator`.

Step sequence:
1. Parse and normalize `premiumRequest`, `lifePremiumRequest`, `riskInsured`.
2. Validate request basics:
   - mandatory `lifePremiumRequest`
   - mandatory categories
   - payment frequency validation
3. Derived calculations:
   - `setCurrencyDefault`
   - `calculateMaturityAge`
   - `calculatePtPpt`
   - `calculateInvestmentTermByPolicyTerm`
   - `resolveEntryAge`
4. Partner details:
   - from `partnerIntegrationList` using `partnerId`/`clientId`.
5. AB config:
   - from `LifeABTestingConfig` with feature `lifeValidatorConfig` and broker.
6. Product catalogue request build and fetch:
   - builds query params and request body.
   - calls `/api/product-management/v1/products/details/filters`.
   - transforms catalogue response to product master rows.
7. AB whitelisting/product filtering:
   - when AB `active=false` and conditions exist, whitelist is applied.
8. Payout defaults for turtlemint (if applicable).
9. Product post-filtering:
   - payout type / payout term / payout frequency filters.
10. Plan feature filtering.
11. Validation request build and nearest-match call:
   - builds life validation request.
   - calls `/api/product-management/v1/life-validator`.
   - nearest PT/PPT and variant option applied.
12. Riders and offers:
   - riders fetched from `lifeRiderMaster`.
   - rider slab/validate rule APIs are called when rule engine URL is configured.
   - offers fetched from `lifeOfferMaster`.
13. Combine validated product + rider + offer metadata.
14. Create `validationMap`.
15. Create premium results one-by-one by product key.
16. Compute `pendingKeyList`.
17. Build response wrapper.
18. Persist each result to `sachetPremiumResponse`.

#### 2.4.1 Code snippets for each step in `processLifeFlow(...)`

1. Parse and normalize request blocks

```java
Map<String, Object> premiumRequest = asMutableMap(request.getPremiumRequest());
Map<String, Object> lifePremiumRequest = asMutableMap(premiumRequest.get("lifePremiumRequest"));
Map<String, Object> riskInsured = asMutableMap(premiumRequest.get("riskInsured"));
premiumRequest.put("lifePremiumRequest", lifePremiumRequest);
premiumRequest.put("riskInsured", riskInsured);
```

2. Validate mandatory structures and payment frequency

```java
if (lifePremiumRequest.isEmpty()) {
    return buildErrorResponse("lifePremiumRequest is mandatory for life quote");
}
ErrorResponseData requestValidationError = validateLifeRequestPayload(lifePremiumRequest);
if (requestValidationError != null) {
    return requestValidationError;
}
LinkedHashSet<String> categories = toOrderedSet(lifePremiumRequest.get("categories"));
if (categories.isEmpty()) {
    return buildErrorResponse("categories is mandatory for life quote");
}
```

3. Derived calculations (`currency`, `entryAge`, `maturityAge`, `PT/PPT`, `investmentTerm`)

```java
setCurrencyDefault(premiumRequest, lifePremiumRequest);
String category = categories.iterator().next();
calculateDefaultParameters(lifePremiumRequest, riskInsured, category);
```

4. Partner details from DB

```java
Map<String, Object> partnerDetails = fetchPartnerDetails(request, premiumRequest);
// fetchPartnerDetails -> partnerIntegrationList lookup by partnerId/clientId
```

5. AB config fetch

```java
Map<String, Object> abConfig = fetchLifeAbTestingConfig(request.getBroker());
// feature = lifeValidatorConfig, broker-aware lookup in LifeABTestingConfig
```

6. Product catalogue query build + IH product filter API call

```java
ProductQueryParam queryParam = buildProductQueryParam(request, lifePremiumRequest, categories);
List<Map<String, Object>> enabledProductMasters = fetchEnabledProductMasters(queryParam, request.getBroker());
// fetchEnabledProductMasters -> /api/product-management/v1/products/details/filters
```

7. AB whitelist filtering

```java
enabledProductMasters = applyAbWhitelisting(enabledProductMasters, abConfig);
if (enabledProductMasters.isEmpty()) {
    return buildWrappedLifeResponse(request, premiumRequest, new LinkedHashMap<>(), new ArrayList<>(), List.of(), "No matching plans found");
}
```

8. Turtlemint payout defaults + post filter

```java
maybeSetPayoutDefaultsForTurtlemint(request.getBroker(), lifePremiumRequest, enabledProductMasters);
enabledProductMasters = postFilterProductMaster(lifePremiumRequest, enabledProductMasters);
```

9. Plan feature filtering

```java
List<Map<String, Object>> planFeatureDetailsList = extractPlanFeatureDetails(lifePremiumRequest);
enabledProductMasters = filterProductsForFeatures(enabledProductMasters, planFeatureDetailsList);
```

10. Validation request + nearest match

```java
Map<String, Object> validationRequest = createValidationRequest(
        category, lifePremiumRequest, productCodes, partnerDetails, request.getBroker()
);
List<Map<String, Object>> validatorRows = createLifeValidatorRows(enabledProductMasters, validationRequest);
validatorRows = applyNearestMatch(validationRequest, validatorRows);
```

11. Riders and offers enrichment

```java
Map<String, List<Map<String, Object>>> productValidRowsMap = createProductValidRowsMap(validatorRows);
Map<String, Map<String, Object>> enabledProductMap = createProductMasterRowMap(enabledProductMasters);
Map<String, Map<String, Object>> riderMetaMap = fetchEligibleRiderRows(productValidRowsMap, validationRequest, enabledProductMap, request.getBroker());
Map<String, Map<String, Object>> offerMetaMap = fetchEligibleOfferRows(productValidRowsMap, enabledProductMap, request.getBroker());
```

12. Combine product+rider+offer rows

```java
Map<String, List<Map<String, Object>>> validatedProductRows = combineValidProductRiderRows(
        productValidRowsMap, riderMetaMap, offerMetaMap, enabledProductMap, planFeatureDetailsList, lifePremiumRequest
);
```

13. Build validationMap

```java
Map<String, Map<String, Object>> validationMap = createValidationMap(validatedProductRows);
```

14. Build premium results one-by-one by product key and call IH/provider

```java
List<Map<String, Object>> premiumResults = createPremiumResults(validatedProductRows, request, premiumRequest, lifePremiumRequest, riskInsured);
// createPremiumResults -> per product key loop -> createLifePremiumResponse(...)
// createLifePremiumResponse -> ihService.sendPremiumRequestToIH(...)
```

15. Compute pending keys and wrap response

```java
List<String> pendingKeyList = computePendingKeyList(premiumResults);
QuotationResponse response = buildWrappedLifeResponse(request, premiumRequest, validationMap, premiumResults, pendingKeyList, errorMessage);
```

16. Persist results to DB

```java
persistLifePremiumResults(request, premiumRequest, premiumResults, pendingKeyList, errorMessage);
```

### 2.5 How `validationMap` is created

```java
Map<String, Map<String, Object>> validationMap = new LinkedHashMap<>();
for (Map.Entry<String, List<Map<String, Object>>> entry : validatedProductRows.entrySet()) {
    String productCode = entry.getKey();
    Map<String, Object> validatorNm = asMutableMap(entry.getValue().get(0).get("lifeRequestValidatorNM"));

    Map<String, Object> info = new LinkedHashMap<>();
    info.put("valid", true);
    info.put("key", productCode);
    info.put("insurerCode", getString(validatorNm, "insurerCode", null));
    validationMap.put(productCode, info);
}
```

### 2.6 How result flow is executed (one-by-one by productCode)

Yes, result is done one-by-one by product key from `validatedProductRows`:

```java
for (Map.Entry<String, List<Map<String, Object>>> entry : validatedProductRows.entrySet()) {
    String productCode = entry.getKey();
    List<Map<String, Object>> validRows = entry.getValue();

    List<Map<String, Object>> responses = new ArrayList<>();
    for (Map<String, Object> row : validRows) {
        responses.add(createLifePremiumResponse(...));
    }
    ...
}
```

Inside `createLifePremiumResponse(...)`:
- local response is created.
- external IH/provider call is made via:

```java
IHResponse ihResponse = ihService.sendPremiumRequestToIH(...).block();
```

- IH response life payload is extracted and merged:

```java
Map<String, Object> lifeResponse = extractLifeResponseMapFromIHData(ihResponse.getData());
Map<String, Object> merged = mergeExternalLifeResponse(localResponse, lifeResponse);
```

- if IH fails, provider-error response is created (`status=ERROR`).
- best/main response is selected; others go to `responseOptions`.

### 2.7 Final wrapper for POST /quotes

`buildWrappedLifeResponse(...)` sets:
- top-level: `productCode`, `referenceId`, `quoteId`, `status`, `error`, `pendingKeyList`, `quotes`, `transactionSource`, `transactionMode`
- each quote row has flattened fields: `lifePremiumResponse`, `validationMap`, `pendingKeyList`, `errorMessage`
- no `productDetails` wrapper is returned in quote rows for life APIs

### 2.8 All IH calls in life flow (payload and response usage)

#### IH call 1: enabled products (product catalogue)

Code path: `fetchEnabledProductMasters(...)`

Endpoint:
- `POST /api/product-management/v1/products/details/filters`

Payload built in code:

```java
Map<String, Object> requestBody = new LinkedHashMap<>();
requestBody.put("broker", broker);
requestBody.put("businessCategory", "LIFE");
requestBody.put("inclusions", Arrays.asList("PDP", "INTEGRATION_INFO"));
requestBody.put("productStatus", "LIVE");
requestBody.put("categoryTags", queryParam.getCategoryTags());
requestBody.put("policyTypes", queryParam.getPolicyTypes());
requestBody.put("planType", queryParam.getPlanType());
requestBody.put("insurerCodes", queryParam.getInsurerCodes());
requestBody.put("paymentFrequency", queryParam.getPaymentFrequency());
requestBody.put("currency", queryParam.getCurrency());
requestBody.put("selectedPlans", queryParam.getSelectedPlans());
```

How response is used:
1. product catalogue response list is extracted (`data/products` compatible parsing).
2. each row is transformed into internal `enabledProductMasters` shape (`productCode`, `internalProductCode`, `insurerCode`, `optionCode`, PDP fields, integration provider, riders/offers/features).
3. this list is then filtered by AB, payout, plan features.

#### IH call 2: nearest match validator

Code path: `applyNearestMatch(...)`

Endpoint:
- `POST /api/product-management/v1/life-validator`

Payload built in code:

```java
Map<String, Object> scope = new LinkedHashMap<>();
scope.put("age", getInteger(validationRequest, "entryAge", null));
scope.put("policyTerm", getInteger(validationRequest, "policyTerm", null));
scope.put("premiumPaymentTerm", getInteger(validationRequest, "premiumPaymentTerm", null));
scope.put("premium", getDouble(validationRequest, "premium", null));
scope.put("premiumPaymentMode", getInteger(validationRequest, "paymentFrequency", 12));
scope.put("sumAssuredOnDeath", getLong(validationRequest, "sumAssured", 0L));
scope.put("income", getLong(validationRequest, "minIncome", 0L));

Map<String, Object> nearestMatchRequest = new LinkedHashMap<>();
nearestMatchRequest.put("productCode", internalProductCodes);
nearestMatchRequest.put("scope", scope);
```

How response is used:
1. reads `lifeValidatorOptions` or `productVariants[*].lifeValidatorOptions`.
2. computes nearest PT/PPT/variant by score and nearest attribute bounds.
3. updates validator rows (`pt`, `ppt`, `option`, `score`).
4. updated rows then drive validationMap and result construction.

#### IH call 3: result quote call (`sendPremiumRequestToIH`)

Code path: `createLifePremiumResponse(...) -> fetchExternalLifePremiumResponse(...) -> executeExternalLifePremiumCall(...)`

Method used:

```java
IHResponse ihResponse = ihService.sendPremiumRequestToIH(
    integrationProvider,
    "LIFE",
    planCode,
    request.getBroker(),
    request.getTenant(),
    request.getReferenceId(),
    providerRequest
).block();
```

`providerRequest` payload includes:

```java
providerRequest.put("premiumRequest", premiumRequest);
providerRequest.put("lifePremiumRequest", lifePremiumRequest);
providerRequest.put("riskInsured", asMutableMap(premiumRequest.get("riskInsured")));
providerRequest.put("lifeRequestValidatorNM", validatorNM);
providerRequest.put("productMaster", productMaster);
providerRequest.put("validProductRowsMapper", validatedRow);
providerRequest.put("referenceId", request.getReferenceId());
providerRequest.put("quoteId", request.getQuoteId());
```

How IH result response is used:
1. extract life payload from IH data using fallback order:
   - `data.lifePremiumResponse`
   - `data.premiumResult.lifePremiumResponse`
   - `data.data.lifePremiumResponse`
   - direct map if contains quote fields (`premium/premiumWithTax/status`)
2. merge extracted fields over local response (`mergeExternalLifeResponse`) for non-null keys.
3. if IH call fails or empty payload, convert row to provider error:
   - `status=ERROR`
   - `insurerStatus=ERROR`
   - `insurerMessage=<error>`

Poll refresh also uses the same result IH call via `executeExternalLifePremiumCall(...)` for pending rows.

---

## 3) PendingKeyList deep explanation (IH to response)

### 3.1 Where pending status comes from
`PENDING` originates from insurer/IH payload merged into `lifePremiumResponse`.

Source order used in logic:
- `premiumResult.status`
- `lifePremiumResponse.status`
- `lifePremiumResponse.insurerStatus`

### 3.2 Exact pending list computation
In life aggregator (`computePendingKeyList`):

```java
String status = firstNonBlank(
    getString(premiumResult, "status", null),
    getString(lifePremiumResponse, "status", null),
    getString(lifePremiumResponse, "insurerStatus", null)
);
if (isPendingStatus(status)) {
    pendingKeys.add(key);
}
```

Also `responseOptions` are checked. If any option is `PENDING`, key is added.

### 3.3 Is pending only in top-level list or also in quote row?
Both:
- top-level: `data.pendingKeyList`
- per quote: `quotes[*].status` is `pending` and `quotes[*].lifePremiumResponse.status` is `PENDING`

### 3.4 Poll endpoint pending behavior
Poll endpoint reads selected rows from DB and recomputes pending keys with equivalent status checks (`pending`, `PENDING`, `insurerStatus=PENDING`, response options).

### 3.5 Detailed pendingKeyList algorithm

Pending evaluation is done at two places:
1. during POST `/quotes` life flow from in-memory `premiumResults`
2. during GET poll/get flow from stored `QuotationResult` rows

Decision order per quote/product:
1. read primary status (`premiumResult.status` or `lifePremiumResponse.status` or `lifePremiumResponse.insurerStatus`)
2. if primary status is pending -> key is pending
3. else inspect `lifePremiumResponse.responseOptions[*]`
4. if any option status is pending -> key is pending
5. key is de-duplicated in insertion order

Key resolution order:
1. `premiumResult.key`
2. `lifePremiumResponse.productCode`
3. `planCode`
4. `provider`
5. fallback to productCode

Code-level representation used in service layer:

```java
String status = firstNonBlank(
    toStringValue(premiumResult.get("status")),
    toStringValue(lifePremiumResponse.get("status")),
    toStringValue(lifePremiumResponse.get("insurerStatus"))
);
if (StringUtils.equalsIgnoreCase("PENDING", status)) {
    pendingKeys.add(resolvePendingKey(...));
}

for (Map<String, Object> responseOption : responseOptions) {
    String optionStatus = firstNonBlank(
        toStringValue(responseOption.get("status")),
        toStringValue(responseOption.get("insurerStatus"))
    );
    if (StringUtils.equalsIgnoreCase("PENDING", optionStatus)) {
        pendingKeys.add(resolvePendingKey(...));
    }
}
```

---

## 4) Full poll API flow

### 4.1 FE call sequence
1. Call `POST /products/life/quotes`.
2. Read `data.pendingKeyList`.
3. Collect pending `resultIds` from `data.quotes[*]._id` for rows that are pending.
4. Call `GET /products/life/quotes/poll?referenceId=...&resultIds=...`.
5. Repeat until `pendingKeyList` is empty and all rows are success/failed.

### 4.2 Poll endpoint service logic
`fetchResultsByReferenceIdAndResultIds(...)`:
- query by `referenceId + productCode + _id in resultIds`
- build same aggregate response shape using `buildAggregatedQuoteResponse(...)`
- compute `pendingKeyList` from stored rows
- for life, if any selected row is pending, it triggers result refresh using aggregator poll flow before final DB read

Refresh logic snippet:

```java
Set<String> pendingQuoteIds = resultList.stream()
    .filter(this::isPendingQuoteRow)
    .map(QuotationResult::getQuoteId)
    .collect(...);

return Flux.fromIterable(pendingQuoteIds)
    .concatMap(quoteId -> triggerLifeResultRefresh(referenceId, quoteId))
    .then(fetchResultsByReferenceIdAndResultIdsInternal(productCode, referenceId, resultIds));
```

`triggerLifeResultRefresh(...)` delegates to life aggregator poll, which internally calls IH for pending products and updates `sachetPremiumResponse` rows:

```java
QuotationRequest pollRequest = new QuotationRequest();
pollRequest.setProductCode("life");
pollRequest.setReferenceId(referenceId);
pollRequest.setQuoteId(quoteId);
return pollQuote(pollRequest).then();
```

### 4.3 Does poll endpoint call IH directly?
Not directly at controller response-building level. Response is DB-based.
- Pending becomes success/failed when DB records are updated by async processing/callback pipelines.
- Poll API reads latest stored status and returns current state.

Latest life behavior clarification:
- poll endpoint still returns from DB, but for `life` it now triggers poll refresh first.
- that refresh path calls IH/provider for pending rows (inside life aggregator), persists updates, then poll response is built from updated DB rows.

---

## 5) GET /quotes response behavior (current implementation)

### 5.1 includeRequest=false
- Calls `fetchAggregateResultsFromDB(...)`
- Returns same response structure as POST (`pendingKeyList`, `quotes`, etc.)
- Does not include `premiumRequest`
- Optional `x-provider` header filters by provider

### 5.2 includeRequest=true
- Calls `fetchRequestAndAggregateResultsFromDB(...)`
- Returns same response structure as POST
- Additionally includes `data.premiumRequest`

---

## 6) DB storage details and linkage

### 6.1 Collections used
From `DBColConstants.GRID_BASED`:
- `sachetPremiumRequest`
- `sachetPremiumRequestHistory`
- `sachetPremiumResponse`

### 6.2 What is saved where

Request save:
- collection: `sachetPremiumRequest`
- history: `sachetPremiumRequestHistory`
- includes normalized incoming quote request and metadata (`referenceId`, `quoteId`, tenant/broker/partner, payload)

Result save:
- collection: `sachetPremiumResponse`
- one row per provider-plan combination
- includes flattened quote row fields (same shape as life API quote row):
  - `lifePremiumResponse`
  - `validationMap`
  - `pendingKeyList`
  - `errorMessage`
  - along with standard quote identifiers and premium fields

### 6.3 Unique IDs and how request/result are connected
- request connection keys:
  - `referenceId`
  - `quoteId`
- result document key:
  - `_id` (resultId returned in API as `quotes[*]._id`)
- result upsert identity in life save:
  - `referenceId + quoteId + provider + planCode`

How to fetch linked data:
- request: query `sachetPremiumRequest` by `referenceId`
- results: query `sachetPremiumResponse` by `referenceId` and `quoteId`
- poll by selected rows: query `sachetPremiumResponse` by `_id in resultIds`

### 6.4 Need new DB fields/models for companyDetails or extra life fields?
For persistence, no new DB collection or DB schema migration is required.
For API response flattening, quote-row response fields are exposed at top level (`lifePremiumResponse`, `validationMap`, `pendingKeyList`, `errorMessage`).
Stored DB structure also uses the same flattened fields (no `productDetails` wrapper for life quote rows).
No new result collection is required.

### 6.5 Detailed persistence flow with code snippets

Request save (`sachetPremiumRequest`):

```java
private Mono<QuotationRequest> saveAPIPremiumRequest(QuotationRequest req) {
    Mono<QuotationRequest> pr = quotationRequestRepository.saveData(req);
    return pr.map(r -> {
        r.set_id(null);
        savePremiumRequestHistory(r).subscribe();
        return r;
    });
}
```

History save (`sachetPremiumRequestHistory`):

```java
private Mono<QuotationRequestHistory> savePremiumRequestHistory(QuotationRequest premiumRequest) {
    return quotationRequestHistoryRepository.saveData(
        JsonUtils.fromJsonObject(premiumRequest, QuotationRequestHistory.class)
    );
}
```

Result save (`sachetPremiumResponse`) from life aggregator:

```java
Map<String, Object> lookupConditions = new LinkedHashMap<>();
lookupConditions.put("referenceId", quotationResult.getReferenceId());
lookupConditions.put("quoteId", quotationResult.getQuoteId());
lookupConditions.put("provider", quotationResult.getProvider());
lookupConditions.put("planCode", quotationResult.getPlanCode());

QuotationResult existingResult = quotationResultRepository.findOneByProperties(lookupConditions, "createdAt").block();
if (existingResult != null) {
    quotationResult.set_id(existingResult.get_id());
}
quotationResultRepository.saveData(quotationResult).block();
```

This means result rows are updated (upsert-like behavior) on same `referenceId + quoteId + provider + planCode` identity.

### 6.6 Sample stored documents

`sachetPremiumRequest` sample:

```json
{
  "_id": "67b1....",
  "referenceId": "REF123",
  "quoteId": "QID123",
  "productCode": "life",
  "tenant": "dhofar",
  "broker": "dhofar",
  "partnerId": "67a2....",
  "premiumRequest": {
    "requestType": "INITIAL",
    "vertical": "LIFE",
    "lifePremiumRequest": { "...": "..." }
  },
  "riskInsured": { "...": "..." },
  "transactionSource": "API",
  "transactionMode": "ONLINE",
  "createdAt": "2026-02-16T...",
  "updatedAt": "2026-02-16T..."
}
```

`sachetPremiumRequestHistory` sample:

```json
{
  "_id": "67b1....",
  "referenceId": "REF123",
  "quoteId": "QID123",
  "productCode": "life",
  "premiumRequest": { "...": "..." },
  "riskInsured": { "...": "..." },
  "createdAt": "2026-02-16T..."
}
```

`sachetPremiumResponse` sample:

```json
{
  "_id": "RID-P-1",
  "referenceId": "REF123",
  "quoteId": "QID123",
  "productCode": "life",
  "provider": "HDFCLI",
  "planCode": "P31",
  "status": "pending",
  "premium": 59322.02,
  "tax": 10677.97,
  "totalPremium": 69999.99,
  "transactionSource": "API",
  "transactionMode": "ONLINE",
  "lifePremiumResponse": {
    "status": "PENDING",
    "insurerStatus": "PENDING",
    "insurerCode": "HDFCLI",
    "productCode": "P31"
  },
  "validationMap": {
    "P31": {
      "valid": true,
      "key": "P31",
      "insurerCode": "HDFCLI"
    }
  },
  "pendingKeyList": ["P31"],
  "errorMessage": null,
  "createdAt": "2026-02-16T...",
  "updatedAt": "2026-02-16T..."
}
```

---

## 7) Mandatory dependencies/collections for flow

### 7.1 External APIs
- product catalogue: `/api/product-management/v1/products/details/filters`
- life validator: `/api/product-management/v1/life-validator`
- IH provider quote execution (via `ihService.sendPremiumRequestToIH`)
- rider rule APIs (if rule engine URL is configured):
  - `/api/rules/v0/life/riders/slab`
  - `/api/rules/v0/life/riders/validate`

### 7.2 Collections referenced by life flow
- `LifeABTestingConfig`
- `partnerIntegrationList`
- `lifeRiderMaster`
- `lifeOfferMaster`
- `CompanyDetails`

If some metadata is unavailable:
- base quote flow still executes with fallbacks,
- but enrichment/rider/offer/company metadata may be reduced.

---

## 8) Final response examples for key scenarios

### 8.1 Pending

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF001",
    "quoteId": "QID001",
    "status": "success",
    "error": null,
    "pendingKeyList": ["P31"],
    "quotes": [
      {
        "_id": "RID-P-1",
        "provider": "HDFCLI",
        "productCode": "life",
        "planCode": "P31",
        "quoteId": "QID001",
        "status": "pending",
        "lifePremiumResponse": {
          "status": "PENDING",
          "insurerStatus": "PENDING",
          "insurerCode": "HDFCLI",
          "productCode": "P31",
          "companyDetails": {
            "insurerCode": "HDFCLI",
            "insurerName": "HDFC Life"
          },
          "responseOptions": []
        },
        "pendingKeyList": ["P31"]
      }
    ],
    "transactionSource": "API",
    "transactionMode": "ONLINE"
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

### 8.2 Success

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF002",
    "quoteId": "QID002",
    "status": "success",
    "error": null,
    "pendingKeyList": [],
    "quotes": [
      {
        "_id": "RID-S-1",
        "provider": "HDFCLI",
        "productCode": "life",
        "planCode": "P31",
        "quoteId": "QID002",
        "status": "success",
        "premium": 59322.02,
        "tax": 10677.97,
        "totalPremium": 69999.99,
        "planName": "Life Investment Plan",
        "lifePremiumResponse": {
          "status": "SUCCESS",
          "insurerStatus": "SUCCESS",
          "insurerCode": "HDFCLI",
          "productCode": "P31",
          "internalProductCode": "P31_INTERNAL",
          "productName": "Life Investment Plan",
          "option": "Default",
          "optionCode": -1,
          "planType": "TRADITIONAL",
          "policyType": "TRADITIONAL",
          "currency": "INR",
          "policyTerm": 10,
          "premiumPaymentTerm": 10,
          "paymentFrequency": 12,
          "age": 38,
          "sumAssured": 1000000.0,
          "premium": 59322.02,
          "tax": 10677.97,
          "taxRate": 18.0,
          "premiumWithTax": 69999.99,
          "premiumWithTaxWithRider": 69999.99,
          "score": 0,
          "showRider": true,
          "showRiderPremium": true,
          "totalRiderPremium": 0.0,
          "riderList": [],
          "offerList": [],
          "addons": [],
          "defaultOption": true,
          "bqpRedirectionEnabled": false,
          "paymentFirstJourneyCheck": false,
          "stayInvestedValue": "TRADITIONAL",
          "specialBenefits": [],
          "planFeatureDetailsList": [],
          "planFeatureList": [],
          "maturityBenefits": [],
          "planNote": [],
          "tags": [],
          "taxSavingAmount": 5932.2,
          "inputCoverAmount": 1000000,
          "inputPremium": 70000,
          "companyDetails": {
            "insurerCode": "HDFCLI",
            "insurerName": "HDFC Life"
          },
          "responseOptions": []
        },
        "validationMap": {
          "P31": {
            "valid": true,
            "key": "P31",
            "insurerCode": "HDFCLI"
          }
        },
        "pendingKeyList": [],
        "errorMessage": null
      }
    ],
    "transactionSource": "API",
    "transactionMode": "ONLINE"
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

### 8.3 Success with rider and variant

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF003",
    "quoteId": "QID003",
    "status": "success",
    "error": null,
    "pendingKeyList": [],
    "quotes": [
      {
        "_id": "RID-R-1",
        "provider": "ICICILI",
        "productCode": "life",
        "planCode": "P51",
        "quoteId": "QID003",
        "status": "success",
        "premium": 62000.0,
        "tax": 11160.0,
        "totalPremium": 73160.0,
        "lifePremiumResponse": {
          "status": "SUCCESS",
          "insurerStatus": "SUCCESS",
          "insurerCode": "ICICILI",
          "productCode": "P51",
          "option": "Gold Variant",
          "optionCode": 2,
          "showRider": true,
          "showRiderPremium": true,
          "totalRiderPremium": 1800.0,
          "riderList": [
            {
              "riderCode": "ADB",
              "isSelected": true,
              "riderPremium": 1800.0
            }
          ],
          "responseOptions": [
            {
              "status": "SUCCESS",
              "optionCode": 1,
              "premiumWithTax": 71390.0
            }
          ],
          "companyDetails": {
            "insurerCode": "ICICILI",
            "insurerName": "ICICI Prudential"
          }
        },
        "pendingKeyList": []
      }
    ]
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

### 8.4 Failure

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF004",
    "quoteId": "QID004",
    "status": "success",
    "error": "Some provider calls failed",
    "pendingKeyList": [],
    "quotes": [
      {
        "_id": "RID-F-1",
        "provider": "BAJAJLI",
        "productCode": "life",
        "planCode": "P99",
        "quoteId": "QID004",
        "status": "failed",
        "lifePremiumResponse": {
          "status": "ERROR",
          "insurerStatus": "ERROR",
          "insurerCode": "BAJAJLI",
          "insurerMessage": "INTERNAL_LIFE_SERVICE_ERROR"
        },
        "pendingKeyList": []
      }
    ]
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

### 8.5 Multiple products

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF005",
    "quoteId": "QID005",
    "status": "success",
    "error": null,
    "pendingKeyList": ["P51"],
    "quotes": [
      {
        "_id": "RID-M-1",
        "provider": "HDFCLI",
        "productCode": "life",
        "planCode": "P31",
        "quoteId": "QID005",
        "status": "success",
        "lifePremiumResponse": {
          "status": "SUCCESS",
          "insurerCode": "HDFCLI",
          "productCode": "P31"
        }
      },
      {
        "_id": "RID-M-2",
        "provider": "ICICILI",
        "productCode": "life",
        "planCode": "P51",
        "quoteId": "QID005",
        "status": "pending",
        "lifePremiumResponse": {
          "status": "PENDING",
          "insurerCode": "ICICILI",
          "productCode": "P51"
        }
      },
      {
        "_id": "RID-M-3",
        "provider": "BAJAJLI",
        "productCode": "life",
        "planCode": "P77",
        "quoteId": "QID005",
        "status": "failed",
        "lifePremiumResponse": {
          "status": "ERROR",
          "insurerCode": "BAJAJLI",
          "productCode": "P77"
        }
      }
    ],
    "transactionSource": "API",
    "transactionMode": "ONLINE"
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```
