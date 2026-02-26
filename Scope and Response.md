# IH Scope & Response Architecture — Key Differences

---

## 1. Non-Life vs Life IH Payload

### Non-Life (Motor/Health) — Simple Passthrough

```json
{
  "mappingQuery": { "integrationProvider": "HDFC_ERGO", "tmRequestType": "MOTOR_PREMIUM", "vertical": "MOTOR" },
  "businessFunctionInput": {
    "premiumRequest": { /* raw request — passed as-is, no field mapping */ }
  }
}
```

### Life — Multi-Source Assembled Scope

```json
{
  "mappingQuery": { "integrationProvider": "BAJAJLI", "tmRequestType": "TERM_BAJAJ_PREMIUM", "vertical": "LIFE" },
  "scope": {
    "request":       { /* 25+ individually mapped IRM fields */ },
    "master":        { /* NM row from master-service-v2 API call */ },
    "constants":     { /* from lifeInsurerProviderMeta DB */ },
    "insurerConfig": { /* from brokerConfig DB */ },
    "partnerDetails":{ /* from partner API */ }
  }
}
```

| Aspect | Non-Life | Life |
|---|---|---|
| Request | Raw passthrough | 25+ fields mapped per insurer |
| Master data | Not needed | master-service-v2 API (NM row) |
| Auth/Constants | Not needed | `lifeInsurerProviderMeta` DB |
| Insurer config | Not needed | `brokerConfig` DB |
| Routing | Single `tmRequestType` | Category-aware: `TERM_BAJAJ_PREMIUM` vs `NON_TERM_BAJAJ_PREMIUM` |

---

## 2. scope.request Differs Per Insurer

Every insurer has its own `createIntegrationRequest` method. All share the same **12 core fields** (age, PT, PPT, DOB, gender, smoker, productCode, optionCode, sumAssured/premium, paymentFrequency, currentDate, productCodeOptionCode) but differ in extra fields and formatting:

| Insurer | Date Format | Timezone | Unique Fields | Riders | Proposer Details |
|---|---|---|---|---|---|
| **Generic** | `yyyy-MM-dd` | UTC | investmentReturns | ❌ | ❌ |
| **BAJAJ** | `dd/MM/yyyy` | IST | firstName, productFullName, premium always `"0"`, whole life PT=`99` | ✅ coverDetails→riders | ❌ |
| **HDFC** | `dd/MM/yyyy` | IST | occupation, customerName, nvestFlag adjusts PT/PPT | ✅ | ✅ (only if isNonSelfJourney) |
| **ICICI** | `yyyy-MM-dd` | IST | category, suitability (maritalStatus, riskAppetite, investmentGoals), isNonSelfJourney | ✅ | ✅ always |
| **MaxLife** | `yyyy-MM-dd` | IST | annualIncome, pptFormula, nvestFlag adjusts PPT, investmentGoals, maritalStatus | ✅ | ✅ (non-term only) |
| **Aegon** | `yyyy-MM-dd` | UTC | policyPaymentType, special LP_60 logic (age+PPT=60 → override PPT) | ❌ | ❌ |
| **Federal** | `yyyy-MM-dd` | UTC | payoutType (term), inputBonus | ❌ | ❌ |
| **Mashreq** | Custom | — | UAE-specific fields | ✅ | ❌ |
| **BAJAJ NonTerm** | `dd/MM/yyyy` | IST | Separate class from BAJAJ term, different rider mapping | ✅ | ❌ |
| **IDFC BAJAJ** | Custom | — | IDFC-specific encryption | ❌ | ❌ |

**Key difference categories:**
- **Date format & timezone** — `dd/MM/yyyy` IST vs `yyyy-MM-dd` UTC
- **Extra fields** — proposer details, suitability, occupation, policyPaymentType
- **Rider handling** — some map riders, some don't; different mapping methods (`mapRider` vs `mapRiderv2`)
- **Special PT/PPT logic** — nvestFlag, LP_60, whole life = `99` vs `99-age`

---

## 3. Why Insurer-Specific Response Classes

Each insurer API returns a **different JSON structure** — different field names, wrappers, and status indicators.

### Premium Field Differences

| Insurer | Base Premium | Tax | Death Benefit |
|---|---|---|---|
| BAJAJ | `basePremium` | `serviceTax` | Not in main response |
| HDFC | `outputjsillustration.db[].Premium` | Computed | `db[].deathbenefitguaranteed[]` (year-wise array) |
| ICICI | `basePremium` | `serviceTax` | `illustrationDetails[].deathBenefit` |
| Tata | `premium` | `gst` | `coverDetails.totalDeathBenefit` |

### Error Detection Differences

| Insurer | How It Detects Error |
|---|---|
| BAJAJ | `basePremium == null` |
| HDFC | `status != "SUCCESS"` |
| MaxLife | `validationResults[]` has invalid entries |
| Aegon | `result == null` |
| Tata | `responseStatus != "S"` |

### Term vs Non-Term — Same Insurer, Different Response

HDFC and BAJAJ each have **two separate provider classes** because term and non-term responses have fundamentally different fields:
- **Term** → death benefit, cover amount
- **Non-term** → maturity benefit, survival benefit (at 4%/8%), fund values, income data

### Why Generic Won't Work

A single generic handler would need 6+ if-else branches for each of: wrapper unwrapping, field name mapping, status checking, benefit extraction, and category-specific logic. The **Strategy Pattern** (one class per insurer) keeps each insurer's logic isolated and maintainable.

Life-service has **14 ResultsProvider classes** and **11 insurer-specific response POJOs** for this reason.
