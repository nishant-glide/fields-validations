# Set 1 — Onboarding (Individual)

7 endpoints under `AccountController` + `IdentityVerificationController`. These cover the full individual-onboarding lifecycle: prospect creation, account creation, KYC update, profile update, compliance update, and SSN identity lookup.

---

## Headline findings (consolidated)

The biggest cross-cutting gaps surfaced by this set:

| # | Field | Layer divergence | Risk |
|---|---|---|---|
| 1 | `accountName` | gateway @Size=80, but `osiris_client_account.account_name` = **varchar(60)** | ❌ Truncation: a partner-supplied 70-char account name will be rejected/truncated at DB write. |
| 2 | `identificationNumber` / `id_number_updated` | gateway @Size=60, `khonsu_prospect.identification_number`=64, `hathor_client.identification_number`=60, `osiris_spending_account.id_number_updated`=**260** | ❌ Same logical field, four different lengths. |
| 3 | `identificationType` / `id_type_updated` | gateway @Size=40, `khonsu_prospect.identification_type`=40, `hathor_client.identification_type`=40, `osiris_spending_account.id_type_updated`=**240** | ❌ Same logical field, two different lengths. |
| 4 | `sourceOfIncomeOrFunds` | gateway @Size=100, `khonsu_prospect`=varchar(255), `hathor_client`=varchar(255) | ❌ Gateway is the bottleneck — DB allows 255, partner cap is 100. |
| 5 | `purposeOfAccount` | gateway @Size=100, DB columns=varchar(255) | ❌ Same as above. |
| 6 | `countryCode` (Address) | gateway @Size=2, `khonsu_prospect.country_code`=varchar(20), `hathor_client_address.country_code`=varchar(3) | ❌ Three different lengths for ISO country code. |
| 7 | `phoneNumber` | gateway @Size=20, `Prospect` DTO has `@Pattern("[0-9]{4,15}")`, no DB pattern | ⚠ Gateway accepts non-digit content; downstream rejects. |
| 8 | `dateOfBirth` (CreateAccountRequest only) | gateway is `String @NotEmpty` (no format), `Prospect` is `LocalDate` with `@DobValidator`, DB is `date` | ⚠ Gateway accepts any string; format failure surfaces only deep downstream. |
| 9 | `CloseAccountReq` | All 5 fields have **zero validation** (no @NotNull / @Size / @Pattern) | ❌ Partner can send empty/oversize payload; service-layer guards via `validateAndSetContext`. |
| 10 | `UpdateProspectFieldsRequest` data fields | `sourceOfIncomeOrFunds`, `purposeOfAccount` have NO `@Size` at gateway, but downstream `UpdateComplianceDataRequest` only has `@NotNull` (also no length); DB has 255 | ⚠ No upper bound enforced anywhere except DB. |
| 11 | `CreateSignzyJourneyRequest` | Gateway controller accepts the **downstream DTO directly** (no gateway-layer DTO defined). No validation anywhere; only a manual `null`-check in the controller. | ⚠ Gateway leaks downstream contract; no field-level validation. |
| 12 | `PartnerSsnFetchRequest` | Gateway has strong `@Pattern` validation for phone/SSN/DOB. Downstream `SsnFetchRequest` has **no validation** on those fields. | ⚠ Consistent only because gateway enforces it. |
| 13 | **Async / Kafka-driven persistence** — Set 1 entities emit Kafka events that produce side-effect writes to `hathor_signzy`, `hathor_ofac`, `hathor_ofac_address`, `iris_account_action_history`, `iris_account_review`, `demeter_webhook_outbound_event_log`, `hathor_user_credential_update_log`. **None of these store partner data in typed columns** that need validation alignment — all use jsonb/text payloads or server-controlled enums. See "Async / Kafka-driven persistence" section after §7 for the full trace. | ✅ No new gaps from the async surface. |

Mythological-prefix mapping confirmed for this set:
- `khonsu_*` → **glide-prospects** (prospect data)
- `osiris_*` → **glide-client-account** (client/spending account lifecycle)
- `hathor_*` → **glide-client** (client master record, address)

Note: `correlationId`, `productId`, and `clientIpAddress` are stored on api-gateway's own table (`CompanyUserAccountEntity`); not included in the per-API tables below because they don't travel to a downstream contract field.

---

## 1. `POST /admin/accounts` | `POST /external/accounts` — Create individual account

- **Controller:** `AccountController.createAccount` ([file](../../glide-api-gateway/service/src/main/java/com/glidetech/api_gateway/service/controller/AccountController.java#L55-L74))
- **Request DTO (gateway):** `CreateAccountRequest` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/account/CreateAccountRequest.java))
- **Service flow:** `CompanyUserAccountService.createAccount` → fans out to:
  1. `ProspectService.createProspect` → `CompanyMapper.mapToProspect` → `ProspectManager.createProspect(Prospect)` (module: **glide-prospects**)
  2. `ClientAccountService.createIndividualAccount` → `ClientAccountManager.createSpendingAccount(CreateIntegrationSpendingAcRequest)` (module: **glide-client-account**)
- **Downstream DTOs:** `Prospect` (`glide-prospects/api/.../domain/Prospect.java`), `CreateIntegrationSpendingAcRequest` (`glide-client-account/api/.../domain/account/CreateIntegrationSpendingAcRequest.java`)
- **Persisted to:**
  - `khonsu_prospect` (glide-prospects, JPA: `ProspectEntity`)
  - `osiris_spending_account` (glide-client-account, JPA: `SpendingAccountEntity`)
  - `osiris_client_account` (glide-client-account, JPA: `ClientAccountEntity`) — created downstream
  - `hathor_client` + `hathor_client_address` (glide-client) — client master record created downstream

### PersonalInfo

| # | Field | Type | Gateway DTO | Downstream DTO (`Prospect`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `firstName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `khonsu_prospect.first_name` varchar(20) NOT NULL · `hathor_client.first_name` varchar(32) NOT NULL | ❌ | hathor_client column is wider (32) than every other layer (20). Slack; not a truncation risk. |
| 2 | `lastName` | String | @NotEmpty @Size(max=20) | @NotNull @Size(min=2,max=20) | `khonsu_prospect.last_name` varchar(20) · `hathor_client.last_name` varchar(32) | ❌ | Same as firstName. |
| 3 | `email` | String | @NotEmpty @Size(max=100) | @Email (no @Size) | `khonsu_prospect.email` varchar(100) · `hathor_client.email_id` varchar(100) | ⚠ | Lengths align (100). Downstream missing `@Size` — relies on DB cap. |
| 4 | `phoneNumber` | String | @NotEmpty @Size(max=20) | @Pattern("[0-9]{4,15}") | `khonsu_prospect.phone_number` varchar(20) · `hathor_client.primary_phone` varchar(20) | ⚠ | Gateway has no `@Pattern` — non-digit strings up to 20 chars pass gateway, rejected at Prospect layer. CLAUDE.md convention says use `\d{1,15}` at gateway. |
| 5 | `phoneCountryCode` | String | @Pattern("\\d{1,4}") | @Pattern("\\d{1,4}") | `khonsu_prospect.phone_country_code` varchar(6) · `hathor_client.phone_country_code` varchar(6) | ✅ | Regex caps at 4; DB allows 6 (harmless slack). |
| 6 | `identificationType` | String | @NotEmpty @Size(max=40) @Pattern("^(SSN\|SSN4\|DL\|Passport\|National Id\|Other)$") | (none on Prospect) | `khonsu_prospect.identification_type` varchar(40) · `hathor_client.identification_type` varchar(40) · `osiris_spending_account.id_type_updated` **varchar(240)** | ❌ | Same logical field has length 40 in three places and 240 in `osiris_spending_account.id_type_updated`. |
| 7 | `identificationNumber` | String | @NotEmpty @Size(max=60) | (none on Prospect) | `khonsu_prospect.identification_number` varchar(64) · `khonsu_prospect.ssn` varchar(64) · `khonsu_prospect.ssn_last_4_digits` varchar(64) · `hathor_client.identification_number` varchar(60) · `hathor_client.ssn_number` varchar(64) · `hathor_client.ssn_last_4_digits` varchar(64) · `osiris_spending_account.id_number_updated` **varchar(260)** | ❌ | 60 / 64 / 260 across the chain. ProspectService.createProspect splits this into either `ssn` or `ssn_last_4_digits` based on type; both stored separately. |
| 8 | `dateOfBirth` | String (gateway) | @NotEmpty (String!) | LocalDate @DobValidator @JsonFormat("yyyy-MM-dd") | `khonsu_prospect.date_of_birth` date · `hathor_client.date_of_birth` date | ⚠ | Gateway accepts any non-empty string. Recommend `LocalDate` + `@JsonFormat("yyyy-MM-dd")` + `@Past` per CLAUDE.md convention. |

### Address

| # | Field | Type | Gateway DTO | Downstream (`ProspectAddress` via mapper) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 9 | `addressLine1` | String | @NotEmpty @Size(max=80) | mapped to `prospectAddress.address1` | `khonsu_prospect.address_one` varchar(80) · `hathor_client_address.address_one` varchar(80) NOT NULL | ✅ | |
| 10 | `addressLine2` | String | @Size(max=80) | mapped to `prospectAddress.address2` | `khonsu_prospect.address_two` varchar(80) · `hathor_client_address.address_two` varchar(80) | ✅ | |
| 11 | `zipCode` | String | @NotEmpty @Size(max=10) | mapped to `prospectAddress.zipCode` | `khonsu_prospect.postal_code` varchar(10) · `hathor_client_address.postal_code` varchar(10) NOT NULL | ✅ | |
| 12 | `city` | String | @NotEmpty @Size(max=128) | mapped to `prospectAddress.city` | `khonsu_prospect.city` varchar(128) · `hathor_client_address.city` varchar(128) NOT NULL | ✅ | |
| 13 | `state` | String | @NotEmpty @Size(max=64) | mapped to `prospectAddress.state` | `khonsu_prospect.state` varchar(64) · `hathor_client_address.state` varchar(64) NOT NULL | ✅ | |
| 14 | `countryCode` | String | @NotEmpty @Size(max=2) | mapped to `prospectAddress.countryCode` (Prospect default = "US", @Size 2–3) | `khonsu_prospect.country_code` **varchar(20)** · `hathor_client_address.country_code` varchar(3) NOT NULL | ❌ | Gateway: 2 chars, Prospect DTO: 2–3, hathor: 3, khonsu: 20. ISO-3166 alpha-2 is 2 chars, alpha-3 is 3. Recommend `@Size(min=2,max=3)` everywhere and DB `varchar(3)`. |

### FinancialInfo

| # | Field | Type | Gateway DTO | Downstream (`ProspectFinancialDetails`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 15 | `occupation` | String | @Size(max=80) | (deferred — not separately validated) | `khonsu_prospect.occupation` varchar(80) | ✅ | |
| 16 | `incomeSource` | String | @Size(max=40) | — | `khonsu_prospect.income_source` varchar(40) | ✅ | |
| 17 | `annualIncome` | String | @Size(max=40) | — | `khonsu_prospect.annual_income` varchar(40) | ✅ | |
| 18 | `netWorth` | String | @Size(max=50) | — | `khonsu_prospect.net_worth` varchar(50) | ✅ | |
| 19 | `employmentStatus` | String | @Size(max=50) | — | `khonsu_prospect.employment_status` varchar(50) | ✅ | |
| 20 | `paidFrequency` | String | (no @Size) | enum `Frequency` (downstream) | `khonsu_prospect.paid_frequency` varchar(20) (enum-as-string) | ⚠ | Gateway untyped, downstream enum-string. Suggest making gateway an enum or `@Pattern`. |
| 21 | `paidMethod` | String | @Size(max=20) | — | `khonsu_prospect.paid_method` varchar(20) | ✅ | |
| 22 | `investmentGoal` | String | @Size(max=40) | enum `InvestmentGoal` (downstream) | `khonsu_prospect.investment_goal` varchar(40) · `hathor_client.investment_goal` varchar(40) | ⚠ | Same shape gap as paidFrequency. |
| 23 | `sourceOfIncomeOrFunds` | String | @Size(max=100) | (none) | `khonsu_prospect.source_of_income_or_funds` varchar(255) · `hathor_client.source_of_income_or_funds` varchar(255) | ❌ | Gateway tighter (100) than DB (255). Pick one. |
| 24 | `purposeOfAccount` | String | @Size(max=100) | (none) | `khonsu_prospect.purpose_of_account` varchar(255) · `hathor_client.purpose_of_account` varchar(255) | ❌ | Same as above. |
| 25 | `companyProfile` | String | @Size(max=100) | (none) | `khonsu_prospect.company_profile` **text** (unbounded) | ⚠ | DB is unbounded text; gateway caps at 100. Decide intent. |

### StatementApplied

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 26 | `associatedWithBroker` | boolean | (no validation; primitive) | mapped | `khonsu_prospect.associated_with_broker` bool | ✅ | |
| 27 | `publicCompanyShareholder` | boolean | (no validation; primitive) | mapped | `khonsu_prospect.public_company_shareholder` bool | ✅ | |
| 28 | `noneOfThese` | boolean | (no validation; primitive) | mapped | `khonsu_prospect.none_of_these` bool | ✅ | |

### Top-level extras

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 29 | `correlationId` | String | @Size(max=80) | passed to `CompanyUserAccountEntity` (api-gateway-local entity) | (api-gateway local table — out of scope for this doc) | ⚠ | Not propagated to khonsu / osiris / hathor. |
| 30 | `clientIpAddress` | String | @Size(max=45) | passed onto Prospect (`clientIpAddress`) | `khonsu_prospect.client_ip_address` varchar(50) · `hathor_client.ip_address` varchar(45) · `osiris_spending_account.ip_address` varchar(50) | ⚠ | 45 / 50 / 45 / 50. IPv6 maxes at 45 — gateway is right. Recommend tightening DB to 45. |
| 31 | `productId` | String | (no validation) | passed to `CreateIntegrationSpendingAcRequest.productId` | `osiris_spending_account.product_id` varchar(100) | ⚠ | No length cap upstream. |
| 32 | `accountName` | String | @Size(max=80) | `CreateIntegrationSpendingAcRequest.accountName` (no @Size) | `osiris_client_account.account_name` **varchar(60)** | ❌ | **Truncation risk** — gateway allows 80, DB only 60. Either tighten gateway to 60 or widen DB. |

### CreateIntegrationSpendingAcRequest (downstream-only side)

| # | Field | Type | Downstream | DB column | Notes |
|---|---|---|---|---|---|
| — | `prospectId` | String | @NotNull | `osiris_spending_account.prospect_id` varchar(200) | Set server-side from prospect creation. |
| — | `companyId` | Long | @NotNull | `osiris_spending_account.company_id` bigint | Set from request context. |
| — | `productId` | String | (none) | `osiris_spending_account.product_id` varchar(100) | |
| — | `accountName` | String | (none) | (carries through to `osiris_client_account.account_name` varchar(60)) | See row 32. |

---

## 2. `POST /admin/accounts/close` — Close account

- **Controller:** `AccountController.closeAccount`
- **Request DTO (gateway):** `CloseAccountReq` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/account/CloseAccountReq.java))
- **Service flow:** `CompanyUserAccountService.closeAccount` → `ClientAccountService.closeAccount` (helper) → `ClientAccountManager.closeAccount(CloseAccountRequest)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.clientaccount.api.v1.domain.CloseAccountRequest`
- **Persisted to:** state-only update on `osiris_client_account.state` ("closed-pending") and `osiris_spending_account.status`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | (no validation) | identifier — `osiris_client_account.client_id` bigint | ❌ | Should be `@NotNull` unless `businessId` is given. Logic gate is in service layer (`validateAndSetContext`). |
| 2 | `businessId` | Long | (no validation) | (no validation) | `osiris_client_account.business_id` bigint | ❌ | Same as above. |
| 3 | `accountId` | String | (no validation) | (no validation) | `osiris_client_account.account_id` varchar(20) | ❌ | No `@Size`. DB is varchar(20). |
| 4 | `reason` | String | (no validation) | (no validation) | `osiris_client_account.close_reason` varchar(255) | ❌ | No `@Size`. Could exceed DB. |
| 5 | `clientIpAddress` | String | (no validation) | (not propagated) | — | ⚠ | Unused downstream. Drop or wire it. |

Downstream `CloseAccountRequest` also carries `status` (always "closed-pending"), `deleteClient`, `removeKeyCloakUser` — set by service layer, not partner input.

---

## 3. `POST /admin/accounts/create-journey-url` — Create Signzy onboarding URL

- **Controller:** `AccountController.createSignzyJourneyUrl`
- **Request DTO (gateway = downstream):** `com.glidetech.clientaccount.api.v1.domain.signzy.CreateSignzyJourneyRequest` ([file](../../glide-client-account/api/src/main/java/com/glidetech/clientaccount/api/v1/domain/signzy/CreateSignzyJourneyRequest.java))
  - **The gateway controller accepts the downstream DTO directly** — no separate gateway DTO. This violates the CLAUDE.md "Always Use DTOs and Mappers" rule.
- **Service flow:** `CompanyUserAccountService.createSignzyJourneyUrl` → `ClientAccountManager.createSignzyJourneyUrl(request)` (**glide-client-account**)
- **Downstream DTO:** Same instance forwarded.
- **Persisted to:** No direct entity at api-gateway. The Signzy journey itself is recorded in glide-client tables (`hathor_signzy` etc.) when the result returns; partner-supplied fields are not persisted.

| # | Field | Type | Gateway / Downstream (single DTO) | DB column (if any) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `prospectId` | String | (no validation) | links to `khonsu_prospect.identifier` varchar(255) | ⚠ | No `@Size`/`@NotBlank`. |
| 2 | `ownerId` | String | (no validation) | — | ⚠ | |
| 3 | `clientId` | Long | (no validation) | — | ⚠ | Controller does manual null-check (`clientId == null && businessId == null`). |
| 4 | `businessId` | Long | (no validation) | — | ⚠ | Same as above. |
| 5 | `companyId` | Long | (no validation) | — | ⚠ | Usually populated server-side. |
| 6 | `userType` | String | (no validation) | — | ⚠ | |
| 7 | `idType` | String | (no validation) | — | ⚠ | |
| 8 | `country` | String | (no validation) | — | ⚠ | |
| 9 | `flowId` | String | (no validation) | — | ⚠ | |
| 10 | `journeyLinkValidity` | Integer | (no validation) | — | ⚠ | No `@Min/@Max`. |
| 11 | `journeySessionExpiry` | Integer | (no validation) | — | ⚠ | No `@Min/@Max`. |

**Structural gap:** introduce a gateway DTO `CreateJourneyUrlRequest` with strict validation, then map to the downstream DTO.

---

## 4. `PUT /admin/accounts/kyc` — Update identification for KYC

- **Controller:** `AccountController.updateIdentification`
- **Request DTO (gateway):** `com.glidetech.api_gateway.api.v1.domain.account.UpdateIdentificationRequest`
- **Service flow:** `ClientAccountService.updateIdentification` (helper) → `ClientAccountManager.updateIdentification(UpdateIdentificationRequest)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.clientaccount.api.v1.domain.UpdateIdentificationRequest`
- **Persisted to:** glide-client `hathor_client.identification_type`, `hathor_client.identification_number`, plus `osiris_spending_account.id_type_updated`, `osiris_spending_account.id_number_updated`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | @NotNull | @NotNull | (key) | ✅ | |
| 2 | `identificationType` | String | @Size(max=40) @Pattern("^(SSN)$") | @NotNull @Size(max=40) | `hathor_client.identification_type` varchar(40) · `osiris_spending_account.id_type_updated` **varchar(240)** | ❌ | Two storage lengths (40 vs 240). |
| 3 | `identificationNumber` | String | @NotEmpty @Size(max=60) | @NotNull @Size(max=60) | `hathor_client.identification_number` varchar(60) · `hathor_client.ssn_number` varchar(64) · `osiris_spending_account.id_number_updated` **varchar(260)** | ❌ | 60 / 64 / 260. |

**Note:** Gateway `@Pattern("^(SSN)$")` is stricter than downstream (only allows "SSN"). The endpoint description says "Updates the identification type and number for a client" — but the regex forbids any value other than "SSN". This is either intentional (force-full-SSN upgrade flow) or a bug.

---

## 5. `PUT /admin/accounts/individual` | `PUT /external/accounts/individual` — Update individual account

- **Controller:** `AccountController.updateIndividualAccount`
- **Request DTO (gateway):** `com.glidetech.api_gateway.api.v1.domain.account.UpdateIndividualAccountRequest` (has nested `Address`)
- **Service flow:** `ClientAccountService.updateIndividualAccount` (helper) → `AccountMapper.toClientAccountUpdateRequest` → `ClientAccountManager.updateIndividualAccount(...)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.clientaccount.api.v1.domain.individual.UpdateIndividualAccountRequest` (uses `com.glidetech.client.api.v1.domain.ClientAddress`)
- **Persisted to:** `hathor_client` + `hathor_client_address` (glide-client)

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `prospectId` | String | @NotBlank | @NotBlank | `osiris_spending_account.prospect_id` varchar(200) | ⚠ | No @Size at either layer; DB caps at 200. |
| 2 | `firstName` | String | @Size(max=100) | @Size(max=100) | `hathor_client.first_name` varchar(32) | ❌ | **Truncation risk** — gateway 100, DB 32. Mismatch vs Create (which uses 20). |
| 3 | `lastName` | String | @Size(max=100) | @Size(max=100) | `hathor_client.last_name` varchar(32) | ❌ | Same as firstName. |
| 4 | `emailAddress` | String | @Email @Size(max=255) | @Size(max=255) (no @Email!) | `hathor_client.email_id` varchar(100) | ❌ | Gateway 255, DB 100. Downstream missing `@Email`. |
| 5 | `phoneNumber` | String | @Size(max=20) | @Size(max=20) | `hathor_client.primary_phone` varchar(20) | ⚠ | No `@Pattern`. |
| 6 | `phoneCountryCode` | String | @Size(max=5) | @Size(max=5) | `hathor_client.phone_country_code` varchar(6) | ⚠ | Slightly inconsistent — 5 vs 6. |
| 7 | `dateOfBirth` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | @JsonFormat("yyyy-MM-dd") | `hathor_client.date_of_birth` date | ⚠ | Missing `@Past`. |
| 8 | `address.addressOne` | String | @Size(max=40) | mapped to ClientAddress.addressOne | `hathor_client_address.address_one` varchar(80) | ❌ | Gateway 40, DB 80. |
| 9 | `address.addressTwo` | String | @Size(max=40) | mapped | `hathor_client_address.address_two` varchar(80) | ❌ | Same. |
| 10 | `address.city` | String | @Size(max=120) | mapped | `hathor_client_address.city` varchar(128) | ⚠ | 120 vs 128 — close, but inconsistent. |
| 11 | `address.state` | String | @Size(max=40) | mapped | `hathor_client_address.state` varchar(64) | ❌ | Gateway 40, DB 64. |
| 12 | `address.postalCode` | String | @Size(max=10) | mapped | `hathor_client_address.postal_code` varchar(10) | ✅ | |
| 13 | `address.countryCode` | String | @Size(max=3) | mapped | `hathor_client_address.country_code` varchar(3) | ✅ | Note this is **inconsistent with CreateAccountRequest.Address.countryCode** which is @Size(max=2). |

---

## 6. `PUT /admin/accounts/account-info` | `PUT /external/accounts/account-info` — Update prospect compliance fields

- **Controller:** `AccountController.updateAccountInfo`
- **Request DTO (gateway):** `com.glidetech.api_gateway.api.v1.domain.account.UpdateProspectFieldsRequest`
- **Service flow:** `CompanyUserAccountService.updateProspectFields` → `ClientAccountManager.updateComplianceData(UpdateComplianceDataRequest)` (**glide-client-account**, which in turn calls glide-client for client-side and glide-business for business-side updates)
- **Downstream DTO:** `com.glidetech.clientaccount.api.v1.domain.compliance.UpdateComplianceDataRequest`
- **Persisted to:** `hathor_client.source_of_income_or_funds`, `hathor_client.purpose_of_account` (individual) — and the business analogue for `businessId` requests (covered in Set 2).

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no @NotNull individually — XOR via `isValid()`) | (no @NotNull) | `hathor_client.id` bigint | ⚠ | XOR check is logical-only. |
| 2 | `businessId` | Long | (same — XOR via `isValid()`) | (no @NotNull) | `hathor_business.id` bigint (business path) | ⚠ | |
| 3 | `sourceOfIncomeOrFunds` | String | (no @Size, no @NotBlank) | @NotNull (no @Size) | `hathor_client.source_of_income_or_funds` varchar(255) · `khonsu_prospect.source_of_income_or_funds` varchar(255) | ❌ | No length cap anywhere except DB. **Inconsistent with CreateAccountRequest.FinancialInfo.sourceOfIncomeOrFunds which is @Size(max=100)**. |
| 4 | `purposeOfAccount` | String | (no @Size, no @NotBlank) | @NotNull (no @Size) | `hathor_client.purpose_of_account` varchar(255) · `khonsu_prospect.purpose_of_account` varchar(255) | ❌ | Same as above. |

---

## 7. `POST /admin/id-search` | `POST /external/id-search` — SSN identity lookup (Signzy)

- **Controller:** `IdentityVerificationController.fetchSsnDetails`
- **Request DTO (gateway):** `PartnerSsnFetchRequest` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/identity/PartnerSsnFetchRequest.java))
- **Service flow:** `IdentityVerificationService.fetchSsnDetails` → `SsnFetchMapper.toSsnFetchRequest` (adds `identifier`=UUID, `identifierType`="PARTNER", `numberOfAddresses`="1", `numberOfEmails`="1", `consentStatus`="optedIn") → `SingzyClientManager.ssnFetch(SsnFetchRequest)` (**glide-client**, then outbound to Signzy)
- **Downstream DTO:** `com.glidetech.client.api.v1.domain.signzy.SsnFetchRequest`
- **Persisted to:** Not directly persisted by the request. Signzy response is logged in glide-client `hathor_signzy` and may populate `hathor_lexis_nexis` / `hathor_ofac` records — but those are response-side, not request-field-validation scope.

| # | Field | Type | Gateway DTO | Downstream DTO | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `phoneNumber` | String | @NotBlank @Pattern("\\d{1,15}") | (no validation) | ⚠ | Gateway enforces digits-only ≤15. Downstream relies on gateway. |
| 2 | `last4` | String | @Pattern("\\d{4}") | (no validation) | ⚠ | Gateway enforces exactly 4 digits. |
| 3 | `ssn` | String | @Pattern (US SSN excluding 000/666/9xx prefixes, etc.) | (no validation) | ⚠ | Strong gateway pattern, no downstream check. |
| 4 | `dateOfBirth` | LocalDate | @Past @JsonFormat("yyyy-MM-dd") | downstream is `String` `dob` (no validation) | ⚠ | Gateway is LocalDate; mapper formats to String for downstream. |
| — | Service-layer validation: at least one of `last4`, `ssn`, `dateOfBirth` must be present (`IDENTITY_VERIFICATION_FIELD_REQUIRED`). | | | | ✅ | |
| — | Mapper-injected fields (`identifier`, `identifierType`, `numberOfAddresses`, `numberOfEmails`, `consentStatus`) are server-controlled. | | | | ✅ | |

**No DB persistence on request side** — fields don't have a column to align to. This endpoint is well-validated at the gateway and the alignment risk is essentially zero.

---

## Async / Kafka-driven persistence

JPA `@PostPersist`/`@PostUpdate` listeners on `ClientEntity`, `SpendingAccountEntity`, `ClientAccountEntity`, and `SignzyEntity` publish events to topics `elastic_data_migration`, `account_logs`, `partner_webhook_outbound_event` (consumed by glide-search, glide-portal, glide-webhook). Event classes: `ClientProfileCreationEvent`, `SpendingAccountUpdateEvent`, `SignzyJourneyUrlCreatedEvent`, `KycDetailsReceivedEvent`, `AccountClosePendingEvent`, `FullKycCompletedEvent`, `CustomerActionEvent`, `SignzyOfacEvent`, `SignzySsnTraceEvent`.

**None of the async-written tables hold partner data in typed columns**, so no new validation alignment is required. Listed for completeness:

| Table | Columns holding partner data | Verdict |
|---|---|---|
| `hathor_signzy` | `request` jsonb · `response` jsonb (partner data inside blob); typed cols `identifier`(100), `journey_url`(300, Signzy-gen), `journey_id`(100), `status`/`step`(50–100 enums) are server-controlled | ❌ Skip |
| `hathor_ofac` | `request`/`response` text (partner data inside); typed cols `identifier`(100), `provider`(40), 11 boolean flags — server | ❌ Skip |
| `hathor_ofac_address` | Same shape as `hathor_ofac` | ❌ Skip |
| `iris_account_action_history` | `action_type`(50), `source_system`(50), `journey_id`(255) — all server-controlled enums | ❌ Skip |
| `iris_account_review` | `primary_account`(20, matches `osiris_client_account.account_id`), `review_comment` text (operator-written) | ❌ Skip |
| `demeter_webhook_outbound_event_log` | `event_payload` **text** (unbounded JSON, partner-visible contract inherited from event class — re-uses §1–§7 DTOs), `event_type`(100), `event_id`(255), `signature`(300) | ⚠ Soft — no column-width gap |
| `hathor_user_credential_update_log` | Stores old/new email/phone on `updateIndividualAccount`. Schema not audited here — assumed to mirror `hathor_client.email_id`(100) / `primary_phone`(20). | ⚠ Flag for follow-up audit (see Rec #10) |
| Elasticsearch indices (`elastic_data_migration` topic) | Not Postgres — out of scope | — |

---

## Verification

Spot-checked one full trace for each endpoint:

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /accounts` | `accountName` → service `clientAccountService.createIndividualAccount` → `ClientAccountManager.createSpendingAccount` → `osiris_client_account.account_name` (varchar 60 in live DB) | ✅ Match — confirmed truncation risk vs gateway @Size(80). |
| `POST /accounts/close` | `accountId` → service close path → `osiris_client_account.account_id` (varchar 20) | ✅ Match — gateway has no @Size; DB caps at 20. |
| `POST /accounts/create-journey-url` | `prospectId` → forwarded → links to `khonsu_prospect.identifier` (varchar 255) | ✅ Match. |
| `PUT /accounts/kyc` | `identificationNumber` → `ClientAccountManager.updateIdentification` → `osiris_spending_account.id_number_updated` (varchar 260) | ✅ Confirmed the 260 outlier. |
| `PUT /accounts/individual` | `firstName` (gateway @Size 100) → `hathor_client.first_name` (varchar 32) | ✅ Confirmed truncation risk. |
| `PUT /accounts/account-info` | `sourceOfIncomeOrFunds` → `hathor_client.source_of_income_or_funds` (varchar 255) | ✅ Confirmed unbounded gateway. |
| `POST /id-search` | (no DB persistence) — manual mapper trace verified | ✅ |

---

## Recommended next-step actions (for the consistency pass — not in scope of this doc)

1. Pick canonical lengths for `firstName`/`lastName` (20 vs 32 vs 100 — likely 50).
2. Standardise `identificationType` (40 chars) and `identificationNumber` (60 chars); drop `id_type_updated`(240)/`id_number_updated`(260) bloat in `osiris_spending_account`.
3. Standardise `sourceOfIncomeOrFunds` & `purposeOfAccount` to one length across gateway and DB (suggest 255 with gateway `@Size(max=255)`).
4. Standardise `countryCode` to ISO-3 (`@Size(max=3)`, DB varchar(3)) across all tables.
5. Tighten `accountName` at the gateway to 60 chars (matching `osiris_client_account.account_name`).
6. Add `@NotNull` + `@Size` + `@Pattern` to `CloseAccountReq`, `UpdateProspectFieldsRequest`, and `CreateSignzyJourneyRequest` per CLAUDE.md partner-DTO rules.
7. Convert `CreateAccountRequest.personalInfo.dateOfBirth` from `String` to `LocalDate` + `@Past` + `@JsonFormat("yyyy-MM-dd")`.
8. Add `@Pattern("\\d{1,15}")` to `phoneNumber` at every gateway DTO.
9. Introduce a gateway-layer `CreateJourneyUrlRequest` DTO (the current controller leaks the downstream type).
10. (async) Audit `hathor_user_credential_update_log` column widths against `hathor_client.email_id`(100) / `primary_phone`(20) to ensure the audit log can store the values the source row holds.
11. (async) Confirm the JSON contract emitted to `demeter_webhook_outbound_event_log.event_payload` — partner-facing webhook payload re-uses Set 1 DTO field shapes. If any DTO width is tightened in the consistency pass, also update the corresponding event class so the partner sees consistent caps.
