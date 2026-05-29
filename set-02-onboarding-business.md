# Set 2 — Onboarding (Business)

9 endpoints under `BusinessAccountController`. Three flows in play:
- **Legacy single-shot:** `POST /business/accounts` (everything in one payload)
- **New 4-step flow:** `POST /business/prospects` → `POST /business/prospects/{id}/owners` → `POST /business/prospects/{id}/account` → `POST /business/{id}/additional-info`
- **Updates** mirror the steps via PUT

Downstream surfaces touched: **glide-prospects** (`khonsu_business_prospect`, `khonsu_bu_prospect_owners`), **glide-client-account** (`osiris_business_account`), **glide-client** (`hathor_business`, `hathor_client_address`), **glide-business** (`cronus_business_additional_info`, `cronus_business_profile`).

Mythology mapping confirmed:
- `khonsu_*` → glide-prospects (already established)
- `osiris_*` → glide-client-account
- `hathor_*` → glide-client
- `cronus_*` → **glide-business** (NEW for this set)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | `BusinessInfo.company` (legacy `CreateBusinessAccountRequest`) | Gateway @Size(max=40), `khonsu_business_prospect.compnay`=varchar(40) **(typo in column name)**, `hathor_business.company`=**varchar(100) NOT NULL** | ❌ DB column is `compnay` not `company`. Also two different lengths (40 vs 100). |
| 2 | `BusinessInfo.companyWebsite` | Gateway @Size(max=50) optional, `khonsu_business_prospect.compnay_website`=**varchar(50) NOT NULL** (typo) | ❌ **Breaking risk** — DB column is NOT NULL but gateway treats it as optional. A request without `companyWebsite` will fail at INSERT. |
| 3 | `Address.addressLine1` (legacy + prospect-update flow) | Gateway @Size(max=80), `khonsu_business_prospect.address_one`=**varchar(30)** | ❌ **Truncation/error**: gateway accepts 80, DB only 30. Inconsistent with `khonsu_prospect.address_one`=80 (individual) and `khonsu_bu_prospect_owners.address_one`=80. |
| 4 | `PersonalInfo.phoneNumber` & `BusinessInfo.phoneNumber` (legacy) | Gateway @Size(max=20), `BusinessProspect` DTO @Pattern(`(^$\|[0-9]{10})`) **exactly 10 digits**, `hathor_business.phone`=**varchar(10)** | ❌ Downstream is stricter than gateway; DB column is varchar(10) but gateway accepts 20-char input. International numbers exceeding 10 digits will fail. |
| 5 | `BusinessInfo.companyWebsite` & `social_media` | Gateway @Size(50/100), `hathor_business.company_website`=varchar(100), `social_media`=varchar(100) vs `khonsu_business_prospect`=varchar(50/100) | ❌ Same field, two lengths across prospect vs business tables. |
| 6 | `BusinessInfo.businessIdentificationType` | Gateway @Size(max=9) @Pattern(TIN/OTHER/EIN), `cronus_business_additional_info.business_identification_type`=varchar(10) | ✅ OK, but no validation in downstream `BusinessAdditionalInfo`. |
| 7 | `CreateAccountFromProspectRequest` | DTO has ONE field `accountName` — **no `@Size`, no `@NotBlank`** | ❌ Unbounded. Account name flows to `osiris_business_account.account_id` candidate or `osiris_client_account.account_name`=varchar(60). |
| 8 | `CreateSpendingAccountRequest` (gateway) | Has `@NotBlank` on `name`, no `@Size`. Downstream `glide-client-account/CreateSpendingAccountRequest` is identical (only `name` field, no @Size) | ⚠ No length cap anywhere on the spending-account `name`. |
| 9 | `BusinessAdditionalInfo` (downstream DTO) | **Zero validation** on any field | ⚠ Gateway has @Size on most fields; downstream relies on it. |
| 10 | `BusinessAdditionalInfo` un-sized fields | `typeOfBusiness`, `destinationOfFunds`, `reasonRelationWithSuvi`, `businessGenerationMethod`, `approximateMonthlyTransaction`, `approximateDepositFistMonth`, `sourceOfFunds`, `phoneNumber` — gateway has **no @Size** but DB caps at 20–60 chars | ❌ Multiple silent truncation risks. |
| 11 | `cronus_business_additional_info` drift | DB has `industry` (varchar 60), `customer_type` (varchar 60) — **not in JPA entity** | ⚠ Schema drift: columns exist in DB but the entity doesn't map them. |
| 12 | `Owner.identificationType/Number` (legacy + new flows) | Gateway @Size(max=40/60) @Pattern, downstream `BuProspectOwner` **no validation**, `khonsu_bu_prospect_owners` cols (40/60) | ⚠ Aligned by accident — relies on gateway enforcement. |
| 13 | `BusinessProspect.firstName/lastName` Pattern | Downstream `BusinessProspect` has `@Pattern("(^$\|[0-9]{10})")` for phoneNumber, but firstName/lastName are `@Size(min=2,max=20)`. `khonsu_business_prospect.first_name`=varchar(20) | ✅ aligned to gateway. |
| 14 | `khonsu_business_prospect.country_code` | varchar(20) — same outlier as the individual prospect table | ❌ Should be varchar(3). |
| 15 | `BusinessInfo.companyMission` | Gateway @Size(max=255), `cronus_business_additional_info.company_mission`=varchar(255) | ✅ |
| 16 | `AddBusinessAdditionalInfoRequest.phoneNumber` | No @Size on gateway, `cronus_business_additional_info.phone_number`=varchar(20) | ⚠ no @Pattern either. |
| 17 | `BusinessInfo.dateEstablished` | Gateway LocalDate @JsonFormat("yyyy-MM-dd"), no @Past | ⚠ Should reject future-dated incorporations. |
| 18 | `osiris_business_account.account_id` | varchar(**255**) — `osiris_client_account.account_id` is varchar(20) | ❌ Same logical field, two lengths across business vs individual account tables. |
| 19 | `osiris_business_account.account_type` | varchar(**255**) — `osiris_client_account.account_type` is varchar(30) | ❌ Same as above. |
| 20 | `UpdateBusinessOwnerRequest` extends legacy `Owner` plus required `ownerType` | Same validation as `Owner` (above). Pattern restricts to KeyPerson\|Owners. | ✅ |
| 21 | **`cronus_business_users.phone` = varchar(15)** but gateway `keyPerson.phoneNumber` is `@Size(max=20)`. Written **async via Kafka** when `osiris_business_account` is persisted (`ClientProfileCreationEvent` → glide-business `KafkaEventConsumer` → `userService.createUserOrAssignGroup`). | ❌ **Production gap** — partner with 16–20 char phone passes every sync validation layer, then async INSERT fails. The business account exists but the keyPerson user record is missing. |
| 22 | **`cronus_business_users.first_name`/`last_name`/`email` = varchar(255)** — async-written from keyPerson data | ❌ Adds new outlier widths to the messy first_name landscape (now 20/32/50/150/255) and email (now 100/255). |
| 23 | **`hathor_business_client.phone` = varchar(10)** — async-written when the keyPerson is linked as a business-client. Adds a third phone width for the same logical field (10 in hathor_business / 15 in cronus_business_users / 20 at gateway). `first_name`/`last_name` are varchar(32) NOT NULL — gateway @Size(20) is tighter so safe, but the NOT NULL means missing keyPerson name will fail. | ❌ |
| 24 | `osiris_business_lexnex_response.business_lexnex_check_status` = varchar(20) vs `.owner_lexnex_check_status` = varchar(30) — same logical KYC-status field at two widths on the same table | ⚠ Server-controlled enum drift |
| 25 | **Async / Kafka-driven persistence** — Set 2 entities emit Kafka events that produce side-effect writes to additional tables (see "Async / Kafka-driven persistence" section after §9 for the full table list and Kafka topic trace). | ❌ Two new validation gaps confirmed (#21, #22, #23) |

---

## 1. `POST /admin/business/accounts` | `POST /external/business/accounts` — Legacy single-shot business account create

- **Controller:** `BusinessAccountController.createBusinessAccount`
- **Request DTO (gateway):** `CreateBusinessAccountRequest` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/account/CreateBusinessAccountRequest.java))
- **Service flow:** `CompanyUserAccountService.createBusinessAccount` → `ProspectService.createBusinessProspect` (→ `CompanyMapper.mapToBusinessProspect` → `ProspectManager.createProspect(BusinessProspect)`) → `ProspectService.createBusinessOwners` (→ `CompanyMapper.mapToBuProspectOwner` → `ProspectManager.createBuProspectOwners(BuProspectOwner)`) → `ClientAccountService.createBusinessAccount` (→ `ClientAccountManager.createBusinessAccount(prospectId, BusinessAcProp)`) → `BusinessService.addAdditionalBusinessInfo` (→ `AccountMapper.mapToBusinessAdditionalInfo` → `BusinessManager.createAdditionalInfo(businessId, BusinessAdditionalInfo)`)
- **Downstream DTOs:** `BusinessProspect`, `BuProspectOwner` (glide-prospects), `BusinessAcProp` (glide-client-account), `BusinessAdditionalInfo` (glide-business)
- **Persisted to:** `khonsu_business_prospect`, `khonsu_bu_prospect_owners`, `osiris_business_account`, `hathor_business` (+ `hathor_client_address`), `cronus_business_additional_info`

### PersonalInfo

| # | Field | Type | Gateway DTO | Downstream (`BusinessProspect`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `firstName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `khonsu_business_prospect.first_name` varchar(20) NOT NULL | ✅ | Aligned to 20. |
| 2 | `lastName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `khonsu_business_prospect.last_name` varchar(20) NOT NULL | ✅ | |
| 3 | `email` | String | @Size(max=100) (no @NotEmpty, no @Email) | @Email (no @Size) | `khonsu_business_prospect.email` varchar(100) · `hathor_business.email` varchar(100) | ⚠ | Gateway missing @Email. |
| 4 | `phoneNumber` | String | @Size(max=20) | @Pattern(`(^$\|[0-9]{10})`) — **exactly 10 digits or empty** | `khonsu_business_prospect.phone_number` varchar(20) · `hathor_business.phone` **varchar(10)** | ❌ | Three different rules. Inconsistent with individual (`Prospect` allows 4–15 digits). |
| 5 | `phoneCountryCode` | String | @Pattern("\\d{1,4}") | @Pattern("\\d{1,4}") | `khonsu_business_prospect.phone_country_code` varchar(6) · `hathor_business.phone_country_code` varchar(6) | ✅ | |

### Address

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 6 | `addressLine1` | String | @NotEmpty @Size(max=80) | mapped → `ProspectAddress.address1` | `khonsu_business_prospect.address_one` **varchar(30)** · `hathor_client_address.address_one` varchar(80) | ❌ | **Truncation risk** — DB column on prospect is 30, gateway allows 80. |
| 7 | `addressLine2` | String | @Size(max=40) | mapped → `address2` | `khonsu_business_prospect.address_two` varchar(40) · `hathor_client_address.address_two` varchar(80) | ⚠ | 40 vs 80 — inconsistent across tables, but no truncation risk (gateway is the tighter bound). |
| 8 | `zipCode` | String | @NotEmpty @Size(max=10) | mapped → `zipCode` | `khonsu_business_prospect.postal_code` varchar(10) · `hathor_client_address.postal_code` varchar(10) | ✅ | |
| 9 | `city` | String | @NotEmpty @Size(max=128) | mapped → `city` | `khonsu_business_prospect.city` varchar(128) · `hathor_client_address.city` varchar(128) | ✅ | |
| 10 | `state` | String | @NotEmpty @Size(max=64) | mapped → `state` | `khonsu_business_prospect.state` varchar(64) · `hathor_client_address.state` varchar(64) | ✅ | |
| 11 | `countryCode` | String | @NotEmpty (no @Size!) | mapped → `countryCode` (BusinessProspect has @Size(min=2,max=3)) | `khonsu_business_prospect.country_code` **varchar(20)** · `hathor_client_address.country_code` varchar(3) | ❌ | Gateway has no @Size; DB col is 20 (prospect) vs 3 (client). Inconsistent with `CreateAccountRequest` (individual) where gateway is @Size(max=2). |

### BusinessInfo

| # | Field | Type | Gateway DTO | Downstream (`BusinessProspect` / `BusinessAdditionalInfo`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 12 | `company` | String | @Size(max=40) | (no validation) | `khonsu_business_prospect.compnay` **varchar(40) NOT NULL** (col-name typo) · `hathor_business.company` varchar(100) NOT NULL | ❌ | Two lengths (40 vs 100) and a typo (`compnay`). |
| 13 | `businessEin` | String | @Size(max=30) | (no validation) | `cronus_business_additional_info.business_ein` varchar(30) | ✅ | No @Pattern for EIN format. |
| 14 | `phoneNumber` | String | (no @Size) | (no validation) | `cronus_business_additional_info.phone_number` varchar(20) | ⚠ | No @Size or @Pattern at gateway. |
| 15 | `phoneCountryCode` | String | @Pattern("\\d{1,4}") | — | (not persisted in business tables) | ⚠ | |
| 16 | `businessIdentificationType` | String | @Size(max=9) @Pattern("^(TIN\|OTHER\|EIN)$") | (no validation) | `cronus_business_additional_info.business_identification_type` varchar(10) | ✅ | |
| 17 | `stateOfIncorporation` | String | @Size(max=15) | (no validation) | `cronus_business_additional_info.state_of_incorporation` varchar(15) | ✅ | |
| 18 | `primaryIndustry` | String | @Size(max=80) | (no validation) | `cronus_business_additional_info.primary_industry` varchar(80) | ✅ | |
| 19 | `companyType` | String | @Size(max=150) | (no validation) | `cronus_business_additional_info.company_type` varchar(150) | ✅ | |
| 20 | `companyWebsite` | String | @Size(max=50) | (no validation) | `khonsu_business_prospect.compnay_website` **varchar(50) NOT NULL** (typo) · `hathor_business.company_website` varchar(100) | ❌ | NOT NULL in DB but optional at gateway → INSERT will fail without a value. Plus typo and 50/100 mismatch. |
| 21 | `socialMedia` | String | @Size(max=100) | (no validation) | `khonsu_business_prospect.social_media` varchar(100) · `hathor_business.social_media` varchar(100) | ✅ | |
| 22 | `annualRevenue` | String | @Size(max=40) | (no validation) | `cronus_business_additional_info.annual_revenue` varchar(40) | ✅ | |
| 23 | `subIndustry` | String | @Size(max=100) | (no validation) | `cronus_business_additional_info.sub_industry` varchar(100) | ✅ | |
| 24 | `transactionsMonth` | String | (no @Size) | (no validation) | — (not directly persisted, maps to `approximateMonthlyTransaction`?) | ⚠ | Field name doesn't match downstream. |
| 25 | `depositFirstMonth` | String | (no @Size) | (no validation) | — | ⚠ | Same. |
| 26 | `companyMission` | String | @Size(max=255) | (no validation) | `cronus_business_additional_info.company_mission` varchar(255) | ✅ | |
| 27 | `sourceOfFunds` | String | (no @Size) | (no validation) | `cronus_business_additional_info.source_of_funds` **varchar(60)** | ❌ | No gateway length; DB caps at 60. |
| 28 | `customerSupplierBased` | String | @Size(max=50) | (no validation) | `cronus_business_additional_info.customer_supplier_based` varchar(50) | ✅ | |
| 29 | `dateEstablished` | LocalDate | @JsonFormat("yyyy-MM-dd") | (no validation) | `cronus_business_additional_info.date_established` date | ⚠ | Missing `@Past`. |
| 30 | `no25PercentOwnership` | boolean | (primitive) | (no validation) | `khonsu_business_prospect.no_25pct_ownership` bool | ✅ | |
| 31 | `entityType` | String | @Size(max=150) | (no validation) | `khonsu_business_prospect.entity_type` varchar(150) | ✅ | |
| 32 | `companyProfile` | String | @Size(max=100) | (no validation) | `khonsu_business_prospect.company_profile` **text** (unbounded) | ⚠ | Gateway caps 100; DB unlimited. |
| 33 | `agreeToTermsAndConditions` | boolean | (primitive, default false) | (no validation) | `cronus_business_profile.agree_to_terms_and_conditions` bool | ✅ | |

### FinancialInfo

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 34 | `annualRevenue` | String | (no @Size) | — | (duplicate of BusinessInfo.annualRevenue) | ⚠ | Same field appears in two nested DTOs. |
| 35 | `approximateMonthlyTransaction` | String | (no @Size) | (no validation) | `cronus_business_additional_info.approximate_monthly_transaction` varchar(40) | ⚠ | No gateway length cap. |
| 36 | `approximateDepositFistMonth` | String | (no @Size) | (no validation) | `cronus_business_additional_info.approximate_deposit_fist_month` varchar(40) | ⚠ | Same. Note misspelled `Fist`. |
| 37 | `sourceOfIncomeOrFunds` | String | @Size(max=100) | (no validation) | `khonsu_business_prospect.source_of_income_or_funds` varchar(255) · `hathor_business.source_of_income_or_funds` varchar(255) | ❌ | Gateway 100, DB 255. |
| 38 | `purposeOfAccount` | String | @Size(max=100) | (no validation) | `khonsu_business_prospect.purpose_of_account` varchar(255) · `hathor_business.purpose_of_account` varchar(255) | ❌ | Same as above. |

### Owner (keyPerson + owners[])

| # | Field | Type | Gateway DTO | Downstream (`BuProspectOwner`) | DB column (`khonsu_bu_prospect_owners`) | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 39 | `firstName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `first_name` varchar(20) NOT NULL | ✅ | |
| 40 | `lastName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `last_name` varchar(20) NOT NULL | ✅ | |
| 41 | `email` | String | @Size(max=100) | @Email | `email` varchar(100) | ⚠ | Gateway missing @Email. |
| 42 | `phoneNumber` | String | @Size(max=20) | @Pattern(`(^$\|\\d{6,15})`) | `phone_number` varchar(20) | ⚠ | Different ranges (gateway 20-char free-form vs downstream 6–15 digits). |
| 43 | `phoneCountryCode` | String | @Pattern("^\\+?\\d{1,4}$") | @Pattern("^\\+?\\d{1,4}$") | `phone_country_code` varchar(6) | ✅ | |
| 44 | `citizenship` | String | @Size(max=70) | (no validation) | `citizenship` varchar(70) | ✅ | |
| 45 | `jobTitle` | String | @Size(max=50) | (no validation) | `job_title` varchar(50) | ✅ | |
| 46 | `identificationType` | String | @Size(max=40) @Pattern(SSN/SSN4/DL/Passport/National Id/Other) | (no validation) | `identification_type` varchar(40) | ⚠ | Gateway-only enforcement. |
| 47 | `identificationNumber` | String | @Size(max=60) | (no validation) | `identification_number` varchar(60) | ⚠ | Same. |
| 48 | `dateOfBirth` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | @DateTimeFormat("yyyy-MM-dd") | `date_of_birth` date | ⚠ | Missing `@Past`. |
| 49 | `ownershipPercentage` | int | (primitive) | BigDecimal | `ownership_percentage` numeric | ⚠ | Type drift (int → BigDecimal). No @Min/@Max. |
| 50 | `noAdditional25PercentOwner` | boolean | (primitive) | — | (not persisted) | ⚠ | |
| 51 | `address.addressLine1` | String | @NotEmpty @Size(max=80) | mapped → ProspectAddress | `address_one` varchar(80) | ✅ | |
| 52 | `address.addressLine2` | String | @Size(max=40) | mapped | `address_two` varchar(40) | ✅ | |
| 53 | `address.zipCode` | String | @NotEmpty @Size(max=10) | mapped | `postal_code` varchar(10) | ✅ | |
| 54 | `address.city` | String | @NotEmpty @Size(max=128) | mapped | `city` varchar(128) | ✅ | |
| 55 | `address.state` | String | @NotEmpty @Size(max=64) | mapped | `state` varchar(64) | ✅ | |
| 56 | `address.countryCode` | String | @NotEmpty (no @Size!) | mapped | `country_code` varchar(20) | ❌ | Same as #11. |

### Top-level extras

| # | Field | Type | Gateway DTO | Persisted to | Notes |
|---|---|---|---|---|---|
| 57 | `clientIpAddress` | String | (no @Size) | `khonsu_business_prospect.client_ip_address` varchar(50) · `osiris_business_account.ip_address` varchar(50) | ⚠ |
| 58 | `correlationId` | String | (no @Size) | api-gateway-local entity | ⚠ |
| 59 | `productId` | String | (no @Size) | `osiris_business_account.product_id` varchar(100) | ⚠ |
| 60 | `accountName` | String | (no @Size) | `osiris_client_account.account_name` varchar(60) (shared with individual flow) | ❌ Same gap as Set 1: no gateway cap; DB caps at 60. |

---

## 2. `POST /admin/business/accounts/add` | `POST /external/business/accounts/add` — Create business spending account

- **Controller:** `BusinessAccountController.createBusinessSpendingAccount`
- **Request DTO (gateway):** `com.glidetech.api_gateway.api.v1.domain.business.CreateSpendingAccountRequest`
- **Service flow:** `CompanyUserAccountService.createBusinessSpendingAccount` → `ClientAccountService.createBusinessSpendingAccount(name)` → `ClientAccountManager.createBusinessSpendingAccount(CreateSpendingAccountRequest)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.clientaccount.api.v1.domain.spending.CreateSpendingAccountRequest`
- **Persisted to:** new row in `osiris_business_account` (spending variant)

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `name` | String | @NotBlank | @NotBlank | `osiris_business_account.account_id`? · `osiris_client_account.account_name` varchar(60) | ⚠ | No @Size at any layer. Account name written into varchar(60). |
| 2 | `businessId` | Long | (no validation) | (not in downstream DTO) | — | ⚠ | Validated via service `validationService.validateBusinessAccess`. |
| 3 | `clientIpAddress` | String | (no validation) | — | — | ⚠ | |
| 4 | `correlationId` | String | (no validation) | — | (api-gateway local) | ⚠ | Validated server-side via `validationService.validateCorrelationId`. |

---

## 3. `POST /admin/business/prospects` | `POST /external/business/prospects` — Step 1: create business prospect

- **Controller:** `BusinessAccountController.createBusinessProspect`
- **Request DTO (gateway):** `CreateBusinessProspectRequest`
- **Service flow:** `ProspectService.createBusinessProspectForController` → maps to `CreateBusinessAccountRequest` (internally), then → `createBusinessProspect` → `CompanyMapper.mapToBusinessProspect` → `ProspectManager.createProspect(BusinessProspect)` (**glide-prospects**)
- **Downstream DTO:** `BusinessProspect`
- **Persisted to:** `khonsu_business_prospect` + api-gateway `CompanyUserAccountEntity` row (`saveProspectAccountEntity`)

`CreateBusinessProspectRequest` re-uses `CreateBusinessAccountRequest.PersonalInfo` and `CreateBusinessAccountRequest.Address` directly, plus a slimmed-down `BusinessInfo`. All rows from PersonalInfo (#1–5) and Address (#6–11) above apply identically here.

### BusinessInfo (slimmed)

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `company` | String | @NotEmpty (no @Size) | (no validation) | `khonsu_business_prospect.compnay` varchar(40) NOT NULL · `hathor_business.company` varchar(100) NOT NULL | ⚠ | **`@NotEmpty` here but no `@NotEmpty` in legacy DTO row #12** — same field, two different rules. No @Size at all. |
| 2 | `companyWebsite` | String | @Size(max=50) | — | `khonsu_business_prospect.compnay_website` varchar(50) **NOT NULL** | ❌ | Same NOT NULL trap as legacy DTO row #20. |
| 3 | `socialMedia` | String | @Size(max=100) | — | `khonsu_business_prospect.social_media` varchar(100) | ✅ | |
| 4 | `no25PercentOwnership` | boolean | (primitive) | — | `khonsu_business_prospect.no_25pct_ownership` bool | ✅ | |
| 5 | `entityType` | String | @Size(max=150) | — | `khonsu_business_prospect.entity_type` varchar(150) | ✅ | |
| 6 | `companyProfile` | String | @Size(max=100) | — | `khonsu_business_prospect.company_profile` text | ⚠ | |

**Drift:** the legacy `CreateBusinessAccountRequest.BusinessInfo` has many more fields than `CreateBusinessProspectRequest.BusinessInfo` (e.g. `businessEin`, `phoneNumber`, `stateOfIncorporation`, `primaryIndustry`, …). In the new 4-step flow those are deferred to Step 4 (`AddBusinessAdditionalInfoRequest`). Make sure to align validation between the legacy and new shapes.

---

## 4. `PUT /admin/business/prospects/{prospectId}` | `PUT /external/business/prospects/{prospectId}` — Update business prospect (Step 1 update)

- **Controller:** `BusinessAccountController.updateBusinessProspect`
- **Request DTO (gateway):** `CreateBusinessProspectRequest` (re-used)
- **Service flow:** `ProspectService.updateBusinessProspect` → `ProspectManager.updateBusinessProspect(BusinessProspect, prospectId)` (**glide-prospects**) → then `clientManager.updateBusinessEntity(...)` to also sync `hathor_business`
- **Downstream DTOs:** `BusinessProspect` + `com.glidetech.client.api.v1.domain.business.CreateBusinessRequest` (the latter is what writes to `hathor_business`)
- **Persisted to:** `khonsu_business_prospect` UPDATE, `hathor_business` UPDATE (if account already created)

Same field set / validations as Set 2 §3. All the PersonalInfo / Address / BusinessInfo rows in §1 and §3 apply.

**Additional gap:** `clientManager.updateBusinessEntity` builds a `CreateBusinessRequest` with `phone`, `phoneCountryCode`, etc. — those write to `hathor_business.phone` **varchar(10)** while gateway allows 20 (see §1 row #4).

---

## 5. `POST /admin/business/prospects/{prospectId}/owners` | `POST /external/...` — Step 2: add owners

- **Controller:** `BusinessAccountController.addBusinessOwners`
- **Request DTO (gateway):** `AddBusinessOwnersRequest` (wraps `List<Owner>` where `Owner extends CreateBusinessAccountRequest.Owner` + adds `ownerType`)
- **Service flow:** `ProspectService.createBusinessOwners(owners, prospectId, ownerType)` → `CompanyMapper.mapToBuProspectOwner` → `ProspectManager.createBuProspectOwners(BuProspectOwner, prospectId)` (**glide-prospects**)
- **Downstream DTO:** `BuProspectOwner`
- **Persisted to:** `khonsu_bu_prospect_owners`

Wrapper validation:

| # | Field | Type | Gateway DTO | Notes |
|---|---|---|---|---|
| 1 | `owners` | List<Owner> | @NotEmpty @Valid | ✅ Validates list non-empty + cascades validation per element. |
| 2 | `Owner.ownerType` | String | @NotBlank @Pattern("^(KeyPerson\|Owners)$") | ✅ Strict enum. |

All `Owner` fields are inherited from `CreateBusinessAccountRequest.Owner` — see §1 rows #39–56 for the full per-field trace. The `khonsu_bu_prospect_owners.owner_type` is varchar(20), fully covered by the gateway pattern.

---

## 6. `PUT /admin/business/prospects/{prospectId}/owners/{ownerId}` | `PUT /external/...` — Update business owner (Step 2 update)

- **Controller:** `BusinessAccountController.updateBusinessOwner`
- **Request DTO (gateway):** `UpdateBusinessOwnerRequest` (wraps a single `Owner`)
- **Service flow:** `ProspectService.updateBusinessOwner` → `ProspectManager.updateBuProspectOwners(BuProspectOwner, prospectId)` (**glide-prospects**)
- **Downstream DTO:** `BuProspectOwner`
- **Persisted to:** `khonsu_bu_prospect_owners` UPDATE

Same field-by-field as §5 / §1 Owner. Wrapper validation is `@NotNull @Valid` on `owner` plus `@NotBlank @Pattern` on `ownerType` (same as §5 row #2).

---

## 7. `POST /admin/business/prospects/{prospectId}/account` | `POST /external/...` — Step 3: create business account from prospect

- **Controller:** `BusinessAccountController.createBusinessAccountFromProspect`
- **Request DTO (gateway):** `CreateAccountFromProspectRequest` — **just one field: `accountName`** (no validation annotations)
- **Service flow:** `CompanyUserAccountService.createBusinessAccountFromProspect` → `ClientAccountManager.createBusinessAccount(prospectId, BusinessAcProp)` (**glide-client-account**)
- **Downstream DTO:** `BusinessAcProp`
- **Persisted to:** `osiris_business_account` (new row), `osiris_client_account` (new row with `account_name`), `hathor_business`, `hathor_client_address`

| # | Field | Type | Gateway DTO | Downstream (`BusinessAcProp`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `accountName` | String | (no validation) | (no validation on `accountName`) | `osiris_client_account.account_name` **varchar(60)** | ❌ | No length cap anywhere except DB. Should be `@Size(max=60)` at gateway. |

`BusinessAcProp` also carries `isInternal`, `accountNumber`, `isApproved`, `ignoreKeycloakUser`, `accountType`, `productId`, `currency` — these are all set server-side from the saved `CompanyUserAccountEntity` / `CompanyProductAccessEntity` and are not partner-supplied.

**Critical:** This is the only validation on the DTO. A partner could send `{"accountName": "<10000 chars>"}` and the request would pass gateway validation, fail at DB INSERT.

---

## 8. `POST /admin/business/{businessId}/additional-info` | `POST /external/...` — Step 4: add additional business info

- **Controller:** `BusinessAccountController.addBusinessAdditionalInfo`
- **Request DTO (gateway):** `AddBusinessAdditionalInfoRequest`
- **Service flow:** `BusinessService.addAdditionalBusinessInfoFromRequest` (helper) → `AccountMapper.mapToBusinessAdditionalInfoFromRequest` → `BusinessManager.createAdditionalInfo(businessId, BusinessAdditionalInfo)` (**glide-business**)
- **Downstream DTO:** `BusinessAdditionalInfo`
- **Persisted to:** `cronus_business_additional_info`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `dateEstablished` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | LocalDate (no @Past) | `cronus_business_additional_info.date_established` date | ⚠ | Missing `@Past`. |
| 2 | `businessEin` | String | @Size(max=30) | (none) | `business_ein` varchar(30) | ✅ | No `@Pattern` for EIN format (e.g. `\d{2}-\d{7}`). |
| 3 | `phoneNumber` | String | (no @Size, no @Pattern) | (none) | `phone_number` varchar(20) | ❌ | Same gap as #14 in §1. |
| 4 | `stateOfIncorporation` | String | @Size(max=15) | (none) | `state_of_incorporation` varchar(15) | ✅ | |
| 5 | `typeOfBusiness` | String | (no @Size) | (none) | `type_of_business` **varchar(30)** | ❌ | No length cap anywhere except DB. |
| 6 | `destinationOfFunds` | String | (no @Size) | (none) | `destination_of_funds` **varchar(30)** | ❌ | Same. |
| 7 | `reasonRelationWithSuvi` | String | (no @Size) | (none) | `reason_relation_with_suvi` **varchar(30)** | ❌ | Same. |
| 8 | `businessGenerationMethod` | String | (no @Size) | (none) | `business_generation_method` **varchar(30)** | ❌ | Same. |
| 9 | `primaryIndustry` | String | @Size(max=80) | (none) | `primary_industry` varchar(80) | ✅ | |
| 10 | `subIndustry` | String | @Size(max=100) | (none) | `sub_industry` varchar(100) | ✅ | |
| 11 | `companyType` | String | @Size(max=150) | (none) | `company_type` varchar(150) | ✅ | |
| 12 | `annualRevenue` | String | @Size(max=40) | (none) | `annual_revenue` varchar(40) | ✅ | |
| 13 | `customerSupplierBased` | String | @Size(max=50) | (none) | `customer_supplier_based` varchar(50) | ✅ | |
| 14 | `approximateMonthlyTransaction` | String | (no @Size) | (none) | `approximate_monthly_transaction` varchar(40) | ❌ | No length cap at gateway. |
| 15 | `businessIdentificationType` | String | @Size(max=9) (no @Pattern) | (none) | `business_identification_type` varchar(10) | ⚠ | Note: legacy `CreateBusinessAccountRequest.BusinessInfo.businessIdentificationType` has @Pattern; this one doesn't. |
| 16 | `approximateDepositFistMonth` | String | (no @Size) | (none) | `approximate_deposit_fist_month` varchar(40) | ❌ | No gateway cap. Note column name typo `fist` (should be `first`). |
| 17 | `companyMission` | String | @Size(max=255) | (none) | `company_mission` varchar(255) | ✅ | |
| 18 | `sourceOfFunds` | String | (no @Size) | (none) | `source_of_funds` **varchar(60)** | ❌ | No gateway cap. |
| 19 | `sessionId` | String | (no @Size) | (none) | `session_id` varchar(40) | ⚠ | Usually set server-side. |
| 20 | `agreeToTermsAndConditions` | Boolean | (default false) | (default false) | `cronus_business_profile.agree_to_terms_and_conditions` bool | ✅ | Lives on `cronus_business_profile`, not `cronus_business_additional_info`. |

**Schema drift (not from DTO but worth flagging):** `cronus_business_additional_info` has DB columns `industry` (varchar 60) and `customer_type` (varchar 60) that are NOT mapped by the JPA entity. Either pull them into the entity or remove via migration.

---

## 9. `PUT /admin/business/{businessId}/additional-info` | `PUT /external/...` — Update additional business info (Step 4 update)

- **Controller:** `BusinessAccountController.updateBusinessAdditionalInfo`
- **Request DTO (gateway):** `AddBusinessAdditionalInfoRequest` (re-used)
- **Service flow:** `BusinessService.updateAdditionalBusinessInfo` → `BusinessManager.updateAdditionalInfo(businessId, BusinessAdditionalInfo)` (**glide-business**)
- **Downstream DTO:** `BusinessAdditionalInfo`
- **Persisted to:** `cronus_business_additional_info` UPDATE

Same field-by-field as §8 (DTO re-used).

---

## Async / Kafka-driven persistence

Set 2 emits Kafka events on every entity persist. Two downstream tables (`cronus_business_users`, `hathor_business_client`) receive partner-supplied keyPerson data in typed columns with width mismatches — already in Headline rows #21–#23. Full column breakdown below.

**Event chain:** `osiris_business_account` `@PostPersist` → `ClientProfileCreationEvent` (carries keyPerson firstName/lastName/email/phone/companyId) → topic `glide.kafka.consumer.client-account.event` → glide-business `KafkaEventConsumer.createBusinessClient` → INSERT `cronus_business_users` + assign default group. `BusinessClientEntity` (`hathor_business_client`) is written by glide-client when the keyPerson becomes a business-client.

### `cronus_business_users` — async INSERT (full schema)

| # | Column | Type | Source | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `id` | bigint | server | ✅ | PK |
| 2 | `business_id` | bigint NOT NULL | server (from event) | ✅ | |
| 3 | `client_id` | bigint | server (after client created) | ✅ | |
| 4 | `first_name` | varchar(**255**) | `keyPerson.firstName` (gateway @Size=20) | ❌ | New outlier — 20/32/50/150/255 across codebase. |
| 5 | `last_name` | varchar(**255**) | `keyPerson.lastName` (gateway @Size=20) | ❌ | Same. |
| 6 | `email` | varchar(**255**) | `keyPerson.email` (gateway @Size=100) | ❌ | Inconsistent with `hathor_*.email`(100). |
| 7 | `phone` | varchar(**15**) | `keyPerson.phoneNumber` (gateway @Size=20) | ❌ | **Truncation/INSERT fail** for 16–20 char phones. |
| 8 | `status` | varchar(15) | server | ✅ | enum-as-string |
| 9 | `session_id` | varchar(40) | server | ✅ | |
| 10 | `monthly_fx_limit` | numeric | server (policy) | ✅ | |
| 11 | `bank_transfer_allowed` | boolean | server | ✅ | |
| 12 | `group_to_assign` | bigint | server | ✅ | FK to `cronus_business_user_groups` |
| 13 | `created_by`, `created_on`, `updated_on` | bigint / timestamps | server | ✅ | |

### `hathor_business_client` — sync write by glide-client when keyPerson linked

| # | Column | Type | Source | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `id` | bigint | server | ✅ | PK |
| 2 | `client_id` | bigint NOT NULL | server | ✅ | |
| 3 | `business_id` | bigint NOT NULL | server | ✅ | |
| 4 | `first_name` | varchar(32) **NOT NULL** | `keyPerson.firstName` (gateway @Size=20) | ⚠ | Gateway tighter (safe), but NOT NULL — mapper must always send a value. |
| 5 | `last_name` | varchar(32) **NOT NULL** | `keyPerson.lastName` (gateway @Size=20) | ⚠ | Same. |
| 6 | `email` | varchar(100) | `keyPerson.email` (gateway @Size=100) | ✅ | |
| 7 | `phone` | varchar(**10**) | `keyPerson.phoneNumber` (gateway @Size=20) | ❌ | Third width for the same field (10/15/20 across hathor_business / cronus_business_users / gateway). |
| 8 | `phone_country_code` | varchar(6) | `keyPerson.phoneCountryCode` (gateway @Pattern \d{1,4}) | ✅ | |
| 9 | `address_id` | bigint | FK → `hathor_client_address` | ✅ | |
| 10 | `request_identifier` | varchar (**unbounded**) | server | ❌ | Schema oversight — no `length=` set. |
| 11 | `removed` | boolean | server | ✅ | |
| 12 | `company_id`, `relationship_id` | bigint | server | ✅ | |
| 13 | `created_on`, `updated_on` | timestamp | server | ✅ | |

### Other async-touched tables (no validation alignment needed)

| Table | Why no gap | Verdict |
|---|---|---|
| `osiris_business_lexnex_response` | Partner data in `request`/`response` jsonb. Typed cols are server-controlled (`business_lexnex_check_status` 20 vs `owner_lexnex_check_status` 30 — minor enum drift noted in Headline #24) | ⚠ |
| `osiris_business_owner_kyc_status` | Only server-controlled enums (`owner_id` 255, `ofac_status`/`kyc_status` 30) | ❌ Skip |
| `hathor_signzy`, `hathor_ofac`, `hathor_ofac_address` | Partner data in text/jsonb blobs (same shape as Set 1) | ❌ Skip |
| `cronus_business_user_groups`, `cronus_user_group_membership`, `cronus_user_policy_mapping`, `cronus_user_group_rules` | Default groups/FK linkage seeded server-side | ❌ Skip |
| `cronus_business_dashboard_modules`, `cronus_dashboard_modules`, `cronus_user_dashboard_modules` | Boolean flags + module_id refs, seeded on `BusinessAccountVerifiedEvent` | ❌ Skip |
| `cronus_state_machine` | Server-controlled state + `attributes` jsonb | ❌ Skip |
| `cronus_transaction_policy` | Default policy seeded server-side; `account_number`/`identifier`/`transaction_type` are **unbounded varchar** (schema oversight — flagged in recommendation #18) | ⚠ |
| `iris_account_action_history`, `iris_account_review`, `demeter_webhook_outbound_event_log` | Same shape as Set 1 (audit / outbound webhook log) | ❌ Skip |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /business/accounts` | `BusinessInfo.companyWebsite` → mapper → `khonsu_business_prospect.compnay_website` | ✅ Confirmed NOT NULL on DB col + typo. Confirmed legacy DTO has it as optional. |
| `POST /business/accounts` | `Address.addressLine1` → mapper → `khonsu_business_prospect.address_one` varchar(30) | ✅ Confirmed truncation risk. |
| `POST /business/accounts/add` | `name` → `osiris_client_account.account_name` varchar(60) | ✅ Confirmed no @Size anywhere. |
| `POST /business/prospects` | `BusinessInfo.company` — `@NotEmpty` at gateway here but not in legacy DTO | ✅ Confirmed inconsistency between the two DTOs. |
| `POST /business/prospects/{id}/owners` | `Owner.identificationNumber` → `khonsu_bu_prospect_owners.identification_number` varchar(60) | ✅ Aligned. |
| `POST /business/prospects/{id}/account` | `accountName` → `osiris_client_account.account_name` varchar(60) (no @Size at gateway) | ✅ Confirmed gap. |
| `POST /business/{id}/additional-info` | `typeOfBusiness` → `cronus_business_additional_info.type_of_business` varchar(30) (no @Size at gateway) | ✅ Confirmed truncation risk. |
| `POST /business/accounts` (async chain) | `keyPerson.phoneNumber` → `ClientProfileCreationEvent` → glide-business `KafkaEventConsumer.createBusinessClient` → `cronus_business_users.phone` varchar(15) | ✅ Confirmed truncation risk — gateway @Size(20) > DB(15). |
| `POST /business/accounts` (async chain) | `keyPerson.firstName/lastName/email` → same chain → `cronus_business_users` varchar(255) | ✅ Confirmed new outlier widths. |
| `POST /business/accounts` (sync, but linked downstream) | `keyPerson.phoneNumber` → `hathor_business_client.phone` varchar(10) | ✅ Confirmed third phone width for the same field. |

---

## Recommended next-step actions (for the consistency pass — not in scope of this doc)

1. **Critical**: Make `khonsu_business_prospect.compnay_website` NULLABLE — partner flows currently treat it as optional, but the column is NOT NULL.
2. **Critical**: Standardise `phoneNumber` validation. Either accept up to 15 digits everywhere (gateway `@Pattern("\\d{1,15}")`, DB varchar(20)) or strict 10 digits, but not both.
3. Fix typos in `khonsu_business_prospect`: `compnay` → `company`, `compnay_website` → `company_website`. Will require a Flyway migration + entity rename.
4. Fix typo in `cronus_business_additional_info`: `approximate_deposit_fist_month` → `..._first_month`.
5. Reconcile two `BusinessInfo` shapes — legacy `CreateBusinessAccountRequest.BusinessInfo` (24 fields) vs new-flow `CreateBusinessProspectRequest.BusinessInfo` (6 fields). Either one DTO or clearly separate them.
6. Add `@Size` to all `AddBusinessAdditionalInfoRequest` fields that hit varchar columns: `typeOfBusiness(30)`, `destinationOfFunds(30)`, `reasonRelationWithSuvi(30)`, `businessGenerationMethod(30)`, `approximateMonthlyTransaction(40)`, `approximateDepositFistMonth(40)`, `sourceOfFunds(60)`, `phoneNumber(20)`.
7. Add `@Size(max=60)` to `CreateAccountFromProspectRequest.accountName` and `CreateSpendingAccountRequest.name`.
8. Decide canonical length for `company` (40 prospect vs 100 client) and align both columns.
9. Decide canonical length for `companyWebsite` (50 prospect vs 100 client) — recommend 255 since URLs can be long.
10. Standardise `osiris_business_account.account_id`/`account_type` lengths with their `osiris_client_account` siblings (255 vs 20/30 today).
11. Add `@Email` to gateway `email` fields (PersonalInfo, Owner).
12. Add `@Past` to all `dateOfBirth` and `dateEstablished` fields.
13. Standardise `countryCode` to ISO-3 with `@Size(max=3)` everywhere (and DB varchar(3)) — also addresses `khonsu_business_prospect.country_code` varchar(20) outlier.
14. Resolve drift: `cronus_business_additional_info.industry` and `.customer_type` are in DB but not in entity — pull in or drop.
15. Add `@NotBlank` + `@Size` to `CreateSpendingAccountRequest.name`.
16. **Critical (async):** Widen `cronus_business_users.phone` from varchar(15) to at least varchar(20) to match gateway @Size, or tighten gateway to varchar(15). Same field is currently varchar(10) in `hathor_business`/`hathor_business_client` — pick one canonical width across all three.
17. (async) Tighten `cronus_business_users.first_name`/`last_name`/`email` from varchar(255) to the canonical width chosen in recommendation #11 (gateway is @Size=20 for names, 100 for email).
18. (async) Add `length=` to `cronus_transaction_policy.account_number`, `identifier`, `transaction_type` (currently unbounded varchar — schema oversight on the entity that triggers default-policy seeding).
19. (async) Reconcile `osiris_business_lexnex_response.business_lexnex_check_status` (varchar 20) with `.owner_lexnex_check_status` (varchar 30) — same logical KYC-status field, two widths.
20. (async) Add `length=` to `hathor_business_client.request_identifier` (currently unbounded varchar).
