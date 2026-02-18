# Life Quote FE Payload Spec (Payload-Only, Category + PlanType)

This document is only for FE request payloads for:

- `POST /api/minterprise/v2/products/life/quotes`

## 1) Fixed request wrapper (same for all combinations)

Use this exact envelope:

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "vertical": "LIFE",
      "policyType": "<TERM|TRADITIONAL|ULIP|PENSION>",
      "lifePremiumRequest": {
        "categories": ["<category>"],
        "planType": "<plan-type>",
        "...": "all existing life-service fields remain supported"
      }
    }
  }
}
```

Notes:
- `policyType` must be sent in `data.premiumRequest.policyType` (not at `data` level).
- Keep one payload contract only (no second payload format).

---

## 2) Common top-level fields used across categories

Minterprise fields supported in `data`:

- `referenceId` (optional)
  - Do not send for first quote request. Backend generates it.
  - Send on re-quote/retry if you want to continue same journey.
- `premiumRequest` (mandatory)

`quoteId` should not be sent in `POST /quotes`. Backend always generates a new `quoteId` per call.

These fields are supported in `data.premiumRequest` for all life categories:

- `requestType`, `isAsync`, `vertical`, `policyType`, `timestamp`
- `utmParams.utmSource`, `utmParams.utmMedium`, `utmParams.utmUrl`
- `initialReqFlag`, `customerName`, `userMobile`, `userEmail`
- `customerDetailId`, `createdByCustomer`, `pospUserName`
- optional journey fields as available from FE flow

Reference usage examples:

Initial request (new journey, no `referenceId`):

```json
{
  "data": {
    "premiumRequest": { "...": "..." }
  }
}
```

Re-quote request (existing journey, pass `referenceId`):

```json
{
  "data": {
    "referenceId": "AHUYGZWE7GO",
    "premiumRequest": { "...": "..." }
  }
}
```

---

## 3) Category + PlanType payloads (full examples)

All examples below are initial-call payloads (no `data.referenceId`).  
For re-quote, keep same payload and add `data.referenceId`.

### 3.1 `term` + `Term` (`policyType=TERM`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TERM",
      "timestamp": "2026-02-17T15:15:33.821Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.jkbank.topgun.turtle-feature.com/life-insurance/profile/retirement/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "ABID AHMAD SHAH",
      "userMobile": "7400400747",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["term"],
        "planType": "Term",
        "coverAmount": 10000000,
        "dateOfBirth": "1972-05-03T00:00:00+05:30",
        "entryAge": 53,
        "gender": "M",
        "incomeBracketCode": "7 Lakhs",
        "investmentTermCode": "medium",
        "isSmoking": false,
        "maritalStatus": "SINGLE",
        "maxIncome": 699999,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "profileType": "term",
        "tmOccupation": "salaried",
        "tmPincode": 713216,
        "tmQualification": "Graduate_Post_Graduate",
        "businessModel": "B2B",
        "businessChannel": "FIELD_SALES",
        "investmentRisk": "high"
      }
    }
  }
}
```

### 3.2 `guaranteed` + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:22:19.538Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["guaranteed"],
        "planType": "Non-participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "guaranteed-returns",
        "businessModel": "B2B",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES"
      }
    }
  }
}
```

### 3.3 `guaranteed` + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:22:19.538Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["guaranteed"],
        "planType": "Participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "guaranteed-returns",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.4 `guaranteed` + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:22:19.538Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["guaranteed"],
        "planType": "ULIP",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "guaranteed-returns",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.5 `ulip` + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:27:59.837Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["ulip"],
        "planType": "ULIP",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "ulip",
        "businessModel": "B2B",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "currency": "INR"
      }
    }
  }
}
```

### 3.6 `ulip` + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:27:59.837Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["ulip"],
        "planType": "Non-participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "ulip",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.7 `ulip` + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:27:59.837Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["ulip"],
        "planType": "Participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "ulip",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.8 `participating` + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:35:55.796Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["participating"],
        "planType": "Participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "3 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 300000,
        "minIncome": 300000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "participating-plans",
        "businessModel": "B2B",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES"
      }
    }
  }
}
```

### 3.9 `participating` + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:35:55.796Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["participating"],
        "planType": "Non-participating",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "3 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 300000,
        "minIncome": 300000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "participating-plans",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.10 `participating` + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:35:55.796Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["participating"],
        "planType": "ULIP",
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "WEALTH_CREATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "3 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 300000,
        "minIncome": 300000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "participating-plans",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "includeCategory": false,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.11 `child` + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:43:53.040Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["child"],
        "planType": "Non-participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "CHILD_EDUCATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "8 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 800000,
        "minIncome": 800000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "saving-for-child",
        "businessModel": "B2B",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2026-02-03T00:00:00+05:30",
        "gender": "M",
        "isNonSelfJourney": true,
        "propFullName": "kkk jkkk",
        "propGender": "M",
        "propDOB": "2008-02-05T00:00:00+05:30",
        "propAge": 18,
        "businessChannel": "FIELD_SALES"
      }
    }
  }
}
```

### 3.12 `child` + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:43:53.040Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["child"],
        "planType": "Participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "CHILD_EDUCATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "8 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 800000,
        "minIncome": 800000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "saving-for-child",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2026-02-03T00:00:00+05:30",
        "gender": "M",
        "isNonSelfJourney": true,
        "propFullName": "kkk jkkk",
        "propGender": "M",
        "propDOB": "2008-02-05T00:00:00+05:30",
        "propAge": 18,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.13 `child` + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:43:53.040Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["child"],
        "planType": "ULIP",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "CHILD_EDUCATION",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "8 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 800000,
        "minIncome": 800000,
        "paymentFrequency": 12,
        "policyTerm": 15,
        "premiumPaymentTerm": 10,
        "premium": 100000,
        "profileType": "saving-for-child",
        "payoutFrequency": "YEARLY",
        "defermentPeriod": 0,
        "businessModel": "B2B",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2026-02-03T00:00:00+05:30",
        "gender": "M",
        "isNonSelfJourney": true,
        "propFullName": "kkk jkkk",
        "propGender": "M",
        "propDOB": "2008-02-05T00:00:00+05:30",
        "propAge": 18,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.14 `retirement` single-life + `Pension` (`policyType=PENSION`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "PENSION",
      "timestamp": "2026-02-17T15:46:24.158Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Pension",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutIncome": 100000,
        "payoutFrequency": "YEARLY",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": false,
        "businessChannel": "FIELD_SALES"
      }
    }
  }
}
```

### 3.15 `retirement` single-life + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:46:24.158Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Non-participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": false,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.16 `retirement` single-life + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:46:24.158Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.17 `retirement` single-life + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:46:24.158Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "ULIP",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-05T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": false,
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": true,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.18 `retirement` joint-life + `Pension` (`policyType=PENSION`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "PENSION",
      "timestamp": "2026-02-17T15:49:30.958Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Pension",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutIncome": 100000,
        "payoutFrequency": "YEARLY",
        "tmPincode": 192125,
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-04T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": true,
        "jointLifeDetails": {
          "status": true,
          "dob": "2008-02-05T00:00:00+05:30"
        },
        "businessChannel": "FIELD_SALES"
      }
    }
  }
}
```

### 3.19 `retirement` joint-life + `Non-participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:49:30.958Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Non-participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-04T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": true,
        "jointLifeDetails": {
          "status": true,
          "dob": "2008-02-05T00:00:00+05:30"
        },
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": false,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.20 `retirement` joint-life + `Participating` (`policyType=TRADITIONAL`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "TRADITIONAL",
      "timestamp": "2026-02-17T15:49:30.958Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "Participating",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-04T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": true,
        "jointLifeDetails": {
          "status": true,
          "dob": "2008-02-05T00:00:00+05:30"
        },
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": false,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

### 3.21 `retirement` joint-life + `ULIP` (`policyType=ULIP`)

```json
{
  "data": {
    "premiumRequest": {
      "requestType": "INITIAL",
      "isAsync": true,
      "vertical": "LIFE",
      "policyType": "ULIP",
      "timestamp": "2026-02-17T15:49:30.958Z",
      "utmParams": {
        "utmSource": "(direct)",
        "utmMedium": "(none)",
        "utmUrl": "https://pro.turtlemintinsurance.com/life-insurance/profile/term/about-insured"
      },
      "initialReqFlag": true,
      "customerName": "Bjaja jaja",
      "customerDetailId": "001812758",
      "createdByCustomer": false,
      "pospUserName": "66bb2378ae016500016e5a06",
      "userEmail": "SHAHABID37@GMAIL.COM",
      "userMobile": "7400400747",
      "lifePremiumRequest": {
        "benifitCalculationRate": 8,
        "categories": ["retirement"],
        "planType": "ULIP",
        "includeCategory": true,
        "maritalStatus": "MARRIED_WITH_KIDS",
        "investmentGoals": "RETIREMENT",
        "riskAppetite": "MEDIUM",
        "existingInsuranceCoverAmount": 0,
        "incomeBracketCode": "5 Lakhs",
        "investmentRisk": "high",
        "maxIncome": 500000,
        "minIncome": 500000,
        "paymentFrequency": 12,
        "premiumPaymentTerm": 5,
        "policyTerm": 15,
        "premium": 200000,
        "profileType": "retirement",
        "businessModel": "B2B",
        "plansByPension": false,
        "defermentPeriod": 10,
        "payoutFrequency": "YEARLY",
        "tmPincode": "192125",
        "tmCity": "Srinagar",
        "tmState": "JAMMU & KASHMIR",
        "insuredFullName": "Bjaja jaja",
        "dateOfBirth": "2008-02-04T00:00:00+05:30",
        "gender": "M",
        "entryAge": 18,
        "isNonSelfJourney": false,
        "isJointLife": true,
        "jointLifeDetails": {
          "status": true,
          "dob": "2008-02-05T00:00:00+05:30"
        },
        "businessChannel": "FIELD_SALES",
        "quotesMailSent": false,
        "userRiskConsentAccepted": false,
        "currency": "INR"
      }
    }
  }
}
```

---

## 4) FE rule summary

Keep the same API and same request wrapper always. Only values change by combination:

- `data.premiumRequest.policyType`
- `data.premiumRequest.lifePremiumRequest.categories[0]`
- `data.premiumRequest.lifePremiumRequest.planType`
- category-specific blocks:
  - `jointLifeDetails` for retirement joint-life
  - proposer fields for child non-self journey
  - payout/deferment fields where applicable
