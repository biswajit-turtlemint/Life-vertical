# Life Quote FE Spec (Exact Payloads Only)

This document uses only the payload fields shared by you.
No extra payload fields are added.

APIs:
- `POST /api/minterprise/v2/products/life/quotes`
- `GET /api/minterprise/v2/products/life/quotes?referenceId=...&includeRequest=true|false`
- `GET /api/minterprise/v2/products/life/quotes/poll?referenceId=...&resultIds=...`

---

## 1) Category + self/non-self payloads (exact)

## 1.1 Term, planType=Term

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userMobile": "7400400747",
        "userEmail": "SHAHABID37@GMAIL.COM"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "1972-05-03T00:00:00+05:30",
            "entryAge": 53,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "policyType": "TERM",
        "planType": "Term",
        "categories": [
          "term"
        ],
        "coverAmount": 10000000,
        "paymentFrequency": 12,
        "profileType": "term",
        "businessModel": "B2B",
        "investmentRisk": "high",
        "investmentTermCode": "medium",
        "benifitCalculationRate": 8,
        "tmOccupation": "salaried",
        "tmQualification": "Graduate_Post_Graduate",
        "isSmoking": false,
        "maritalStatus": "SINGLE",
        "incomeBracketCode": "7 Lakhs",
        "minIncome": 500000,
        "maxIncome": 699999,
        "tmPincode": 192125
      },
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "requestType": "INITIAL",
      "vertical": "LIFE",
      "timestamp": "2026-02-18T21:08:18.969Z"
    }
  }
}
```

## 1.2 Guaranteed, non-self journey (`isNonSelfJourney=true`)

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {
        "propAge": 53,
        "propDOB": "1972-05-03T00:00:00+05:30",
        "propFullName": "ABID AHMAD SHAH",
        "propGender": "M"
      },
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "biswajit rout",
            "dateOfBirth": "2000-12-16T00:00:00+05:30",
            "entryAge": 25,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "guaranteed"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": true,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "planType": "Non-participating",
        "policyTerm": 10,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "guaranteed-returns",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "TRADITIONAL",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T21:41:31.199Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.3 Guaranteed, self journey (`isNonSelfJourney=false`)

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "1972-05-03T00:00:00+05:30",
            "entryAge": 53,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "guaranteed"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": false,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "planType": "Participating",
        "policyTerm": 10,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "guaranteed-returns",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "TRADITIONAL",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T21:45:24.432Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.4 Ulip category, non-self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {
        "propAge": 53,
        "propDOB": "1972-05-03T00:00:00+05:30",
        "propFullName": "ABID AHMAD SHAH",
        "propGender": "M"
      },
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "biswajit rout",
            "dateOfBirth": "2000-02-02T00:00:00+05:30",
            "entryAge": 53,
            "gender": "F"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "ulip"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "7 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": true,
        "maritalStatus": "SINGLE",
        "maxIncome": 700000,
        "minIncome": 700000,
        "paymentFrequency": 12,
        "planType": "Non-participating",
        "policyTerm": 9,
        "premium": 70000,
        "premiumPaymentTerm": 8,
        "profileType": "ulip",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "TRADITIONAL",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T22:54:06.630Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.5 Ulip category, self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "1972-05-03T00:00:00+05:30",
            "entryAge": 53,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "ulip"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": false,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "planType": "ULIP",
        "policyTerm": 10,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "ulip",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "ULIP",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T22:55:59.609Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.6 Participating category, non-self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {
        "propAge": 53,
        "propDOB": "1972-05-03T00:00:00+05:30",
        "propFullName": "ABID AHMAD SHAH",
        "propGender": "M"
      },
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "biswajit rout",
            "dateOfBirth": "2000-02-01T00:00:00+05:30",
            "entryAge": 26,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "participating"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": true,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "planType": "Participating",
        "policyTerm": 8,
        "premium": 70000,
        "premiumPaymentTerm": 6,
        "profileType": "participating-plans",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "TRADITIONAL",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T23:02:41.840Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.7 Participating category, self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "1972-05-03T00:00:00+05:30",
            "entryAge": 53,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "participating"
        ],
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "WEALTH_CREATION",
        "investmentRisk": "high",
        "isNonSelfJourney": false,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "planType": "Participating",
        "policyTerm": 9,
        "premium": 70000,
        "premiumPaymentTerm": 8,
        "profileType": "participating-plans",
        "riskAppetite": "MEDIUM"
      },
      "policyType": "TRADITIONAL",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T23:05:21.286Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.8 Child category (always non-self)

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userMobile": "7400400747",
        "userEmail": "SHAHABID37@GMAIL.COM"
      },
      "proposerDetails": {
        "propFullName": "ABID AHMAD SHAH",
        "propGender": "M",
        "propDOB": "1972-05-03T00:00:00+05:30",
        "propAge": 53
      },
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "biswajit rout",
            "dateOfBirth": "2015-02-04T00:00:00+05:30",
            "entryAge": 11,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "categories": [
          "child"
        ],
        "planType": "Non-participating",
        "includeCategory": true,
        "maritalStatus": "SINGLE",
        "investmentGoals": "CHILD_EDUCATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "6 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 600000,
        "minIncome": 600000,
        "paymentFrequency": 12,
        "policyTerm": 10,
        "premiumPaymentTerm": 9,
        "premium": 70000,
        "profileType": "saving-for-child",
        "businessModel": "B2B",
        "isNonSelfJourney": true
      },
      "requestType": "INITIAL",
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-18T23:13:52.776Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "initialReqFlag": true
    }
  }
}
```

## 1.9 Retirement/Pension, non-self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {
        "propAge": 53,
        "propDOB": "1972-05-03T00:00:00+05:30",
        "propFullName": "ABID AHMAD SHAH",
        "propGender": "M"
      },
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "biswajit rout",
            "dateOfBirth": "2000-02-02T00:00:00+05:30",
            "entryAge": 26,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "retirement"
        ],
        "defermentPeriod": 3,
        "existingInsuranceCoverAmount": 0,
        "includeCategory": true,
        "incomeBracketCode": "6 Lakhs",
        "investmentGoals": "RETIREMENT",
        "investmentRisk": "high",
        "isJointLife": false,
        "isNonSelfJourney": true,
        "maritalStatus": "SINGLE",
        "maxIncome": 600000,
        "minIncome": 600000,
        "paymentFrequency": 12,
        "payoutFrequency": "YEARLY",
        "payoutIncome": 100000,
        "planType": "Pension",
        "plansByPension": false,
        "policyTerm": 15,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "retirement",
        "riskAppetite": "MEDIUM"
      },
      "initialReqFlag": true,
      "isAsync": true,
      "policyType": "PENSION",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T23:20:24.500Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

## 1.10 Retirement/Pension, self journey

```json
{
  "data": {
    "premiumRequest": {
      "personalDetails": {
        "customerName": "ABID AHMAD SHAH",
        "userEmail": "SHAHABID37@GMAIL.COM",
        "userMobile": "7400400747"
      },
      "proposerDetails": {},
      "riskInsured": {
        "insuredMembers": [
          {
            "insuredFullName": "ABID AHMAD SHAH",
            "dateOfBirth": "1972-05-03T00:00:00+05:30",
            "entryAge": 53,
            "gender": "M"
          }
        ]
      },
      "planDetails": {
        "benifitCalculationRate": 8,
        "businessModel": "B2B",
        "categories": [
          "retirement"
        ],
        "defermentPeriod": 5,
        "existingInsuranceCoverAmount": 0,
        "includeCategory": true,
        "incomeBracketCode": "5 Lakhs",
        "investmentGoals": "RETIREMENT",
        "investmentRisk": "high",
        "isJointLife": false,
        "isNonSelfJourney": false,
        "maritalStatus": "SINGLE",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "payoutFrequency": "YEARLY",
        "payoutIncome": 100000,
        "planType": "Pension",
        "plansByPension": false,
        "policyTerm": 15,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "retirement",
        "riskAppetite": "MEDIUM"
      },
      "initialReqFlag": true,
      "isAsync": true,
      "policyType": "PENSION",
      "requestType": "INITIAL",
      "timestamp": "2026-02-18T23:21:27.816Z",
      "utmParams": {
        "utmMedium": "(none)",
        "utmSource": "(direct)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "vertical": "LIFE"
    }
  }
}
```

---

## 2) Final response structure (for FE)

Wrapper:

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

`data` fields:
- `productCode`
- `referenceId`
- `status`
- `error`
- `pendingKeyList`
- `quotes`
- `transactionSource`
- `transactionMode`

`data.quotes[*]` fields:
- `_id`, `referenceId`, `provider`, `productCode`, `planCode`, `quoteId`
- `partnerId`, `tenant`, `broker`, `status`
- `leadName`, `insurerName`, `insurerCode`
- `premium`, `tax`, `totalPremium`, `netPremium`, `currency`, `sumInsured`, `policyTerm`
- `riskInsured`, `planName`, `stage`
- `validationMap`, `pendingKeyList`, `errorMessage`
- `premiumResponse` (renamed response field)

`premiumResponse` contains:
- identity/status fields (`status`, `insurerStatus`, `insurerCode`, `productCode`, etc.)
- premium fields (`premium`, `tax`, `premiumWithTax`, etc.)
- variant fields (`option`, `optionCode`, `responseOptions[]`)
- rider fields (`riderList`, `totalRiderPremium`, etc.)
- `companyDetails`

---

## 3) Response examples

### 3.1 Pending

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "AHU-PEND-1",
    "pendingKeyList": ["icici-term"],
    "quotes": [
      {
        "_id": "RID-P-1",
        "quoteId": "QID-P-1",
        "status": "pending",
        "planCode": "icici-term",
        "premiumResponse": {
          "status": "PENDING",
          "insurerStatus": "PENDING",
          "productCode": "icici-term",
          "optionCode": -1,
          "companyDetails": {}
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

### 3.2 Success with rider + variant + companyDetails

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "AHU-S-1",
    "pendingKeyList": [],
    "quotes": [
      {
        "_id": "RID-S-1",
        "quoteId": "QID-S-1",
        "status": "success",
        "planCode": "hdfc-guaranteed",
        "premiumResponse": {
          "status": "SUCCESS",
          "insurerStatus": "SUCCESS",
          "insurerCode": "HDFC",
          "productCode": "hdfc-guaranteed",
          "option": "Base",
          "optionCode": 0,
          "premium": 70000,
          "tax": 12600,
          "premiumWithTax": 82600,
          "riderList": [
            {
              "riderCode": "ADB",
              "isSelected": true,
              "riderPremium": 2100
            }
          ],
          "responseOptions": [
            {
              "option": "Saver",
              "optionCode": 1,
              "status": "SUCCESS",
              "premiumWithTax": 80450
            }
          ],
          "companyDetails": {
            "insurerCode": "HDFC",
            "displayName": "HDFC Life",
            "logo": "https://..."
          }
        }
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

### 3.3 Failure

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "AHU-F-1",
    "pendingKeyList": [],
    "quotes": [
      {
        "_id": "RID-F-1",
        "quoteId": "QID-F-1",
        "status": "failed",
        "planCode": "max-ulip",
        "errorMessage": "INTERNAL_LIFE_SERVICE_ERROR",
        "premiumResponse": {
          "status": "ERROR",
          "insurerStatus": "ERROR",
          "insurerCode": "MAX",
          "productCode": "max-ulip",
          "companyDetails": {}
        }
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

### 3.4 Multiple quotes (mixed)

```json
{
  "data": {
    "productCode": "life",
    "referenceId": "AHU-MIX-1",
    "pendingKeyList": ["icici-term"],
    "quotes": [
      {
        "_id": "RID-M-1",
        "quoteId": "QID-M-1",
        "status": "success",
        "planCode": "hdfc-guaranteed",
        "premiumResponse": { "status": "SUCCESS", "optionCode": 0 }
      },
      {
        "_id": "RID-M-2",
        "quoteId": "QID-M-1",
        "status": "pending",
        "planCode": "icici-term",
        "premiumResponse": { "status": "PENDING", "optionCode": -1 }
      },
      {
        "_id": "RID-M-3",
        "quoteId": "QID-M-1",
        "status": "failed",
        "planCode": "max-ulip",
        "premiumResponse": { "status": "ERROR", "optionCode": 2 }
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

---

## 4) GET /quotes and /poll

- `GET /quotes` response is same shape as POST response.
- with `includeRequest=true`, response additionally includes `data.premiumRequest`.
- `GET /quotes/poll` response is same shape as POST response.
- FE keeps polling until `pendingKeyList` is empty.
