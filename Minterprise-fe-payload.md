# Life Quote FE Spec (Exact Payloads Only)

This document uses only the payload fields shared by you.
No extra payload fields are added.
`policyType` must be sent inside `data.premiumRequest.planDetails.policyType`.
`pospUserName` must be sent at flat `data.premiumRequest.pospUserName`.

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
      "pospUserName": "66bb2378ae016500016e5a06",
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
      "vertical": "LIFE"
    }
  }
}
```

## 1.2 Guaranteed, non-self journey (`isNonSelfJourney=true`)

```json
{
    "data": {
        "premiumRequest": {
            "pospUserName": "647f458828effa0001044980",
            "personalDetails": {
                "customerName": "ABID AHMAD SHAH",
                "userEmail": "SHAHABID37@GMAIL.COM",
                "userMobile": "7400400747",
                "customerId": "8080515100"
            },
            "proposerDetails": {
                "propAge": 53,
                "propDOB": "03/07/1975",
                "propFullName": "ABID AHMAD SHAH",
                "propGender": "M"
            },
            "riskInsured": {
                "insuredMembers": [
                    {
                        "insuredFullName": "BISWAJIT ROUT",
                        "dateOfBirth": "16/12/2000",
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
                "policyType": "TRADITIONAL",
                "policyTerm": 10,
                "premium": 70000,
                "premiumPaymentTerm": 9,
                "profileType": "guaranteed-returns",
                "riskAppetite": "MEDIUM",
                "tmPincode": "192125",
                "tmCity": "Srinagar",
                "tmState": "JAMMU & KASHMIR"
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
            "pospUserName": "647f458828effa0001044980",
            "personalDetails": {
                "customerName": "ABID AHMAD SHAH",
                "userEmail": "SHAHABID37@GMAIL.COM",
                "userMobile": "7400400747",
                "customerId": "8080515100"
            },
            "proposerDetails": {},
            "riskInsured": {
                "insuredMembers": [
                    {
                        "insuredFullName": "BISWAJIT ROUT",
                        "dateOfBirth": "03/07/1975",
                        // "dateOfBirth": "2000-12-16T00:00:00+05:30",
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
                "isNonSelfJourney": false,
                "maritalStatus": "SINGLE",
                "maxIncome": 500000,
                "minIncome": 500000,
                "paymentFrequency": 12,
                "planType": "Non-participating",
                "policyType": "TRADITIONAL",
                "policyTerm": 10,
                "premium": 70000,
                "premiumPaymentTerm": 9,
                "profileType": "guaranteed-returns",
                "riskAppetite": "MEDIUM",
                "tmPincode": "192125",
                "tmCity": "Srinagar",
                "tmState": "JAMMU & KASHMIR"
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
      "pospUserName": "66bb2378ae016500016e5a06",
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
        "policyType": "TRADITIONAL",
        "policyTerm": 9,
        "premium": 70000,
        "premiumPaymentTerm": 8,
        "profileType": "ulip",
        "riskAppetite": "MEDIUM"
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
      "pospUserName": "66bb2378ae016500016e5a06",
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
        "policyType": "ULIP",
        "policyTerm": 10,
        "premium": 70000,
        "premiumPaymentTerm": 9,
        "profileType": "ulip",
        "riskAppetite": "MEDIUM"
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
      "pospUserName": "66bb2378ae016500016e5a06",
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
        "policyType": "TRADITIONAL",
        "policyTerm": 8,
        "premium": 70000,
        "premiumPaymentTerm": 6,
        "profileType": "participating-plans",
        "riskAppetite": "MEDIUM"
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
      "pospUserName": "66bb2378ae016500016e5a06",
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
        "policyType": "TRADITIONAL",
        "policyTerm": 9,
        "premium": 70000,
        "premiumPaymentTerm": 8,
        "profileType": "participating-plans",
        "riskAppetite": "MEDIUM"
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
            "pospUserName": "66bb2378ae016500016e5a06",
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
                "policyType": "TRADITIONAL",
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
                "isNonSelfJourney": true,
                "tmPincode": "400001",
                "tmCity": "Mumbai G.P.O.",
                "tmState": "MAHARASHTRA"
            },
            "vertical": "LIFE",
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
            "pospUserName": "66bb2378ae016500016e5a06",
            "personalDetails": {
                "customerName": "ABID AHMAD SHAH",
                "userEmail": "SHAHABID37@GMAIL.COM",
                "userMobile": "7400400747"
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
                "tmPincode": "400001",
                "tmCity": "Mumbai G.P.O.",
                "tmState": "MAHARASHTRA",
                "paymentFrequency": 12,
                "payoutFrequency": "YEARLY",
                "payoutIncome": 100000,
                "planType": "Non-participating",
                "policyType": "PENSION",
                "plansByPension": false,
                "policyTerm": 15,
                "premium": 70000,
                "premiumPaymentTerm": 9,
                "profileType": "retirement",
                "riskAppetite": "MEDIUM"
            },
            "initialReqFlag": true,
            "isAsync": true,
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
            "pospUserName": "66bb2378ae016500016e5a06",
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
                "tmPincode": "400001",
                "tmCity": "Mumbai G.P.O.",
                "tmState": "MAHARASHTRA",
                "paymentFrequency": 12,
                "payoutFrequency": "YEARLY",
                "payoutIncome": 100000,
                "planType": "Non-participating",
                "policyType": "PENSION",
                "plansByPension": false,
                "policyTerm": 15,
                "premium": 70000,
                "premiumPaymentTerm": 9,
                "profileType": "retirement",
                "riskAppetite": "MEDIUM"
            },
            "initialReqFlag": true,
            "isAsync": true,
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
- payout fields (`payoutValues`, `totalPayoutValues`, `maturityBenefits`, `planNote`, etc.)
- tax fields (`taxSavingAmount`, `taxSavingsInfo`)
- additional insurer fields (for example `resultCardsInfo`, `planFeatureList`, `specialBenefits`, etc.)

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

---

## 5) Final response update (authoritative)

This payload doc keeps request payload examples.
For full response contract, use:
- `/Users/biswajitrout/companyProjects/minterprise/transactional-flows/docs/life-quote-post-api-spec.md` section `6) Final Contract Update (Authoritative)`.

Important final response rules:
- `data.quotes` contains only `SUCCESS` rows.
- `PENDING`/`ERROR` rows are not returned in `quotes`; track them via `data.pendingKeyList` and call poll API.
- `resultId` is mapped to API `quoteId`.
- insurer `quoteId` (if present from IH for that insurer) is mapped to API `insurerQuoteId`.
- `companyDetails` is kept at parent quote level only; it is removed from `responseOptions[*]`.
- `responseOptions` supports multiple options.
- `riderList`, `payoutValues`, `totalPayoutValues`, `maturityBenefits`, `taxSavingsInfo`, and other IH fields are returned as-is when present.
- `addons`/`addOns` is removed from API response.

## 6) Complete full response JSON (authoritative)

```json
{
  "data": {
    "referenceId": "AHUTP1QQBR8",
    "pendingKeyList": [
      "P90",
      "P108",
      "P118"
    ],
    "quotes": [
      {
        "quoteId": "dc28c8a00890741fca957b2f92783599",
        "insurerQuoteId": "b8735fd6-0a0d-4e30-aa73-2ae1e0be8d96",
        "policyType": "TERM",
        "insurerCode": "MAXLIFELI",
        "internalProductCode": "maxlifeli-life-smarttermplanplus",
        "productCode": "P127",
        "productName": "Smart Term Plan Plus",
        "option": "Regular cover",
        "optionCode": 1,
        "productUIN": "104N127V02",
        "tmPlanId": "1460",
        "policyTerm": 42,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 42,
        "score": 0,
        "category": "term",
        "premium": 7845,
        "taxRate": 0,
        "premiumWithTax": 7845,
        "sumAssured": 10000000,
        "deathBenefitTotal": 10000000,
        "deathBenefitGuaranteed": 10000000,
        "responseOptions": [
          {
            "quoteId": "6d5ffe95c64e53f849a48396c049ec80",
            "insurerQuoteId": "fdee0910-ae04-4cfc-b64b-b7556ebb74e9",
            "policyType": "TERM",
            "insurerCode": "MAXLIFELI",
            "internalProductCode": "maxlifeli-life-smarttermplanplus",
            "productCode": "P127",
            "productName": "Smart Term Plan Plus",
            "option": "Early ROP Plus",
            "optionCode": 3,
            "productUIN": "104N127V02",
            "tmPlanId": "1460",
            "policyTerm": 52,
            "paymentFrequency": 12,
            "premiumPaymentTerm": 15,
            "score": 19,
            "category": "term",
            "premium": 23145,
            "taxRate": 0,
            "premiumWithTax": 23145,
            "sumAssured": 10000000,
            "deathBenefitTotal": 10000000,
            "deathBenefitGuaranteed": 10000000,
            "taxSavingAmount": 4629,
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerMessage": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "payoutValues": {
              "LUMPSUM": [
                {
                  "payoutType": "LUMPSUM",
                  "basePayoutValue": 10000000
                }
              ],
              "LUMPSUM_PLUS_LEVEL_INCOME": [
                {
                  "payoutType": "LUMPSUM",
                  "basePayoutValue": 5000000
                },
                {
                  "payoutType": "LEVEL_INCOME",
                  "basePayoutValue": 83333.33333333333,
                  "payoutFrequency": "MONTHLY",
                  "benefitTerm": 5
                }
              ]
            },
            "totalPayoutValues": {
              "LUMPSUM": 10000000,
              "LUMPSUM_PLUS_LEVEL_INCOME": 10000000
            },
            "maturityBenefits": [
              "Return of Total Premiums Paid at the end of Premium Payment Term, while life cover continues till policy term end."
            ],
            "planNote": [
              "Option to receive early Return of Premium at the end of Premium Payment Term",
              "Life cover continues till end of policy term",
              "Special Exit Value available post premium payment term"
            ],
            "payoutTerm": 5,
            "payoutFrequency": "MONTHLY",
            "planType": "Term",
            "riderList": [
              {
                "riderName": "Accidental Death and Dismemberment",
                "riderDesc": "This rider provides your loved ones with extra financial protection in case of an unexpected eventuality.",
                "riderShortDesc": "This rider provides your loved ones with extra financial protection.",
                "riderCode": "R77",
                "riderApiCode": "R77",
                "riderCategory": "Accidental Death Benefit",
                "riderSumAssured": 50000,
                "riderPolicyTerm": 15,
                "riderPremiumPaymentTerm": 15,
                "inBuilt": false,
                "isSelected": false,
                "isCoverAmountEditable": true,
                "isCoverAmountIncludedInBasePlan": false
              }
            ],
            "showRider": false,
            "showRiderPremium": true,
            "edc": "2026-02-18",
            "biProvider": "MAXLIFELI",
            "isBIPdfAvailable": true,
            "age": 18,
            "inputCoverAmount": 10000000,
            "planFeatureDetailsList": [],
            "bqpRedirectionEnabled": true,
            "paymentFirstJourneyCheck": false,
            "defermentPeriod": 0,
            "calculatedDefermentPeriod": -2,
            "incomeStartYear": -2,
            "calculatedIRR": "-",
            "cashFlowsPerYearArray": [],
            "resultCardsInfo": {
              "investmentAmount": 347175,
              "investmentPerFrequencyAmount": 23145,
              "paymentFrequency": 12,
              "entryAge": 18,
              "premiumPaymentTerm": 15,
              "policyTerm": 52,
              "maturityAge": 70,
              "deathCoverTillAge": 70,
              "deathBenefit": 10000000,
              "taxSavings": 4629,
              "returnOfPremium": false,
              "claimSettlementRatio": "99.50%",
              "incomePeriod": 5,
              "yearlyIncome": 1000000,
              "lumpsumAmount": 5000000,
              "benefitTerm": 5,
              "defermentPeriod": 0,
              "calculatedDefermentPeriod": -2,
              "calculatedIrr": "-",
              "investmentEndYear": 2041,
              "incomeBenefitStartYear": 2068
            },
            "taxSavingsInfo": {
              "oldRegime": {
                "amount": 4629
              }
            },
            "planFeatureList": [
              {
                "code": "returnOfPremium",
                "name": "Return Of Premium",
                "active": true
              }
            ],
            "errorCategory": "SUCCESS",
            "defaultOption": false,
            "cisModuleEnabled": false,
            "investmentEndYear": 2041,
            "incomeBenefitStartYear": 2068,
            "specialBenefits": [
              "Terminal Illness benefit included",
              "Premium waiver on permanent disability"
            ]
          },
          {
            "quoteId": "9f49a203f8207031a79797aff6f8c798",
            "insurerQuoteId": "3218c981-3014-4e85-b15e-5366f721303b",
            "policyType": "TERM",
            "insurerCode": "MAXLIFELI",
            "internalProductCode": "maxlifeli-life-smarttermplanplus",
            "productCode": "P127",
            "productName": "Smart Term Plan Plus",
            "option": "Return of Premium",
            "optionCode": 2,
            "productUIN": "104N127V02",
            "tmPlanId": "1460",
            "policyTerm": 42,
            "paymentFrequency": 12,
            "premiumPaymentTerm": 42,
            "score": 0,
            "category": "term",
            "premium": 16908,
            "taxRate": 0,
            "premiumWithTax": 16908,
            "sumAssured": 10000000,
            "deathBenefitTotal": 10000000,
            "deathBenefitGuaranteed": 10000000,
            "taxSavingAmount": 3381,
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerMessage": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "planType": "Term",
            "showRider": false,
            "showRiderPremium": true,
            "edc": "2026-02-18"
          },
          {
            "quoteId": "099bef497ac57068127de20dd6096cd7",
            "insurerQuoteId": "534731cc-3ac7-464d-acf1-fbf365fe3aaa",
            "policyType": "TERM",
            "insurerCode": "MAXLIFELI",
            "internalProductCode": "maxlifeli-life-smarttermplanplus",
            "productCode": "P127",
            "productName": "Smart Term Plan Plus",
            "option": "Whole Life Cover",
            "optionCode": 4,
            "productUIN": "104N127V02",
            "tmPlanId": "1460",
            "policyTerm": 82,
            "paymentFrequency": 12,
            "premiumPaymentTerm": 15,
            "score": 34,
            "category": "term",
            "premium": 33654,
            "taxRate": 0,
            "premiumWithTax": 33654,
            "sumAssured": 10000000,
            "deathBenefitTotal": 10000000,
            "deathBenefitGuaranteed": 10000000,
            "taxSavingAmount": 6730,
            "status": "SUCCESS",
            "insurerStatus": "SUCCESS",
            "insurerMessage": "SUCCESS",
            "insurerBusinessFlowType": "QUOTES_REQUEST",
            "planType": "Term",
            "showRider": false,
            "showRiderPremium": true,
            "edc": "2026-02-18"
          }
        ],
        "taxSavingAmount": 1569,
        "status": "SUCCESS",
        "insurerStatus": "SUCCESS",
        "insurerMessage": "SUCCESS",
        "insurerBusinessFlowType": "QUOTES_REQUEST",
        "companyDetails": {
          "InsurerId": "MAXLIFELI",
          "InsurerName": "Axis Max Life",
          "Logo": "MAXLIFELI.jpg",
          "CompanyDetails": "Axis Max Life Insurance Limited, formerly known as Max Life Insurance Company Ltd., is a joint venture between Max Financial Services Limited (MFSL) and Axis Bank Limited.",
          "ClaimSettlementRate": {
            "OneMonth": "99.93",
            "OneToThreeMonths": "0.13",
            "ThreeMonthsPlus": "0.02"
          },
          "SpeedOfClaimSettlementSummary": "99.93",
          "InsurerCode": "MAXLIFELI",
          "ContactDetails": {
            "Telephone": "1860 120 5577",
            "Address": "419, Bhai Mohan Singh Nagar, Railmajra, Tehsil Balachaur, District Nawanshahr, Punjab -144 533."
          },
          "LifeCompanyDetails": {
            "claimRatios": {
              "Claims paid in < 3 months": "",
              "Claims paid < 1 year": "",
              "Claims settled ratio (2016-17)": "",
              "Claims paid < 6 months": "",
              "Claims paid > 3 months": "",
              "Claims rejected (2016-17)": "",
              "Claims paid in < 30 days": ""
            },
            "urlClaimForm": "",
            "solvencyRatio": "201%",
            "claimSettlementRatio": "99.50%",
            "inceptionYear": "2002",
            "numberOfLivesInsured": "1.19 Crs",
            "assetUnderManagement": "1,22,857 Crs",
            "branchesAcrossIndia": "269",
            "freelookPeriod": "Online- 15 days, Offline- 30 days"
          }
        },
        "payoutValues": {
          "LUMPSUM": [
            {
              "payoutType": "LUMPSUM",
              "basePayoutValue": 10000000
            }
          ],
          "LUMPSUM_PLUS_LEVEL_INCOME": [
            {
              "payoutType": "LUMPSUM",
              "basePayoutValue": 5000000
            },
            {
              "payoutType": "LEVEL_INCOME",
              "basePayoutValue": 83333.33333333333,
              "payoutFrequency": "MONTHLY",
              "benefitTerm": 5
            }
          ]
        },
        "totalPayoutValues": {
          "LUMPSUM": 10000000,
          "LUMPSUM_PLUS_LEVEL_INCOME": 10000000
        },
        "maturityBenefits": [
          "On survival till the end of the policy term. No benefit is payable. The policy will be terminated immediately & automatically on the maturity date."
        ],
        "payoutTerm": 5,
        "payoutFrequency": "MONTHLY",
        "planType": "Term",
        "riderList": [
          {
            "riderName": "Accidental Death and Dismemberment",
            "riderDesc": "This rider provides your loved ones with extra financial protection in case of an unexpected eventuality.",
            "riderShortDesc": "This rider provides your loved ones with extra financial protection.",
            "riderCode": "R77",
            "riderApiCode": "R77",
            "riderCategory": "Accidental Death Benefit",
            "riderSumAssured": 50000,
            "riderPolicyTerm": 42,
            "riderPremiumPaymentTerm": 42,
            "inBuilt": false,
            "isSelected": false,
            "isCoverAmountEditable": true,
            "isCoverAmountIncludedInBasePlan": false
          }
        ],
        "showRider": false,
        "showRiderPremium": true,
        "edc": "2026-02-18",
        "biProvider": "MAXLIFELI",
        "isBIPdfAvailable": true,
        "age": 18,
        "inputCoverAmount": 10000000,
        "planFeatureDetailsList": [],
        "bqpRedirectionEnabled": true,
        "paymentFirstJourneyCheck": false,
        "defermentPeriod": 0,
        "calculatedDefermentPeriod": -2,
        "incomeStartYear": -2,
        "calculatedIRR": "-",
        "cashFlowsPerYearArray": [],
        "resultCardsInfo": {
          "investmentAmount": 329490,
          "investmentPerFrequencyAmount": 7845,
          "paymentFrequency": 12,
          "entryAge": 18,
          "premiumPaymentTerm": 42,
          "policyTerm": 42,
          "maturityAge": 60,
          "deathCoverTillAge": 60,
          "deathBenefit": 10000000,
          "taxSavings": 1569,
          "returnOfPremium": false,
          "claimSettlementRatio": "99.50%",
          "incomePeriod": 5,
          "yearlyIncome": 1000000,
          "lumpsumAmount": 5000000,
          "benefitTerm": 5,
          "defermentPeriod": 0,
          "calculatedDefermentPeriod": -2,
          "calculatedIrr": "-",
          "investmentEndYear": 2068,
          "incomeBenefitStartYear": 2068
        },
        "taxSavingsInfo": {
          "oldRegime": {
            "amount": 1569
          }
        },
        "errorCategory": "SUCCESS",
        "defaultOption": true,
        "cisModuleEnabled": false,
        "investmentEndYear": 2068,
        "incomeBenefitStartYear": 2068
      }
    ]
  },
  "meta": {
    "status": "SUCCESS",
    "error": false,
    "traceId": "6997cc25a2396d59a5b88585cb8fe4d3",
    "timestamp": "2026-02-20T02:51:18.843366476"
  }
}
```
