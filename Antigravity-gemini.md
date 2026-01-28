# API Documentation: `/api/tm-life/v0/premiums/request/validate`

## Overview
This API route validates a life insurance premium request against various criteria (product rules, age, income, etc.) and returns a list of eligible products and plans. It is the core validation engine for the Life Insurance flow.

**Method:** `POST`
**Controller:** `LifeResultsAPI`
**Method Name:** `getRequest`

---

## Execution Flow

The request execution path flows through the following key components:

1.  **Entry Point**: `LifeResultsAPI.getRequest(PremiumRequest, ServerHttpRequest)`
    *   Determines `BrokerConfig`.
    *   Sets default currency (INR) and result URL.
    *   Calls `recommondation.findRecommendationProducts` (optional recommendation logic).
    *   **Core Logic**: Delegates to `liService.getValidationService().validatePremiumRequest()`.

2.  **Service Layer**: `LifeValidationServiceImpl.validatePremiumRequest(PremiumRequest, BrokerConfig)`
    *   Extracts `categories` from the request.
    *   Retrieves the appropriate `ILifeRequestValidator` (e.g., `AbstractLifeRequestValidator` implementation).
    *   Calls `lifeRequestValidator.validatePremiumRequest()`.
    *   **Caching**: Caches the validation results using `LifeCacheService`.

3.  **Validator Logic**: `AbstractLifeRequestValidator.validatePremiumRequest`
    *   **Strategy Selection**: Checks `LifeABTestingConfig` to decide between "New Implementation" (Studio/External) or "Old Implementation" (DB/Local).
    
    **New Implementation Flow (`useNewImplementation`)**:
    1.  **Fetch Products**: `LifeProductMasterDao.fetchEnabledProductMaster` (queries `lifeProductMaster`).
    2.  **AB Test Filtering**: `abProductMasterHelper.productMasterFromStudio`.
    3.  **Defaulting**: Sets default Payout Types/Frequency if missing.
    4.  **Post Filters**: Filters by payout mode (`postFilterProductMaster`).
    5.  **Product Visibility**: Filters based on broker rules (`applyProductVisibilityFilter`).
    6.  **Plan Features**: Filters based on requested features (`planFeatureService.filterProductsForFeatures`).
    7.  **Final Validation**: Delegates to `LifeRequestValidatorHelper.applyLifeValidator`.

    **Old Implementation Flow (`useOldImplementation`)**:
    1.  **Fetch Products**: Fetches `LifeProductMaster` from DB.
    2.  **Insurer Config**: Checks `InsurerConfigurationService.getEnabled`.
    3.  **Filters**: Applies AB Testing, Defaults, Post Filters, Visibility, and Feature filters.
    4.  **Fetch Eligible Rows**: Calls `fetchEligibleProductRows` which queries `lifeRequestValidator` and `lifeRequestValidatorNM` collections directly.

4.  **Helper & External Calls**: `LifeRequestValidatorHelper.applyLifeValidator`
    *   **Nearest Macth**: Delegates to `NearestMatchValidators.getNearestMatchRange`.
        *   `NearestMatchValidatorsImpl` calls **External Service** (Product Management Service / Master Service V2).
    *   **Rider Validation**: Calls `LifeRiderValidator.fetchEligibleRiderRows`.
        *   Fetches riders from `lifeRiderMaster`.
        *   Calls **Rule Engine** API (`RULES_RIDER_GET_SLABS_API`, `RULES_RIDER_VALIDATE_API`) for slab validation.
    *   **Offer Validation**: Calls `LifeOfferValidator.fetchEligibleOfferRows` (queries `lifeOfferMaster`).

---

## Functions Involved

| Class | Method | Purpose |
| :--- | :--- | :--- |
| `LifeResultsAPI` | `getRequest` | API Entry point. |
| `LifeValidationServiceImpl` | `validatePremiumRequest` | Service orchestrator, handles caching. |
| `AbstractLifeRequestValidator` | `validatePremiumRequest` | Main validation logic (New/Old strategy selection). |
| `AbstractLifeRequestValidator` | `useNewImplementation` | Validation logic using enabled product masters from Studio. |
| `AbstractLifeRequestValidator` | `useOldImplementation` | Legacy validation logic querying local collections. |
| `AbstractLifeRequestValidator` | `fetchEligibleProductRows` | Queries `lifeRequestValidator` collections (Old Impl). |
| `LifeRequestValidatorHelper` | `applyLifeValidator` | Orchestrates Nearest Match, Riders, and Offers validation. |
| `NearestMatchValidatorsImpl` | `getNearestMatchRange` | Computes nearest match using Product Management Service. |
| `ProductManagementServiceImpl` | `fetchLifeValidator` | Calls external Master Service V2. |
| `LifeRiderValidator` | `fetchEligibleRiderRows` | Validates riders and calls Rule Engine. |
| `LifeOfferValidator` | `fetchEligibleOfferRows` | Validates offers. |

---

## Database Collections

The following MongoDB collections are accessed during the execution of this route:

### Core Data & Validation
*   `lifeProductMaster`: Product definitions and configurations.
*   `lifeRequestValidator`: Legacy validation rules (Policy Term, PPT calculations).
*   `lifeRequestValidatorNM`: Nearest Match validation rules (min/max age, maturity, etc.).

### Supporting Masters
*   `lifeRiderMaster`: Rider definitions.
*   `lifeOfferMaster`: Offer definitions.
*   `lifeAgeIncomeMultiplierValidator`: Validates sum assured multipliers based on age/income.

### Metadata & Configuration
*   `lifeInsurerProviderMeta`: Insurer metadata.
*   `lifeCheckoutPaymentMeta`: Payment gateway configurations.
*   `lifeQualificationMaster`: Qualification codes/names.
*   `lifeOccupationMaster`: Occupation codes/names.
*   `lifePincodeMaster`: Pincode serviceability.
*   `lifeBenefitIllustration`: BI data (if accessed during flow).
*   `lifeInsurerRedirectMapping`: Redirection rules.

---

## External Services
*   **Product Management Service (Master Service V2)**: Fetched via `ProductManagementServiceImpl`.
*   **Rule Engine**: Used for Rider Slabs and Validation via `LifeRiderValidator`.
