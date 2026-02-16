# Life Quote Flow Documentation (Minterprise)

This document explains the life quote flow implemented in minterprise for:
- `POST /api/minterprise/v2/products/life/quotes`
- `POST /api/minterprise/v2/products/life/quotes/poller`

It covers:
- FE API contract (URL, headers, payload, response)
- request flow (new implementation)
- `validationMap` creation and result trigger in same API
- result flow (one-by-one per product key)
- DB writes and request-result linkage
- mandatory/optional data dependencies
- final response structure

## 1) API Contract for FE

Base controller: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/controllers/SachetController.java:43`

### 1.1 Create Quote API

- Method: `POST`
- URL (life): `/api/minterprise/v2/products/life/quotes`
- Generic route: `/api/minterprise/v2/products/{productCode}/quotes`
- Controller method: `addQuotation(...)` at `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/controllers/SachetController.java:94`

Headers:
- `Content-Type: application/json`
- `x-tenant` optional (default tenant applied)
- `x-broker` optional (default broker applied)
- `x-partner-id` optional
- `x-provider` optional

Request body (`PayloadWrapper`):

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

Mandatory input:
- `data.premiumRequest.lifePremiumRequest`
- `categories` inside `data.premiumRequest.lifePremiumRequest` (must have at least one category)
- if `paymentFrequency` is sent, allowed values are `-1,1,3,6,12`

ID behavior:
- `referenceId` is generated if both `referenceId` and `quoteId` are empty.
- `quoteId` is generated on each create-quote call.
- logic in `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java:211`

### 1.2 Poller API

- Method: `POST`
- URL (life): `/api/minterprise/v2/products/life/quotes/poller`
- Generic route: `/api/minterprise/v2/products/{productCode}/quotes/poller`
- Controller method: `pollQuotation(...)` at `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/controllers/SachetController.java:115`

Headers:
- same as create quote

Request body (`PayloadWrapper`):

```json
{
  "data": {
    "referenceId": "REF123",
    "quoteId": "QID123"
  }
}
```

Mandatory input:
- `data.referenceId`
- `data.quoteId`

Note:
- poller is supported only for `productCode=life`.
- check in `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java:145`

### 1.3 FE Calling Pattern

1. FE calls `POST /api/minterprise/v2/products/life/quotes`.
2. FE reads `data.productDetails.pendingKeyList`.
3. If not empty, FE calls `POST /api/minterprise/v2/products/life/quotes/poller` with same `referenceId` and `quoteId`.
4. Repeat step 3 until `pendingKeyList` is empty.

---

## 2) Controller to Aggregator Call Chain

### 2.1 Controller extracts `QuotationRequest` from payload

```java
final QuotationRequest premiumRequest = buildQuotationRequest(
        request, productCode, tenant, broker, partnerId, provider
);
return quotationService.generateQuote(premiumRequest)
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/controllers/SachetController.java:97`

### 2.2 Service stores request, then routes to life aggregator

```java
if ("life".equalsIgnoreCase(request.getProductCode())) {
    return Flux.merge(validatePremium(request)
            .map(pr -> gridPremiumAggregatorFactory
                    .getPremiumAggregator(request.getProductCode())
                    .generateQuote(pr)));
}
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java:92`

---

## 3) Request Flow (New Implementation)

Main method: `processLifeFlow(...)`
- file: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:113`

### 3.1 Input extraction + base validations

```java
Map<String, Object> premiumRequest = asMutableMap(request.getPremiumRequest());
Map<String, Object> lifePremiumRequest = asMutableMap(premiumRequest.get("lifePremiumRequest"));
Map<String, Object> riskInsured = asMutableMap(premiumRequest.get("riskInsured"));

setCurrencyDefault(premiumRequest, lifePremiumRequest);
ErrorResponseData requestValidationError = validateLifeRequestPayload(lifePremiumRequest);
LinkedHashSet<String> categories = toOrderedSet(lifePremiumRequest.get("categories"));
```

What it does:
- ensures `lifePremiumRequest` is present
- sets default `currency=INR` if missing
- validates `paymentFrequency`
- ensures `categories` is present

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:114`

### 3.2 Derived field calculations

```java
calculateDefaultParameters(lifePremiumRequest, riskInsured, category);

// inside calculateDefaultParameters(...)
lifePremiumRequest.put("entryAge", entryAge);
calculateMaturityAge(lifePremiumRequest, category, entryAge);
calculatePtPpt(lifePremiumRequest);
calculateInvestmentTermByPolicyTerm(lifePremiumRequest);
```

Derived values include:
- `entryAge`
- `maturityAge`
- `policyTerm`
- `premiumPaymentTerm`
- `investmentTermCode`

DOB parsing accepted for entry-age derivation:
- date only: `yyyy-MM-dd`, `dd-MM-yyyy`, `dd/MM/yyyy`, `yyyy/MM/dd`
- ISO offset datetime, for example: `1987-10-11T00:00:00+05:30`

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:139`

### 3.3 Partner + AB config fetch

```java
Map<String, Object> partnerDetails = fetchPartnerDetails(request, premiumRequest);
Map<String, Object> abConfig = fetchLifeAbTestingConfig(request.getBroker());
```

Data sources:
- `partnerIntegrationList`
- `LifeABTestingConfig` (`feature=lifeValidatorConfig`, with broker filter)

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:141`

### 3.4 Product catalogue call + product master build

```java
ProductQueryParam queryParam = buildProductQueryParam(request, lifePremiumRequest, categories);
List<Map<String, Object>> enabledProductMasters = fetchEnabledProductMasters(queryParam, request.getBroker());
```

Catalogue endpoint called:
- `/api/product-management/v1/products/details/filters`

What happens:
- builds request using category/planType/insurer/paymentFrequency/currency/selectedPlans
- fetches live products
- transforms PDP + variant + integration info into internal product master rows
- filters rows again against query params

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:664`

### 3.5 AB whitelist behavior

```java
enabledProductMasters = applyAbWhitelisting(enabledProductMasters, abConfig);

boolean active = getBoolean(abConfig, "active", false);
List<Map<String, Object>> whitelistedProducts = asListOfMap(conditions.get("lifeValidatorConfig"));
if (active || whitelistedProducts.isEmpty()) {
    return enabledProductMasters;
}
```

Actual behavior in current implementation:
- if `active=true`, no whitelist filtering (all enabled products continue)
- if `active=false` and whitelist exists, keeps only whitelisted `productCode-optionCode`

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:869`

### 3.6 Payout + plan-feature filtering

```java
maybeSetPayoutDefaultsForTurtlemint(request.getBroker(), lifePremiumRequest, enabledProductMasters);
enabledProductMasters = postFilterProductMaster(lifePremiumRequest, enabledProductMasters);

List<Map<String, Object>> planFeatureDetailsList = extractPlanFeatureDetails(lifePremiumRequest);
enabledProductMasters = filterProductsForFeatures(enabledProductMasters, planFeatureDetailsList);
```

What it does:
- broker-specific payout defaults (for turtlemint)
- filters by payout type/term/frequency when applicable
- filters products by requested plan features

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:153`

### 3.7 Validation request + nearest match

```java
Map<String, Object> validationRequest = createValidationRequest(
        category, lifePremiumRequest, productCodes, partnerDetails, request.getBroker()
);
List<Map<String, Object>> validatorRows = createLifeValidatorRows(enabledProductMasters, validationRequest);
validatorRows = applyNearestMatch(validationRequest, validatorRows);
```

Nearest match endpoint:
- `/api/product-management/v1/life-validator`

What it does:
- builds validator rows from enabled products
- calls nearest match API
- updates PT/PPT/score/variant-option using nearest result
- rows without nearest match are dropped

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:170`

### 3.8 Riders + offers enrichment

```java
Map<String, Map<String, Object>> riderMetaMap = fetchEligibleRiderRows(productValidRowsMap, validationRequest, enabledProductMap, request.getBroker());
Map<String, Map<String, Object>> offerMetaMap = fetchEligibleOfferRows(productValidRowsMap, enabledProductMap, request.getBroker());
```

Data sources:
- riders: `lifeRiderMaster`
- offers: `lifeOfferMaster`

Optional rule-engine calls (if configured):
- `/api/rules/v0/life/riders/slab`
- `/api/rules/v0/life/riders/validate`

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:1287`

---

## 4) `validationMap` Creation and Result Trigger (Same API)

After validation rows are prepared, the same API continues directly into result execution.

```java
Map<String, Map<String, Object>> validationMap = createValidationMap(validatedProductRows);
List<Map<String, Object>> premiumResults = createPremiumResults(
        validatedProductRows, request, premiumRequest, lifePremiumRequest, riskInsured
);
List<String> pendingKeyList = computePendingKeyList(premiumResults);

QuotationResponse response = buildWrappedLifeResponse(
        request, premiumRequest, validationMap, premiumResults, pendingKeyList, errorMessage
);
persistLifePremiumResults(request, premiumRequest, premiumResults, pendingKeyList, errorMessage);
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:199`

How `validationMap` is built:

```java
for (Map.Entry<String, List<Map<String, Object>>> entry : validatedProductRows.entrySet()) {
    String productCode = entry.getKey();
    Map<String, Object> validatorNm = asMutableMap(entry.getValue().get(0).get("lifeRequestValidatorNM"));
    validationMap.put(productCode, Map.of(
            "valid", true,
            "key", productCode,
            "insurerCode", getString(validatorNm, "insurerCode", null)
    ));
}
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:1694`

---

## 5) Result Flow (One-by-One by `productCode`)

Yes, result generation is one-by-one by product key.

Core loop:

```java
for (Map.Entry<String, List<Map<String, Object>>> entry : validatedProductRows.entrySet()) {
    String productCode = entry.getKey();
    List<Map<String, Object>> validRows = entry.getValue();

    List<Map<String, Object>> responses = new ArrayList<>();
    for (Map<String, Object> row : validRows) {
        Map<String, Object> rowWithRequestOverrides = applyRequestedOptionOverrides(row, lifePremiumRequest);
        responses.add(createLifePremiumResponse(rowWithRequestOverrides, premiumRequest, lifePremiumRequest, riskInsured, request));
    }

    Map<String, Object> mainResponse = pickMainResponse(responses);
    mainResponse.put("responseOptions", responseOptions);
    premiumResults.add(premiumResult);
}
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:1717`

Inside `createLifePremiumResponse(...)`:
- computes local premium/tax defaults
- merges rider and offer selections
- calls external provider via IH
- merges provider response, else marks `ERROR`

External call:

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

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:1984`

---

## 6) Pending Keys and Poller Flow

### 6.1 How `pendingKeyList` is computed

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

Also checks `responseOptions` for pending statuses.

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:2326`

### 6.2 Poller processing logic

Main method: `processLifeQuotePoller(...)`
- reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:209`

Flow:
1. validate `referenceId` + `quoteId`
2. read saved rows from `sachetPremiumResponse` for `referenceId + quoteId + productCode=life`
3. extract persisted `productDetails.premiumResult`
4. for rows in pending state, re-call provider
5. update saved result row
6. recompute `pendingKeyList`
7. return same response shape as create quote API

DB fetch snippet:

```java
conditions.put("referenceId", request.getReferenceId());
conditions.put("quoteId", request.getQuoteId());
conditions.put("productCode", "life");

List<QuotationResult> storedResults = quotationResultRepository
        .findAllByProperties(conditions)
        .collectList()
        .block();
```

Reference: `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:214`

---

## 7) DB Persistence: Request + Result

## 7.1 Request writes

Request is saved before aggregator execution:

```java
Mono<QuotationRequest> pr = quotationRequestRepository.saveData(req);
...
savePremiumRequestHistory(r).subscribe();
```

References:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java:242`
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/impl/IQuotationServiceImpl.java:256`

Collections:
- `sachetPremiumRequest`
- `sachetPremiumRequestHistory`

Constant source:
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/constant/DBColConstants.java:287`

## 7.2 Result writes

Result rows are persisted from life aggregator:

```java
QuotationResult quotationResult = createLifeQuotationResultRecord(...);
quotationResultRepository.saveData(quotationResult).block();
```

Reference:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:2361`

Collection:
- `sachetPremiumResponse`

Constant source:
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/constant/DBColConstants.java:289`

## 7.3 Unique IDs and linkage between request/result

Primary linkage fields:
- `referenceId` (journey-level id)
- `quoteId` (quote-run id)

Result upsert key used by aggregator:
- `referenceId + quoteId + provider + planCode`

Reference:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:2384`

Also persisted in each result document under `productDetails`:
- `premiumRequest`
- `premiumResult`
- `pendingKeyList`
- `errorMessage`

Reference:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:2481`

## 7.4 How to find request/result in DB

```javascript
// request
 db.sachetPremiumRequest.find({ referenceId: "REF123" }).sort({ createdAt: -1 })

// request history
 db.sachetPremiumRequestHistory.find({ referenceId: "REF123" }).sort({ createdAt: -1 })

// life quote results for one run
 db.sachetPremiumResponse.find({
   referenceId: "REF123",
   quoteId: "QID123",
   productCode: "life"
 }).sort({ updatedAt: -1 })

// one insurer/plan result row
 db.sachetPremiumResponse.find({
   referenceId: "REF123",
   quoteId: "QID123",
   provider: "HDFCLI",
   planCode: "P31"
 })
```

---

## 8) Mandatory vs Optional Data Dependencies

## 8.1 Mandatory input for API to work

- `premiumRequest.lifePremiumRequest`
- `categories` inside `premiumRequest.lifePremiumRequest`
- valid `paymentFrequency` if present

## 8.2 Mandatory runtime dependencies

These are functionally required for meaningful quote output:
- product catalogue API (`/api/product-management/v1/products/details/filters`) must return eligible products
- life-validator API (`/api/product-management/v1/life-validator`) must return nearest-matchable options
- IH provider integration must be configured for insurer call execution

If no eligible/validated rows survive, API returns valid wrapper with empty results and message `No matching plans found`.

## 8.3 Optional/local collections (flow still runs with fallback)

- `LifeABTestingConfig`
  - if missing, default config is created in-memory (`active=true`, empty conditions)
- `partnerIntegrationList`
  - if missing, fallback partner map is used
- `lifeRiderMaster`
  - missing data reduces rider enrichment
- `lifeOfferMaster`
  - missing data reduces offer enrichment
- rider rule APIs
  - executed only when rule-engine base URL is configured

---

## 9) Final Response Structure

Response envelope is always `PayloadWrapper`:

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

`QuotationResponse` for life quote contains:
- `productCode`
- `referenceId`
- `quoteId`
- `status` (service-level response status)
- `productDetails`

`productDetails` includes:
- `premiumRequest`
- `valid`, `status`, `errorMessage`
- `validationMap`
- `pendingKeyList`
- `premiumResults`
- `minPremium`
- `validationResponse`
- `resultResponse`

Creation code:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:487`

### 9.1 Example response (pending)

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF123",
    "quoteId": "QID123",
    "status": "success",
    "productDetails": {
      "status": "valid",
      "valid": true,
      "errorMessage": null,
      "validationMap": {
        "P31": {
          "valid": true,
          "key": "P31",
          "insurerCode": "HDFCLI"
        }
      },
      "pendingKeyList": ["P31"],
      "premiumResults": [
        {
          "key": "P31",
          "status": "PENDING",
          "insurer": "HDFCLI",
          "lifePremiumResponse": {
            "status": "PENDING",
            "insurerStatus": "PENDING",
            "responseOptions": []
          }
        }
      ]
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false
  }
}
```

### 9.2 Example response (poller complete)

When poller resolves everything:
- `pendingKeyList: []`
- each `premiumResults[*].lifePremiumResponse.status` is `SUCCESS` or `ERROR`

---

## 10) Error Response Pattern

If request validation fails or poller input is invalid, response is returned in the same wrapper with failure meta.

Example:

```json
{
  "data": {
    "errorCode": "INVALID_REQUEST",
    "errors": [
      {
        "field": "data",
        "message": "referenceId and quoteId are mandatory for life quote poller"
      }
    ]
  },
  "meta": {
    "status": "FAILURE",
    "error": true
  }
}
```
