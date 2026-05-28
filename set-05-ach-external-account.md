# Set 5 — ACH & External Account

8 endpoints under `ACHController`. Covers ACH money movement (ODFI credit/debit + internal transfer) and the full external-account lifecycle (Plaid link, manual link, update, micro-deposit verify, settle, approve).

Downstream surfaces:
- **glide-extpaymt** — `zeus_ach_details` (ACH transactions)
- **glide-client-account** / externalaccount module — `osiris_external_account` (linked external bank accounts)
- **glide-business** — internal-transfer entry (`businessManager.createPayment`); persisted on the business side (not detailed here — separate set if needed)

Mythology mapping confirmed:
- `zeus_*` → **glide-extpaymt** (NEW for this set)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`ExternalAccountRequest` (gateway) has ZERO validation** | No `@NotNull`, `@NotBlank`, `@Size` on any of 8 fields. Controller `createExternalAccount` also **missing `@Valid`** on `@RequestBody`. | ❌ |
| 2 | **`UpdateExternalAccountRequest` (gateway) has ZERO validation** | All 8 fields are unannotated. | ❌ |
| 3 | **`LinkExternalAccountRequest` (gateway)** | Only `accountSubType` has `@NotBlank`. The other 12 fields (bank account number, routing number, public token, mask, etc.) have **no validation at all**. | ❌ |
| 4 | `osiris_external_account.account_no` = **varchar(35) NOT NULL** but downstream `UpdateExternalAccount.accountNo` = `@ValidIdentifier(maxLength=20)` | DB allows 35 chars, validation only 20. Direct create-path (via `PlaidExternalAccount`) does not validate `accountNo` at all (the field lives on the nested `ExternalAccount` DTO). | ❌ |
| 5 | `osiris_external_account.access_token` and `.account_subtype` are **unbounded varchar** (no length set) | Hibernate default — partner-supplied access tokens are not capped. | ❌ |
| 6 | `osiris_external_account.merchant_category_code` = varchar(60) vs downstream `@Length(max=30)` | DB allows twice the validated length. | ❌ |
| 7 | `zeus_ach_details.company_entry_description` = **varchar(50)** but gateway/downstream `@Size(max=10)` (NACHA spec) | DB is 5× the spec maximum. ⚠ Not a correctness bug (validation catches it), but column is needlessly wide. |
| 8 | `zeus_ach_details` has **many varchar(255) columns from Hibernate default** | `ach_message`, `transaction_gid`, `ach_type`, `external_account_id`, `transaction_type`, `currency_code`, `created_by`, `fed_trace_number`, `provider_status`, `sender_account`, `end_to_end_id`, `customer_reference` — all `varchar(255)` because the entity has no `length=...` and no migration set one. | ⚠ Schema sprawl. |
| 9 | **`InternalTransferRequest.amount` `@NotNull @Positive`** at gateway, but **downstream `BusinessInternalTransferRequest.amount` only `@NotNull`** (no `@Positive`) | Downstream is weaker. If gateway is bypassed, negative amounts could flow. | ❌ |
| 10 | `ODFIPaymentRequest` extends `PaymentRequest` (parent) | Parent has `@Size(10)` on `companyEntryDescription`, `@Pattern` on `settlementPriority`, `@NotNull @Positive` on `amount`, `@NotNull` on `fromAccount`/`toAccount`. Reasonable validation. | ✅ Best-validated DTO in Set 5. |
| 11 | `ODFIPaymentRequest`: missing `@Size` on `description`, `correlationId`, `reference`, `clientIpAddress` | DB `description` = text (unbounded), `correlation_id` = varchar(50), `traceNumber` = varchar(50) | ⚠ |
| 12 | **`ExtAccountVerification` (gateway) `amountOne`/`amountTwo` `@NotNull` only** | Missing `@Positive` (downstream `ExternalAccountVerification` has it ✅) and **no `@Max(99)`** — micro-deposits are cents (< $1.00 = 99 cents). | ❌ |
| 13 | `osiris_external_account.amount_one`/`amount_two` = **smallint** (16-bit, max 32767) | Gateway DTO uses `Integer`. Smallint can technically overflow on Java Integer assignment but micro-deposits are tiny — safe in practice. | ⚠ Type drift (Integer → smallint). |
| 14 | **`SettleTransactionRequest` (gateway)** only `referenceId` is `@NotNull` (not `@NotBlank`, no `@Size`) | Same gap as `BeneficiaryAccountCreationRequest.referenceId` in Set 3. | ❌ |
| 15 | `LinkExternalAccountRequest.plaidAuthResponse` and `ExternalAccountRequest`/`UpdateExternalAccountRequest.plaidAuthResponse` are typed `Object` | No structural validation. Stored as `text` in `osiris_external_account.plaid_auth_response`. | ⚠ |
| 16 | `osiris_external_account.external_acc_id` = varchar(35) NOT NULL — system-generated | Wider than the `osiris_client_account.account_id`=varchar(20) but consistent with `dionysus_international_transaction.beneficiary_account_number`=varchar(35). | ⚠ |
| 17 | `LinkExternalAccountRequest` and `ExternalAccountRequest` share most fields but have different names (`accountNo` alias on `bankAccountNumber`) and **different default behaviors** in `ClientAccountService` (one defaults `mask` to "0000", the other defaults `merchantCategoryCode` to "12343") | Confusing. Two paths to the same outcome. | ⚠ |
| 18 | `osiris_external_account.routing_number` = varchar(15) NOT NULL | Downstream `UpdateExternalAccount.routingNo` = `@ValidIdentifier(maxLength=15)`. Gateway `ExternalAccountRequest.routingNo` and `LinkExternalAccountRequest.routingNo` have no validation. US ABA routing is 9 digits. | ⚠ Aligned width-wise but missing `@Pattern("\\d{9}")` at every layer. |
| 19 | `ExternalAccountRequest.accountType` is enum `ExternalAccountType` at gateway, but server-side code overrides it to `ExternalAccountType.ACH` regardless (`plaidExternalAccount.getExternalAccount().setAccountType(ExternalAccountType.ACH);` in `ClientAccountService`) | Field is effectively read-only from partner perspective. | ⚠ |
| 20 | `ACHController` is admin-only (`@RequestMapping("admin")`) — **no `/external` variants** | Per the spec this is intentional (ACH is internal-bank-rail and exposed via `RemittanceController.transferFund` for the partner-facing transfer). | — |

---

## 1. `POST /admin/ach/move-fund` — ACH ODFI fund movement

- **Controller:** `ACHController.moveFund`
- **Request DTO (gateway):** `ODFIPaymentRequest extends PaymentRequest`
- **Service flow:** `TransactionService.achMoveFund` → validates clientId/businessId access + from/to account ownership + account approval → `TransactionMapper.toMap(request)` → `ODFIPaymentManager.moveFunds(ODFIPaymentRequest)` (**glide-extpaymt**)
- **Downstream DTO:** `com.glidetech.extpaymt.api.v1.odfi.ODFIPaymentRequest`
- **Persisted to:** `zeus_ach_details`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | (no validation) | `client_id` bigint | ⚠ | XOR validated by service. |
| 2 | `businessId` | Long | (no validation) | (no validation) | `business_id` bigint | ⚠ | |
| 3 | `fromAccount` | String | @NotNull (no @Size) | @NotNull (no @Size) | `primary_account` varchar(20) | ⚠ | Service validates account ownership. |
| 4 | `toAccount` | String | @NotNull (no @Size) | @NotNull (no @Size) | (recipient — server-resolved) | ⚠ | |
| 5 | `amount` | BigDecimal | @NotNull @Positive | @NotNull | `amount` numeric(19,2) | ⚠ | Downstream missing `@Positive`. |
| 6 | `companyEntryDescription` | String | @Size(max=10) | @Size(max=10) | `company_entry_description` **varchar(50)** | ⚠ | DB is 5× wider than the NACHA-enforced validation max. |
| 7 | `description` | String | (no validation) | (no validation) | `description` text | ⚠ | |
| 8 | `correlationId` | String | (no validation) | (no validation) | `correlation_id` varchar(50) | ❌ | No length cap. |
| 9 | `reference` | String | (no validation) | — (no `reference` field downstream) | — | ⚠ | Dropped at mapper. |
| 10 | `clientIpAddress` | String | (no validation) | — | (not persisted on `zeus_ach_details`) | ⚠ | |
| 11 | `instantPayment` | Boolean | (no validation) | (no validation) | `instant_payment` bool | ✅ | |
| 12 | `settlementPriority` | String | @Pattern("^(STANDARD\|SAME_DAY)$") | @Pattern("^(STANDARD\|SAME_DAY)$") | `settlement_priority` varchar(50) | ✅ | Gateway and downstream both validate. |
| 13 | `traceId` | String | (not on gateway DTO) | (no validation) | `trace_id` varchar(100) | — | Server-resolved. |
| 14 | `momo` | String | (not on gateway DTO) | (no validation) | (not directly persisted) | — | |

---

## 2. `POST /admin/internal-transfer` — Internal (book) transfer between accounts

- **Controller:** `ACHController.internalTransfer`
- **Request DTO (gateway):** `InternalTransferRequest`
- **Service flow:** `TransactionService.internalTransfer` → service-level validation (access + both account ownership + isInternalBusinessAccount + accountApproved) → `TransactionMapper.toInternalPaymentRequest` → `BusinessManager.createPayment(BusinessInternalTransferRequest, businessId)` (**glide-business**)
- **Downstream DTO:** `com.glidetech.business.api.v1.domain.schedule.BusinessInternalTransferRequest`
- **Persisted to:** persisted on the business side as a transaction row (`hathor_business_transaction` / similar — not part of this file's scope).

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `businessId` | Long | (no validation) | — | (key) | ⚠ | |
| 2 | `fromAccount` | String | @NotNull (no @Size) | @NotNull (no @Size) | (business-side) | ⚠ | |
| 3 | `toAccount` | String | @NotNull (no @Size) | @NotNull (no @Size) | (business-side) | ⚠ | |
| 4 | `amount` | BigDecimal | @NotNull @Positive | @NotNull (no @Positive) | (business-side) | ❌ | Downstream weaker. |
| 5 | `description` | String | (no validation) | (no validation) | (business-side) | ⚠ | |
| 6 | `correlationId` | String | (no validation) | (not in downstream) | (api-gateway echo) | ⚠ | |
| 7 | `reference` | String | (no validation) | (no validation) | (business-side) | ⚠ | |
| 8 | `clientIpAddress` | String | (no validation) | (not in downstream) | — | ⚠ | |
| 9 | `type` (downstream-only) | String | — | @Pattern("^(INTERNAL\|CARD\|ACH\|WIRE)$") default INTERNAL | (business-side) | ✅ | |

---

## 3. `POST /admin/ach/link/external-account/plaid` — Link external account via Plaid

- **Controller:** `ACHController.createPlaidExternalAccount`
- **Request DTO (gateway):** `LinkExternalAccountRequest`
- **Service flow:** `BeneficiaryService.createPlaidExternalAccount` → checks (`publicToken` OR `plaidAuthResponse` required; if no public token then `bankAccountNumber` + `routingNo` required) → `ClientAccountService.createExternalAccount(LinkExternalAccountRequest)` → `ExternalAccountMapper.toMap` → `ClientAccountManager.createExternalAccount(PlaidExternalAccount)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.externalaccount.api.v1.domain.plaid.PlaidExternalAccount` (wraps a nested `ExternalAccount`)
- **Persisted to:** `osiris_external_account`

| # | Field | Type | Gateway DTO | Downstream DTO (`PlaidExternalAccount`) | DB column (`osiris_external_account`) | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | (not on DTO; set server-side via context) | `client_id` bigint | ⚠ | |
| 2 | `businessId` | Long | (no validation) | (not on DTO; set server-side) | `business_id` bigint | ⚠ | |
| 3 | `bankName` | String | (no validation) | @Length(max=60) | `bank_name` varchar(60) | ⚠ | Gateway has no cap; downstream + DB aligned at 60. |
| 4 | `accountAlias` | String | (no validation) | (no validation on this DTO) | `account_name` varchar(60) | ⚠ | |
| 5 | `routingNo` | String | (no validation, no @Pattern) | (no validation on PlaidExternalAccount; on nested ExternalAccount) | `routing_number` varchar(15) NOT NULL | ❌ | Should be `@Pattern("\\d{9}")` (US ABA). |
| 6 | `mask` | String | (no validation) | @ValidIdentifier(maxLength=4) | (last4-of-account; not in `osiris_external_account` directly) | ⚠ | Service overrides `mask` to "0000" if blank. |
| 7 | `accountSubType` | String | @NotBlank | (no validation) | `account_subtype` varchar (**unbounded**) | ⚠ | Service defaults to "checking" if blank — yet DTO declares `@NotBlank`. Inconsistent. |
| 8 | `accountType` | String | (no validation) | (set server-side, forced to ACH) | `account_type` varchar(60) | ⚠ | |
| 9 | `clientIpAddress` | String | (no validation) | (not persisted) | — | ⚠ | |
| 10 | `isTokenize` | Boolean | (no validation; default false) | (not on this DTO) | `tokenize` bool | ⚠ | |
| 11 | `bankAccountNumber` | String | (no validation, no @Size) | (on nested ExternalAccount, no validation visible from local repo) | `account_no` **varchar(35) NOT NULL** | ❌ | DB allows 35 chars; the only length cap (`@ValidIdentifier(maxLength=20)`) lives on `UpdateExternalAccount.accountNo`. Create-path has no cap. |
| 12 | `publicToken` | String | (no validation) | (no validation) | (not directly persisted; consumed to fetch account details from Plaid) | ⚠ | Service-level XOR check between `publicToken` and `plaidAuthResponse` (one must be present). |
| 13 | `plaidAuthResponse` | Object | (no validation) | (no validation) | `plaid_auth_response` text | ⚠ | Structural validation absent. |

---

## 4. `POST /admin/ach/link/external-account` — Link external account manually

- **Controller:** `ACHController.createExternalAccount`
- **Request DTO (gateway):** `ExternalAccountRequest`
- **Service flow:** `BeneficiaryService.createExternalAccount` → `ClientAccountService.createExternalAccount(ExternalAccountRequest)` → `ExternalAccountMapper.toMap` (sets server-side defaults: `mask="0000"`, `merchantCategoryCode="12343"`, `accountSubType="checking"`, `accountType=ACH`, `isVerified=false`) → `ClientAccountManager.createExternalAccount(PlaidExternalAccount)` (**glide-client-account**)
- **Downstream DTO:** `PlaidExternalAccount`
- **Persisted to:** `osiris_external_account`
- **⚠ Controller is missing `@Valid`** on `@RequestBody ExternalAccountRequest request`. Even if the DTO had validation it would not fire.

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | `client_id` | ❌ | XOR with `businessId` not validated anywhere. |
| 2 | `businessId` | Long | (no validation) | — | `business_id` | ❌ | |
| 3 | `bankAccountNumber` (alias `accountNo`) | String | (no validation, no @Size) | (no validation on create-path) | `account_no` varchar(35) NOT NULL | ❌ | No length cap at gateway. |
| 4 | `accountAlias` | String | (no validation) | (no validation) | `account_name` varchar(60) | ❌ | |
| 5 | `routingNo` | String | (no validation, no @Pattern) | (no validation) | `routing_number` varchar(15) NOT NULL | ❌ | |
| 6 | `bankName` | String | (no validation) | @Length(max=60) | `bank_name` varchar(60) | ⚠ | Gateway missing cap. |
| 7 | `accountType` | enum `ExternalAccountType` | (no validation) | (server-overrides to ACH) | `account_type` varchar(60) | ⚠ | Effectively read-only. |
| 8 | `accountSubType` | String | (no validation) | (no validation) | `account_subtype` varchar (**unbounded**) | ❌ | Gateway-side @NotBlank exists on the Plaid variant but not here. |
| 9 | `clientIpAddress` | String | (no validation) | — | (not persisted) | ⚠ | |

---

## 5. `PATCH /admin/ach/link/external-account/{externalAccountId}` — Update external account

- **Controller:** `ACHController.updateExternalAccount`
- **Request DTO (gateway):** `UpdateExternalAccountRequest`
- **Service flow:** `BeneficiaryService.updateExternalAccount` → manually builds `UpdateExternalAccount` → `ClientAccountService.updateExternalAccount` → `ClientAccountManager.updateExternalAccount(externalAccountId, UpdateExternalAccount)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.externalaccount.api.v1.domain.UpdateExternalAccount`
- **Persisted to:** `osiris_external_account` UPDATE

| # | Field | Type | Gateway DTO | Downstream DTO (`UpdateExternalAccount`) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | — | ⚠ | |
| 2 | `businessId` | Long | (no validation) | — | — | ⚠ | |
| 3 | `externalAccountId` | String | (no validation; set from path var by controller) | (no validation) | `external_acc_id` varchar(35) NOT NULL | ⚠ | |
| 4 | `bankAccountNumber` (alias `accountNo`) | String | (no validation) | @ValidIdentifier(maxLength=20) | `account_no` **varchar(35)** NOT NULL | ❌ | Validation max=20, DB allows 35. |
| 5 | `routingNo` | String | (no validation) | @ValidIdentifier(maxLength=15) | `routing_number` varchar(15) NOT NULL | ⚠ | Aligned downstream + DB; gateway missing @Pattern. |
| 6 | `accountAlias` | String | (no validation) | mapped to `accountName` @Length(max=60) | `account_name` varchar(60) NOT NULL | ⚠ | Gateway missing cap. |
| 7 | `bankName` | String | (no validation) | @Length(max=60) | `bank_name` varchar(60) | ⚠ | |
| 8 | `accountSubType` | String | (no validation) | (no validation) | `account_subtype` varchar (unbounded) | ❌ | |
| 9 | `plaidAuthResponse` | Object | (no validation) | (no validation) | `plaid_auth_response` text | ⚠ | |

`UpdateExternalAccount` also carries `merchantCategoryCode` @Length(max=30) (server-set, not exposed in gateway DTO). DB column allows 60 — mismatch.

---

## 6. `POST /admin/ach/link/external-account/verify` — Verify micro-deposit amounts

- **Controller:** `ACHController.verifyExternalAccount`
- **Request DTO (gateway):** `ExtAccountVerification`
- **Service flow:** `BeneficiaryService.verifyExternalAccount` → `ClientAccountService.verifyExternalAccount` → `BeneficiaryMapper.toMap` → `ClientAccountManager.verifyExternalAccount(ExternalAccountVerification)` (**glide-client-account**)
- **Downstream DTO:** `com.glidetech.externalaccount.api.v1.domain.ExternalAccountVerification`
- **Persisted to:** UPDATE on `osiris_external_account` (`is_verified=true` when amounts match)

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `externalAccountId` | String | @NotBlank | @NotBlank | `external_acc_id` varchar(35) NOT NULL | ⚠ | No @Size at either layer. |
| 2 | `amountOne` | Integer | @NotNull | @NotNull @Positive | `amount_one` smallint | ❌ | Gateway missing @Positive. **No @Max(99)** — micro-deposits are sub-dollar (cents). |
| 3 | `amountTwo` | Integer | @NotNull | @NotNull @Positive | `amount_two` smallint | ❌ | Same. |
| 4 | `clientIpAddress` | String | (no validation) | (not in downstream) | — | ⚠ | |

---

## 7. `POST /admin/ach/settle-transaction` — Settle ACH ODFI credit

- **Controller:** `ACHController.settleTransaction`
- **Request DTO (gateway):** `SettleTransactionRequest`
- **Service flow:** `TransactionService.settleAchTransaction` → `ODFIPaymentManager.settle(...)` (**glide-extpaymt**)
- **Downstream DTO:** (resolves to a settle-by-reference call — settlement DTO in glide-extpaymt; primarily server-side)
- **Persisted to:** UPDATE on `zeus_ach_details` (status transition + `settlement_request_timestamp`)

| # | Field | Type | Gateway DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 3 | `referenceId` | String | @NotNull (no @NotBlank, no @Size) | matched against `zeus_ach_details.correlation_id` varchar(50) or `transaction_id` varchar(60) | ❌ | Empty string would pass. No `@Size` enforcement. |
| 4 | `clientIpAddress` | String | (no validation) | (not persisted) | ⚠ | |
| 5 | `correlationId` | String | (no validation) | `correlation_id` varchar(50) | ❌ | |

---

## 8. `POST /admin/ach/approve-transaction` — Approve ACH ODFI transaction

- **Controller:** `ACHController.approveTransaction`
- **Request DTO (gateway):** `SettleTransactionRequest` (re-used)
- **Service flow:** `TransactionService.approveAchTransaction` → `ODFIPaymentManager.approve(...)` (**glide-extpaymt**)
- **Persisted to:** UPDATE on `zeus_ach_details` (status transition)

Same field-by-field as §7. Identical DTO and validation gaps.

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /ach/move-fund` | `companyEntryDescription` `@Size(max=10)` → `zeus_ach_details.company_entry_description` varchar(50) | ✅ Confirmed DB-vs-validation gap. |
| `POST /internal-transfer` | `amount` `@Positive` at gateway only — downstream `BusinessInternalTransferRequest.amount` has no @Positive | ✅ Confirmed downstream-weaker. |
| `POST /ach/link/external-account/plaid` | `accountSubType` `@NotBlank` at gateway but service falls back to "checking" when blank | ✅ Confirmed contradictory. |
| `POST /ach/link/external-account` | Controller method **missing `@Valid`** on `@RequestBody` | ✅ Confirmed. |
| `PATCH /ach/link/external-account/{id}` | Downstream `UpdateExternalAccount.accountNo` `@ValidIdentifier(maxLength=20)` vs `osiris_external_account.account_no` varchar(35) | ✅ Confirmed mismatch. |
| `POST /ach/link/external-account/verify` | `amountOne`/`amountTwo` have no `@Max` — micro-deposits should cap at 99 cents | ✅ Confirmed. |
| `POST /ach/settle-transaction` | `referenceId` `@NotNull` only — empty string would pass | ✅ Confirmed. |
| `osiris_external_account` | `access_token` and `account_subtype` columns have no length set (unbounded varchar) | ✅ Confirmed schema sprawl. |
| `zeus_ach_details` | 12+ columns are unbounded `varchar(255)` due to default Hibernate behaviour | ✅ Confirmed. |

---

## Recommended next-step actions (for the consistency pass — not in scope of this doc)

1. **Critical:** Add `@Valid` to `ACHController.createExternalAccount`'s `@RequestBody ExternalAccountRequest request`.
2. Add `@NotBlank` + `@Pattern("\\d{9}")` + `@Size(max=15)` to `routingNo` on `LinkExternalAccountRequest`, `ExternalAccountRequest`, `UpdateExternalAccountRequest` (US ABA = 9 digits).
3. Add `@NotBlank` + `@Size(max=20)` (or 35 — pick one canonical) to `bankAccountNumber` / `accountNo` on all three external-account gateway DTOs. Reconcile with downstream `@ValidIdentifier(maxLength=20)` and DB `account_no` varchar(35) — pick a single canonical width.
4. Add `@NotNull` XOR constraint on `clientId` / `businessId` to all three external-account DTOs.
5. Resolve `accountSubType` contradiction: either remove `@NotBlank` from `LinkExternalAccountRequest` (service falls back to "checking") or stop the service-side fallback.
6. Add `@Size(max=...)` to `osiris_external_account.access_token` and `.account_subtype` columns via migration (currently unbounded).
7. Add `@Positive` and `@Max(99)` to `ExtAccountVerification.amountOne`/`amountTwo` at gateway. Add `@Max(99)` downstream too — these are sub-dollar micro-deposits.
8. Add `@NotBlank` + `@Size(max=60)` to `SettleTransactionRequest.referenceId` (currently `@NotNull` only).
9. Add `@Positive` to `BusinessInternalTransferRequest.amount` (downstream is weaker than gateway).
10. Add `@Size(max=50)` to `correlationId` everywhere (matches `zeus_ach_details.correlation_id` and `dionysus_international_transaction.correlation_id` — both 50, except `CreatePaymentRequest.correlationId` is 80; pick one).
11. Narrow `zeus_ach_details.company_entry_description` from varchar(50) to varchar(10) per NACHA spec.
12. Set explicit `@Column(length=...)` on the many `zeus_ach_details` fields that currently default to varchar(255): `ach_message`, `transaction_gid`, `ach_type`, `external_account_id`, `transaction_type`, `currency_code`, `created_by`, `fed_trace_number`, `provider_status`, `sender_account`, `end_to_end_id`, `customer_reference`. Then run a Flyway migration to align the DB.
13. Reconcile `osiris_external_account.merchant_category_code` varchar(60) with downstream `@Length(max=30)`. Suggest tightening DB to 30.
14. Consider whether `LinkExternalAccountRequest` and `ExternalAccountRequest` can be merged into one DTO — they cover overlapping flows with subtly different defaults.
15. Add `@Pattern` for US bank-account-number format (variable but ≤17 digits per ABA) where applicable.
