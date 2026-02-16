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

Pending behavior clarification:
- If the selected/main quote for a product is pending, both are true:
  - product key is added in `pendingKeyList`
  - `premiumResults[*].status` and `premiumResults[*].lifePremiumResponse.status` are `PENDING`
- If selected/main quote is success but any variant inside `lifePremiumResponse.responseOptions` is pending:
  - product key is still added in `pendingKeyList`
  - main `lifePremiumResponse.status` can remain `SUCCESS`
  - pending status is visible in `lifePremiumResponse.responseOptions[*].status`

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

Create quote and poller APIs both return `PayloadWrapper`:

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF123",
    "quoteId": "QID123",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "isAsync": true,
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "timestamp": "2026-02-16T04:54:35.749Z",
        "currency": "INR",
        "riskInsured": {
          "dateOfBirth": "1987-10-11T00:00:00+05:30"
        },
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12,
          "policyTerm": 10,
          "premiumPaymentTerm": 10,
          "premium": 70000
        }
      },
      "valid": true,
      "status": "valid",
      "errorMessage": null,
      "validationMap": {},
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": [],
      "premiumResults": [],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": null,
      "validationResponse": {
        "premiumRequest": {},
        "status": "valid",
        "valid": true,
        "validationMap": {},
        "errorMessage": null
      },
      "resultResponse": {
        "premiumRequest": {},
        "premiumResults": [],
        "pendingKeyList": []
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "...",
    "timestamp": "..."
  }
}
```

Creation code:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/src/main/java/com/sachetProduct/service/aggregator/SachetLifeAggregator.java:487`

### 9.1 Full example: pending quote

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF123",
    "quoteId": "QID123",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "isAsync": true,
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "currency": "INR",
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12
        }
      },
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
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": ["P31"],
      "premiumResults": [
        {
          "_id": "ff67ce7f-6fb6-4f5f-a0ff-7e95fdbb7438",
          "uniqueId": "QID123",
          "requestId": "REF123",
          "key": "P31",
          "quoteId": "QID123",
          "status": "PENDING",
          "insurer": "HDFCLI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T04:54:35.749Z",
          "tax": "0.0",
          "finalPremium": "0.0",
          "lifePremiumResponse": {
            "resultId": "3e4b8fa3-22e7-4e5b-b6e9-7e0a0d5cfc10",
            "status": "PENDING",
            "insurerStatus": "PENDING",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "HDFCLI",
            "productCode": "P31",
            "internalProductCode": "P31",
            "productName": "Life Investment Plan",
            "option": "Default",
            "optionCode": -1,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "premium": null,
            "tax": null,
            "premiumWithTax": null,
            "responseOptions": []
          }
        }
      ],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": null,
      "validationResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "isAsync": true,
          "vertical": "LIFE"
        },
        "status": "valid",
        "valid": true,
        "validationMap": {
          "P31": {
            "valid": true,
            "key": "P31",
            "insurerCode": "HDFCLI"
          }
        },
        "errorMessage": null
      },
      "resultResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "isAsync": true,
          "vertical": "LIFE"
        },
        "pendingKeyList": ["P31"],
        "premiumResults": [
          {
            "_id": "ff67ce7f-6fb6-4f5f-a0ff-7e95fdbb7438",
            "key": "P31",
            "status": "PENDING"
          }
        ]
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "a6e32aa4c8",
    "timestamp": "2026-02-16T04:54:35.801"
  }
}
```

### 9.2 Full example: success quote

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF123",
    "quoteId": "QID123",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "isAsync": true,
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "currency": "INR",
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12
        }
      },
      "valid": true,
      "status": "valid",
      "errorMessage": null,
      "validationMap": {
        "P31": {
          "valid": true,
          "key": "P31",
          "insurerCode": "HDFCLI"
        }
      },
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": [],
      "premiumResults": [
        {
          "_id": "f6ff9d49-9f45-4f5f-ae9b-c7a2a7fbe123",
          "uniqueId": "QID123",
          "requestId": "REF123",
          "key": "P31",
          "quoteId": "QID123",
          "status": "SUCCESS",
          "insurer": "HDFCLI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T04:54:36.912Z",
          "tax": "10677.97",
          "finalPremium": "69999.99",
          "lifePremiumResponse": {
            "resultId": "8f0c9ae2-b6e2-4cf4-bce2-b8501aaf236f",
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "HDFCLI",
            "productCode": "P31",
            "internalProductCode": "P31",
            "productName": "Life Investment Plan",
            "option": "Default",
            "optionCode": -1,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "age": 38,
            "sumAssured": 1000000.0,
            "premium": 59322.02,
            "tax": 10677.97,
            "premiumWithTax": 69999.99,
            "score": 0,
            "riderList": [],
            "offerList": [],
            "responseOptions": []
          }
        }
      ],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": 70000,
      "validationResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "isAsync": true,
          "vertical": "LIFE"
        },
        "status": "valid",
        "valid": true,
        "validationMap": {
          "P31": {
            "valid": true,
            "key": "P31",
            "insurerCode": "HDFCLI"
          }
        },
        "errorMessage": null
      },
      "resultResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "isAsync": true,
          "vertical": "LIFE"
        },
        "pendingKeyList": [],
        "premiumResults": [
          {
            "_id": "f6ff9d49-9f45-4f5f-ae9b-c7a2a7fbe123",
            "key": "P31",
            "status": "SUCCESS"
          }
        ]
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "0e128f82f6",
    "timestamp": "2026-02-16T04:54:36.939"
  }
}
```

### 9.3 Full example: success with rider and variant option

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF456",
    "quoteId": "QID456",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12,
          "riderMeta": [
            {
              "optionCode": "2",
              "riderInfoList": [
                {
                  "riderCode": "ADB",
                  "isSelected": true
                }
              ]
            }
          ]
        }
      },
      "valid": true,
      "status": "valid",
      "errorMessage": null,
      "validationMap": {
        "P51": {
          "valid": true,
          "key": "P51",
          "insurerCode": "ICICILI"
        }
      },
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": [],
      "premiumResults": [
        {
          "_id": "7bc2a2f1-a6a3-4a63-bcb2-319db2c77f1f",
          "uniqueId": "QID456",
          "requestId": "REF456",
          "key": "P51",
          "quoteId": "QID456",
          "status": "SUCCESS",
          "insurer": "ICICILI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T05:10:01.021Z",
          "tax": "11160.00",
          "finalPremium": "73160.00",
          "lifePremiumResponse": {
            "resultId": "72a7d368-6f40-4e56-a26f-83a5b24cbba2",
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "ICICILI",
            "productCode": "P51",
            "internalProductCode": "P51",
            "productName": "Guaranteed Wealth Pro",
            "option": "Gold Variant",
            "optionCode": 2,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "premium": 62000.0,
            "tax": 11160.0,
            "premiumWithTax": 73160.0,
            "score": 0,
            "riderList": [
              {
                "riderCode": "ADB",
                "riderName": "Accidental Death Benefit",
                "isSelected": true,
                "isInBuilt": false,
                "riderPolicyTerm": 10,
                "riderPremiumPaymentTerm": 10,
                "riderSumAssured": 250000.0,
                "riderPremium": 1800.0
              }
            ],
            "offerList": [
              {
                "offerCode": "LOYALTYBONUS",
                "offerName": "Loyalty Bonus",
                "isSelected": true
              }
            ],
            "responseOptions": [
              {
                "status": "SUCCESS",
                "insurerStatus": "SUCCESS",
                "insurerCode": "ICICILI",
                "productCode": "P51",
                "option": "Silver Variant",
                "optionCode": 1,
                "premium": 60500.0,
                "tax": 10890.0,
                "premiumWithTax": 71390.0,
                "riderList": [],
                "offerList": []
              }
            ]
          }
        }
      ],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": 73160,
      "validationResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "status": "valid",
        "valid": true,
        "validationMap": {
          "P51": {
            "valid": true,
            "key": "P51",
            "insurerCode": "ICICILI"
          }
        },
        "errorMessage": null
      },
      "resultResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "pendingKeyList": [],
        "premiumResults": [
          {
            "_id": "7bc2a2f1-a6a3-4a63-bcb2-319db2c77f1f",
            "key": "P51",
            "status": "SUCCESS"
          }
        ]
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "4adba8f65d",
    "timestamp": "2026-02-16T05:10:01.042"
  }
}
```

### 9.4 Full example: failure quote

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF789",
    "quoteId": "QID789",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12
        }
      },
      "valid": true,
      "status": "valid",
      "errorMessage": "Some quotes failed at provider",
      "validationMap": {
        "P99": {
          "valid": true,
          "key": "P99",
          "insurerCode": "BAJAJLI"
        }
      },
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": [],
      "premiumResults": [
        {
          "_id": "ac7f691f-3bd5-49f2-bcea-89c2e88d22d2",
          "uniqueId": "QID789",
          "requestId": "REF789",
          "key": "P99",
          "quoteId": "QID789",
          "status": "ERROR",
          "insurer": "BAJAJLI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T05:21:32.145Z",
          "tax": "0.0",
          "finalPremium": "0.0",
          "lifePremiumResponse": {
            "resultId": "dfc707fd-4b30-488a-906c-bdc84d0e70b6",
            "status": "ERROR",
            "insurerStatus": "ERROR",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "BAJAJLI",
            "productCode": "P99",
            "internalProductCode": "P99",
            "productName": "Long Term Wealth Plus",
            "option": "Default",
            "optionCode": -1,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "premium": null,
            "tax": null,
            "premiumWithTax": null,
            "insurerMessage": "INTERNAL_LIFE_SERVICE_ERROR",
            "responseOptions": []
          }
        }
      ],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": null,
      "validationResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "status": "valid",
        "valid": true,
        "validationMap": {
          "P99": {
            "valid": true,
            "key": "P99",
            "insurerCode": "BAJAJLI"
          }
        },
        "errorMessage": "Some quotes failed at provider"
      },
      "resultResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "pendingKeyList": [],
        "premiumResults": [
          {
            "_id": "ac7f691f-3bd5-49f2-bcea-89c2e88d22d2",
            "key": "P99",
            "status": "ERROR"
          }
        ]
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "f55b7d672e",
    "timestamp": "2026-02-16T05:21:32.175"
  }
}
```

### 9.5 Full example: multiple products in same response

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "REF901",
    "quoteId": "QID901",
    "status": "success",
    "productDetails": {
      "premiumRequest": {
        "requestType": "INITIAL",
        "vertical": "LIFE",
        "policyType": "TRADITIONAL",
        "lifePremiumRequest": {
          "categories": ["investment"],
          "paymentFrequency": 12
        }
      },
      "valid": true,
      "status": "valid",
      "errorMessage": "Some quotes are still pending",
      "validationMap": {
        "P31": {
          "valid": true,
          "key": "P31",
          "insurerCode": "HDFCLI"
        },
        "P51": {
          "valid": true,
          "key": "P51",
          "insurerCode": "ICICILI"
        }
      },
      "renewalsValid": false,
      "renewalsKey": null,
      "errorDescription": null,
      "healthAddOnValidationInfo": null,
      "pendingKeyList": ["P51"],
      "premiumResults": [
        {
          "_id": "d7e9311a-8c6d-4ce9-93f8-369ec4b2752a",
          "uniqueId": "QID901",
          "requestId": "REF901",
          "key": "P31",
          "quoteId": "QID901",
          "status": "SUCCESS",
          "insurer": "HDFCLI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T06:03:44.121Z",
          "tax": "10200.00",
          "finalPremium": "66866.67",
          "lifePremiumResponse": {
            "resultId": "9bd8f2da-f2be-4f3f-b916-68de24990d99",
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "HDFCLI",
            "productCode": "P31",
            "internalProductCode": "P31",
            "productName": "Life Investment Plan",
            "option": "Default",
            "optionCode": -1,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "premium": 56666.67,
            "tax": 10200.0,
            "premiumWithTax": 66866.67,
            "riderList": [],
            "offerList": [],
            "responseOptions": []
          }
        },
        {
          "_id": "2a9a7b32-0e97-42b6-ab21-53eb1c4ee77f",
          "uniqueId": "QID901",
          "requestId": "REF901",
          "key": "P51",
          "quoteId": "QID901",
          "status": "PENDING",
          "insurer": "ICICILI",
          "vertical": "LIFE",
          "timestamp": "2026-02-16T06:03:44.129Z",
          "tax": "0.0",
          "finalPremium": "0.0",
          "lifePremiumResponse": {
            "resultId": "1f5d7df2-7f58-4c2a-bdad-403c5cc61d74",
            "status": "PENDING",
            "insurerStatus": "PENDING",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "insurerCode": "ICICILI",
            "productCode": "P51",
            "internalProductCode": "P51",
            "productName": "Guaranteed Wealth Pro",
            "option": "Gold Variant",
            "optionCode": 2,
            "planType": "TRADITIONAL",
            "policyTerm": 10,
            "premiumPaymentTerm": 10,
            "paymentFrequency": 12,
            "premium": null,
            "tax": null,
            "premiumWithTax": null,
            "responseOptions": []
          }
        }
      ],
      "motorPremiumResult": null,
      "lifePremiumResults": null,
      "healthPremiumResults": null,
      "errorMsg": null,
      "minPremium": 66867,
      "validationResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "status": "valid",
        "valid": true,
        "validationMap": {
          "P31": {
            "valid": true,
            "key": "P31",
            "insurerCode": "HDFCLI"
          },
          "P51": {
            "valid": true,
            "key": "P51",
            "insurerCode": "ICICILI"
          }
        },
        "errorMessage": "Some quotes are still pending"
      },
      "resultResponse": {
        "premiumRequest": {
          "requestType": "INITIAL",
          "vertical": "LIFE"
        },
        "pendingKeyList": ["P51"],
        "premiumResults": [
          {
            "_id": "d7e9311a-8c6d-4ce9-93f8-369ec4b2752a",
            "key": "P31",
            "status": "SUCCESS"
          },
          {
            "_id": "2a9a7b32-0e97-42b6-ab21-53eb1c4ee77f",
            "key": "P51",
            "status": "PENDING"
          }
        ]
      }
    }
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "6b7a12832a",
    "timestamp": "2026-02-16T06:03:44.166"
  }
}
```

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
