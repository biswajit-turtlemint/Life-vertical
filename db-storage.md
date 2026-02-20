# Life Quote DB Storage Spec

This is the current DB storage contract for life in minterprise.
For life rows:
- `productDetails` is not stored.
- `premiumResponse`, `validationMap`, `pendingKeyList`, `errorMessage` are stored only once at top level.

## 1) Collections
- Request: `sachetPremiumRequest`
- Request history: `sachetPremiumRequestHistory`
- Result: `sachetPremiumResponse`

Source constant:
- `/Users/biswajitrout/companyProjects/minterprise/library/src/main/java/com/library/constant/DBColConstants.java`

## 2) Request document (`sachetPremiumRequest`)

One request row per `referenceId`.
`quoteId` is intentionally `null` for life request rows.

Stored document (full concrete example):

```json
{
  "_id": "6998b8c8df2d8a2f1b9f1001",
  "referenceId": "AHUTP1QQBR8",
  "quoteId": null,
  "productCode": "life",
  "tenant": "turtlemint",
  "broker": "turtlemint",
  "partnerId": "66bb2378ae016500016e5a06",
  "leadName": "ABID AHMAD SHAH",
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
  },
  "createdAt": "2026-02-20T10:15:42.221Z",
  "updatedAt": "2026-02-20T10:15:42.221Z"
}
```

Mapping used:
- `productCode` from `data.premiumRequest.vertical` (normalized lower-case)
- `leadName` from `data.premiumRequest.personalDetails.customerName`
- `partnerId` from `data.premiumRequest.pospUserName`
- `riskInsured` from `data.premiumRequest.riskInsured`
- `premiumRequest` is exact FE payload from `data.premiumRequest`

## 3) Result document (`sachetPremiumResponse`)

One request (`referenceId`) can have multiple result rows (one per plan/product key).

Stored document (full concrete example):

```json
{
  "_id": "6998b8c8df2d8a2f1b9f2001",
  "referenceId": "AHUTP1QQBR8",
  "quoteId": "dc28c8a00890741fca957b2f92783599",
  "productCode": "life",
  "provider": "MAXLIFELI",
  "planCode": "P127",
  "planName": "Smart Term Plan Plus",
  "tenant": "turtlemint",
  "broker": "turtlemint",
  "partnerId": "66bb2378ae016500016e5a06",
  "leadName": "ABID AHMAD SHAH",
  "status": "success",
  "premium": 7845,
  "netPremium": 7845,
  "tax": 0,
  "totalPremium": 7845,
  "currency": "INR",
  "policyTerm": 42,
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
  "premiumResponse": {
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
        "riderPolicyTerm": 42,
        "riderPremiumPaymentTerm": 42,
        "inBuilt": false,
        "isSelected": false,
        "isCoverAmountEditable": true,
        "isCoverAmountIncludedInBasePlan": false
      },
      {
        "riderName": "Critical Illness and Disability-Total & Permanent Disability Variant (TPD)",
        "riderDesc": "This rider provides additional protection against listed critical illnesses.",
        "riderShortDesc": "Additional protection against listed critical illnesses.",
        "riderCode": "R82",
        "riderApiCode": "R82",
        "riderCategory": "Critical Illness",
        "riderSumAssured": 300000,
        "riderPolicyTerm": 20,
        "riderPremiumPaymentTerm": 20,
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
    "planFeatureList": [
      {
        "code": "returnOfPremium",
        "name": "Return Of Premium",
        "active": true
      }
    ],
    "errorCategory": "SUCCESS",
    "defaultOption": true,
    "cisModuleEnabled": false,
    "investmentEndYear": 2068,
    "incomeBenefitStartYear": 2068,
    "specialBenefits": [
      "Terminal Illness benefit included",
      "Premium waiver on permanent disability"
    ],
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
          "Life cover continues till end of policy term"
        ],
        "payoutTerm": 5,
        "payoutFrequency": "MONTHLY",
        "planType": "Term",
        "riderList": [
          {
            "riderCode": "R77",
            "riderName": "Accidental Death and Dismemberment",
            "riderCategory": "Accidental Death Benefit",
            "riderSumAssured": 50000,
            "riderPolicyTerm": 15,
            "riderPremiumPaymentTerm": 15,
            "isSelected": false
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
          "returnOfPremium": false
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
    ]
  },
  "validationMap": {
    "P127": {
      "valid": true,
      "key": "P127",
      "insurerCode": "MAXLIFELI"
    }
  },
  "pendingKeyList": [
    "P90",
    "P108",
    "P118"
  ],
  "errorMessage": "",
  "createdAt": "2026-02-20T10:15:44.921Z",
  "updatedAt": "2026-02-20T10:15:44.921Z"
}
```

Important:
- `productDetails` is not stored for life rows.
- No duplicated storage for `premiumResponse`/`validationMap`/`pendingKeyList`/`errorMessage`.
- `premiumResult` is an in-memory processing object in aggregator flow, not a persisted top-level DB field.
- `premiumResponse` is the persisted quote payload map for that result row.
- `responseOptions[*]` stores full option maps from life flow (not just minimal keys).
- `premiumResponse` and `responseOptions[*]` are persisted in normalized ID format:
  - `quoteId` (internal result ID used for poll/retrieval)
  - `insurerQuoteId` (present only when insurer sends quote ID and it differs)
- `resultId` is not persisted in stored `premiumResponse`.

API mapping note:
- Stored `premiumResponse.quoteId` is returned to FE as `quoteId`.
- Stored `premiumResponse.insurerQuoteId` is returned to FE as `insurerQuoteId` (when present).
- Stored `premiumResponse` keeps `companyDetails` at parent quote level; option-level `companyDetails` is removed.

## 4) Request/result linkage
- One request row in `sachetPremiumRequest` can map to multiple rows in `sachetPremiumResponse`.
- Link key: `referenceId`.
- Result upsert identity: `referenceId + quoteId + provider + planCode`.

## 5) Poll lookup keys
`GET /products/life/quotes/poll?referenceId=<REF>&resultIds=<...>` supports:
- `_id`
- `quoteId`
- `planCode`

`pendingKeyList` values are plan keys (`planCode`/product key), so FE can pass pending keys directly in `resultIds`.
