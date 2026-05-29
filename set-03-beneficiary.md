# Set 3 — Beneficiary

5 endpoints under `BeneficiaryController`. Touches **glide-remittance** (`dionysus_*` tables, plus an outbound to the card-vault for `link-card`).

**Architectural note — dynamic field model.** Unlike Set 1/2, beneficiary creation/update DTOs do **not** declare individual personal-info fields. They carry a single `Map<String, String> fields` map. The field schema (which keys are required, what regex applies, what min/max length) is driven by metadata rows in `dionysus_beneficiary_field_map` and `dionysus_validation`. Partner discovers the schema via `GET /beneficiary/fields` and `GET /beneficiary/account/fields` (out of scope here — read-only). The mapper then fans the map out into specific columns on `dionysus_beneficiary_personal_detail` and `dionysus_beneficiary_account_detail`.

**This means: at the gateway and downstream DTO layers, only the wrapper (currencyId, country, beneficiaryType, channel, referenceId, fields-not-null) is validated. Per-field validation is purely DB-driven and applied inside glide-remittance.** The "alignment" question therefore becomes: are the partner-facing labels/sizes (per `dionysus_validation`) consistent with the actual column widths in `dionysus_beneficiary_personal_detail` / `_account_detail` / `_address`?

Mythology mapping confirmed:
- `dionysus_*` → **glide-remittance** (NEW for this set; ~94 tables — biggest service in the codebase)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | `BeneficiaryCreationRequest.fields` (gateway) | `@NotNull` on the map only — **no per-field validation at gateway**. All length/format enforcement is delegated to `dionysus_validation` metadata in glide-remittance. | ⚠ Architectural — by design, but partner has no schema enforcement until the request reaches glide-remittance. Schema is publishable via `GET /beneficiary/fields`. |
| 2 | Address column widths in `dionysus_beneficiary_address` | `address_one`=varchar(50), `address_two`=varchar(50), `city`=varchar(30), `state`=varchar(25), `postal_code`=varchar(7), `country_code`=varchar(3) | ❌ **All narrower** than every other address table in the codebase (`hathor_client_address`: 80/80/128/64/10/3; `khonsu_prospect`: 80/80/128/64/10/20). Truncation risk for non-US addresses with long city/state names. |
| 3 | `dionysus_beneficiary_personal_detail.phone` = **varchar(14)** | Partner sends phone via dynamic `fields` map. Most other phone columns are varchar(20). International numbers up to 15 digits + country code may not fit. | ❌ |
| 4 | `dionysus_beneficiary_personal_detail.first_name/last_name` = varchar(50) | Mismatch with `hathor_client.first_name`=varchar(32), `khonsu_prospect.first_name`=varchar(20), `khonsu_business_prospect.first_name`=varchar(20) | ❌ Same logical "first_name" stored at 4 different lengths across the stack (20/20/32/50). |
| 5 | `dionysus_beneficiary_personal_detail.country` = **varchar(4)** | Odd: ISO alpha-2 = 2 chars, alpha-3 = 3, but the column allows 4. None of the other country columns use 4. | ❌ |
| 6 | `dionysus_beneficiary_account_detail.account_number` = varchar(30) | Vs `osiris_client_account.account_id`=varchar(20). Different sizing for "account number" across modules. | ❌ |
| 7 | `BeneficiaryCreationRequest.country` (gateway) | Plain String, no `@Size`/`@Pattern`, `dionysus_beneficiary_personal_detail.country`=varchar(4) | ⚠ |
| 8 | `BeneficiaryAccountCreationRequest.channel` (gateway) | `@NotNull` (no `@NotBlank`, no `@Size`), `dionysus_beneficiary_account_detail.channel`=varchar(30) and `dionysus_beneficiary_field_map.channel`=varchar(30) | ⚠ No length cap at gateway. Used to look up provider/channel mapping; an unrecognised value just returns no fields. |
| 9 | `BeneficiaryAccountCreationRequest.referenceId` (gateway) | `@NotNull` only. `dionysus_beneficiary_personal_detail.reference_id`=varchar(60) | ⚠ No `@Size`. |
| 10 | **Downstream `CreateBeneficiaryRequest.currencyId` has `@NotBlank` on a `Long`** | `@NotBlank` is for `CharSequence`, not numerics. Bean Validation silently ignores it (and emits a warning at startup at best). | ❌ Bug — should be `@NotNull`. |
| 11 | **Downstream `CreateBeneficiaryRequest.beneficiaryType` has `@NotBlank` on an enum (`BeneficiaryType`)** | Same misuse — `@NotBlank` only applies to strings. Field defaults to `INDIVIDUAL`, so this only matters if partner explicitly sends `null`. | ❌ Bug — should be `@NotNull`. |
| 12 | `VaultSessionRequest` (gateway side at `linkCard`) | **`@RequestBody` (no `@Valid`)** + DTO has zero validation. Both `clientId` and `businessId` are nullable Longs. | ❌ The controller signature in `BeneficiaryController.linkCard` omits `@Valid` on `@RequestBody VaultSessionRequest` — any validation would be ignored anyway since the DTO has none. Service-level `validationService.validateAccess(clientId, businessId)` handles the XOR. |
| 13 | `dionysus_beneficiary_field_map` row widths | `field_name`=varchar(40), `label`=varchar(40), `field_type`=varchar(10), `description`=varchar(50), `option_key`=varchar(20), `payment_type`=varchar(30), `validation_key`=varchar(30), `depends_on`=varchar(200) | ⚠ Schema-config table only — affects what partner can *configure*, not what they can submit. |
| 14 | `dionysus_validation.regex` = varchar(100) | Complex regex (e.g. SSN patterns) approach 70+ chars; column is tight | ⚠ |
| 15 | `dionysus_beneficiary_account_detail.bank_name` = varchar(255) but `bank_code`/`routing_code`/`bank_account_number`/`swift_code` = varchar(100) | Bank ID widths divergent | ⚠ |
| 16 | `dionysus_beneficiary_personal_detail` has DB columns `account_numbers` (text), `direction` — **not in JPA entity** | Schema drift. | ⚠ |
| 17 | Card-link fields | `dionysus_beneficiary_account_detail.card_token`=varchar(no limit) NULL, `card_last4`=varchar(4) — fine but loose | ✅/⚠ |
| 18 | **`dionysus_beneficiary_transaction_records.beneficiary_account_number` = varchar(35)** vs `dionysus_beneficiary_account_detail.account_number` = varchar(**30**) | ❌ Cross-table account-number width mismatch for the same logical concept (a beneficiary's account number). Async-written when a transaction touches a beneficiary. |
| 19 | **`dionysus_beneficiary_transaction_records.client_account_number` = varchar(20)** vs `dionysus_international_transaction.client_account_number` = varchar(30) | ❌ Same field, two widths across the transaction record tables. |
| 20 | **Async / Kafka-driven persistence** — Set 3 entities emit Kafka events that produce side-effect writes to `dionysus_beneficiary_transaction_records`, `iris_investigation_case`, `iris_investigation_history`, `iris_investigation_comments`, `iris_investigation_related_resource`, `demeter_webhook_outbound_event_log` (and `hathor_signzy`/`ofac` for KYC-linked beneficiaries). See "Async / Kafka-driven persistence" section after §5. | ❌ Two new cross-table width gaps (#18, #19) confirmed. |

---

## 1. `POST /admin/beneficiary` | `POST /external/beneficiary` — Create beneficiary (personal info)

- **Controller:** `BeneficiaryController.createBeneficiary`
- **Request DTO (gateway):** `BeneficiaryCreationRequest`
- **Service flow:** `BeneficiaryService.createBeneficiary` → builds `CreateBeneficiaryRequest` (downstream) — copies `currencyId`, `country`, `beneficiaryType`, then **flattens the `Map<String,String> fields` into `List<BeneficiaryField{fieldName,value}>`** → `RemittanceManager.createBeneficiaryPersonalInfo(CreateBeneficiaryRequest)` (**glide-remittance**)
- **Downstream DTO:** `com.glidetech.remittance.api.v1.domain.beneficiary.CreateBeneficiaryRequest`
- **Persisted to:** `dionysus_beneficiary_personal_detail` (+ `dionysus_beneficiary_address`)

### Wrapper fields

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation; XOR with `businessId` checked in service) | — (not in downstream) | (key, server-resolved) | ⚠ | |
| 2 | `businessId` | Long | (no validation; XOR) | — | (key, server-resolved) | ⚠ | |
| 3 | `beneficiaryType` | enum | @NotNull | **@NotBlank** (bug — should be @NotNull) | `dionysus_beneficiary_personal_detail.transaction_type` varchar(10) (different name) and category mapping internally | ❌ Bug | |
| 4 | `currencyId` | Long | @NotNull | **@NotBlank** (bug — should be @NotNull on Long) | `dionysus_beneficiary_personal_detail.currency_id` integer | ❌ Bug | |
| 5 | `country` | String | (no @Size, no @Pattern) | (no @Size, no @Pattern) | `dionysus_beneficiary_personal_detail.country` **varchar(4)** | ⚠ | Odd column width. |
| 6 | `fields` | Map<String,String> | @NotNull | (`data: List<BeneficiaryField>`, @NotNull) | fanned out per field | ⚠ | Per-field constraints come from `dionysus_validation`. |
| 7 | `clientIpAddress` | String | (no @Size) | — | (not persisted on beneficiary entities) | ⚠ | |

### Dynamic fields (typical keys fanned into `dionysus_beneficiary_personal_detail` columns)

These are the column widths the partner's `fields` map values must fit. Per-field regex/min/max comes from `dionysus_validation` (queried by `validation_key` from `dionysus_beneficiary_field_map`).

| Map key (typical) | DB column | Width | Comparison to other tables |
|---|---|---|---|
| `firstName` | `first_name` | varchar(50) | ⚠ Inconsistent (20 in `hathor_client`, 20 in `khonsu_prospect`, 50 here) |
| `lastName` | `last_name` | varchar(50) | ⚠ Same |
| `email` | `email` | varchar(100) | ✅ Aligned with hathor_client/khonsu_prospect |
| `phone` | `phone` | **varchar(14)** | ❌ Narrower than 20 used elsewhere |
| `businessName` | `business_name` | varchar(100) | ✅ Aligned with hathor_business.company (100) |
| `relationship` | `relationship` | varchar(50) | ✅ |
| `idName` | `id_name` | varchar(50) | ⚠ |
| `idType` | `id_type` | varchar(50) | ⚠ Wider than gateway @Size(40) for identificationType in CreateAccountRequest |
| `customOne` | `custom_one` | varchar(50) | — |
| `customTwo` | `custom_two` | varchar(50) | — |
| `documentType` | `document_type` | varchar(50) | — |
| `documentId` | `document_id` | varchar(50) | — |
| `dateOfBirth` | `date_of_birth` | date | ⚠ Map value is String, must be `yyyy-MM-dd`-parseable. |
| `addressOne` | `dionysus_beneficiary_address.address_one` | **varchar(50)** | ❌ 50 vs 80 elsewhere |
| `addressTwo` | `address_two` | **varchar(50)** | ❌ 50 vs 40/80 elsewhere |
| `city` | `city` | **varchar(30)** | ❌ 30 vs 128 elsewhere |
| `state` | `state` | **varchar(25)** | ❌ 25 vs 64 elsewhere |
| `postalCode` | `postal_code` | **varchar(7)** | ❌ 7 vs 10 elsewhere |
| `countryCode` | `country_code` | varchar(3) | ✅ matches `hathor_client_address` |

---

## 2. `PATCH /admin/beneficiary/account` | `PATCH /external/beneficiary/account` — Create beneficiary account (attach to existing beneficiary)

- **Controller:** `BeneficiaryController.createBeneficiaryAccount`
- **Request DTO (gateway):** `BeneficiaryAccountCreationRequest`
- **Service flow:** `BeneficiaryService.createBeneficiaryAccount` → builds downstream `CreateBeneficiaryAccountRequest` → `RemittanceManager.createBeneficiaryAccount(...)` (**glide-remittance**)
- **Downstream DTO:** `com.glidetech.remittance.api.v1.domain.beneficiary.CreateBeneficiaryAccountRequest`
- **Persisted to:** `dionysus_beneficiary_account_detail`

### Wrapper fields

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | (server-resolved) | ⚠ | |
| 2 | `businessId` | Long | (no validation) | — | (server-resolved) | ⚠ | |
| 3 | `beneficiaryType` | enum | @NotNull | (no @NotNull; default INDIVIDUAL) | derived | ⚠ | |
| 4 | `channel` | String | @NotNull (no @NotBlank, no @Size) | @NotBlank (no @Size) | `dionysus_beneficiary_account_detail.channel` varchar(30) · `dionysus_beneficiary_field_map.channel` varchar(30) | ❌ | Gateway is @NotNull (allows empty string); downstream is @NotBlank. No length cap; DB caps at 30. |
| 5 | `referenceId` | String | @NotNull | @NotBlank | `dionysus_beneficiary_personal_detail.reference_id` varchar(60) | ❌ | Same NotNull-vs-NotBlank inconsistency. No `@Size`. |
| 6 | `fields` | Map<String,String> | @NotNull | (`data: List<BeneficiaryField>`, @NotNull) | fanned to account_detail columns | ⚠ | |
| 7 | `clientIpAddress` | String | (no @Size) | — | (not persisted) | ⚠ | |

### Dynamic fields fanned to `dionysus_beneficiary_account_detail`

| Map key (typical) | DB column | Width |
|---|---|---|
| `bankCode` | `bank_code` | varchar(100) |
| `swiftCode` | `swift_code` | varchar(100) |
| `bankName` | `bank_name` | varchar(255) |
| `routingCode` | `routing_code` | varchar(100) |
| `bankAccountNumber` | `bank_account_number` | varchar(100) |
| `customAcOne` | `custom_ac_one` | varchar(50) |
| `customAcTwo` | `custom_ac_two` | varchar(50) |
| `cardToken` | `card_token` | varchar (unbounded) ❌ |
| `cardLast4` | `card_last4` | varchar(4) |

Note: `dionysus_beneficiary_account_detail.account_number` (varchar 30) is **server-generated** during create — not partner-supplied. So no partner-input alignment issue, but the column width is still divergent from `osiris_client_account.account_id` (varchar 20) — pick one canonical length if a future feature ever cross-references them.

---

## 3. `PUT /admin/beneficiary/{referenceId}` | `PUT /external/beneficiary/{referenceId}` — Update beneficiary (personal info)

- **Controller:** `BeneficiaryController.updateBeneficiary`
- **Request DTO (gateway):** `BeneficiaryCreationRequest` (re-used)
- **Service flow:** `BeneficiaryService.updateBeneficiary` → builds `CreateBeneficiaryRequest` → `RemittanceManager.updateBeneficiaryPersonalInfo(referenceId, ...)` (**glide-remittance**)
- **Downstream DTO:** Same as §1.
- **Persisted to:** UPDATE on `dionysus_beneficiary_personal_detail` (+ `dionysus_beneficiary_address`)

All field rows from §1 apply identically. The path-variable `referenceId` carries the identity; `dionysus_beneficiary_personal_detail.reference_id`=varchar(60).

---

## 4. `PUT /admin/beneficiary/account/{accountNumber}` — Update beneficiary account

- **Controller:** `BeneficiaryController.updateBeneficiaryAccount`
- **Request DTO (gateway):** `BeneficiaryAccountCreationRequest` (re-used)
- **Service flow:** `BeneficiaryService.updateBeneficiaryAccount` → builds `CreateBeneficiaryAccountRequest` → `RemittanceManager.updateBeneficiaryAccount(accountNumber, ...)` (**glide-remittance**)
- **Downstream DTO:** Same as §2.
- **Persisted to:** UPDATE on `dionysus_beneficiary_account_detail`

All field rows from §2 apply. The path-variable `accountNumber` carries identity; `dionysus_beneficiary_account_detail.account_number`=varchar(30).

**Note:** the second path declaration in the controller has a **leading space** (`" external/beneficiary/account/{accountNumber}"` instead of `"external/..."`). That mapping likely never matches a real request. ❌

---

## 5. `POST /admin/beneficiary/{referenceId}/link-card` — Create vault session token for card linking

- **Controller:** `BeneficiaryController.linkCard`
- **Request DTO (gateway):** `com.glidetech.api_gateway.api.v1.domain.card.VaultSessionRequest`
- **Service flow:** `BeneficiaryService.linkCard` → service-layer `validationService.validateAccess(clientId, businessId)` → `RemittanceManager.linkCard(referenceId, com.glidetech.remittance.api.v1.domain.card.VaultSessionRequest)` (**glide-remittance**, which proxies to the configured card-vault provider — Tabapay/Cybersource etc.)
- **Downstream DTO:** `com.glidetech.remittance.api.v1.domain.card.VaultSessionRequest` (functionally identical: `{clientId, businessId}`)
- **Persisted to:** No DB write at this step — session token is held by the vault provider until the SDK uses it. A subsequent card registration creates entries in `dionysus_beneficiary_account_detail` (`card_token`, `card_last4`).

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | (no validation) | (server-resolved, not persisted here) | ❌ | XOR enforced only by `validateAccess`. |
| 2 | `businessId` | Long | (no validation) | (no validation) | (server-resolved) | ❌ | Same. |

**Gap:** the controller method signature is `public VaultSessionResponse linkCard(@PathVariable String referenceId, @RequestBody VaultSessionRequest request)` — **no `@Valid`**. Even if the DTO had validation, it wouldn't fire.

Also: there's no `@PostMapping({"admin/...", "external/..."})` — `linkCard` is admin-only despite being part of the otherwise external-capable BeneficiaryController. ⚠ (Documented behavior — not a defect unless partner-facing access was intended.)

---

## Async / Kafka-driven persistence

`BeneficiaryAccountDetailEntityListener` and `BeneficiaryPersonalDetailEntityListener` publish on every persist. Topics: `elastic_data_migration`, `partner_webhook_outbound_event`, `transaction_investigation_event` (on OFAC fail). Event classes: `BeneficiaryAccountDetailEvent`, `BeneficiaryPersonalDetailEvent`, `BeneficiaryWebhookEvent`, `BeneficiaryOfacEvent`. The only async-written table with typed columns receiving partner data is `dionysus_beneficiary_transaction_records` (Headline #18–#19).

### `dionysus_beneficiary_transaction_records` — written async when a transaction references a beneficiary (full schema)

| # | Column | Type | Source | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `id` | bigint | server | ✅ | PK |
| 2 | `payment_identifier` | varchar(60) NOT NULL | `referenceId` (links to `dionysus_beneficiary_personal_detail.reference_id` varchar 60) | ✅ | |
| 3 | `client_account_number` | varchar(**20**) NOT NULL | partner `fromAccount` | ❌ | Vs `dionysus_international_transaction.client_account_number` varchar(30) — same field, two widths. |
| 4 | `beneficiary_account_number` | varchar(**35**) NOT NULL | partner `beneficiaryAccount` | ❌ | Vs `dionysus_beneficiary_account_detail.account_number` varchar(30) — same field, two widths. |
| 5 | `to_currency_id` | integer NOT NULL | partner `currencyId` | ✅ | |
| 6 | `data` | jsonb NOT NULL | full beneficiary payload (server-snapshotted) | ✅ | |
| 7 | `channel` | varchar(50) | partner `channel` (`BeneficiaryAccountCreationRequest`) | ✅ | |
| 8 | `created_on`, `updated_on` | timestamp | server | ✅ | |

### Other async-touched tables (no validation alignment needed)

| Table | Why no gap | Verdict |
|---|---|---|
| `iris_investigation_case` | Opened on `BeneficiaryOfacEvent`. Typed cols: `case_id`(80), `title`(250), `description`(500, server-templated, may include partner names), `status`/`priority`(20), `investigation_type`(30), `source_module`(60), `tags`(100), `resolution_summary`(500), `reference_id`(100), `correlation_id`(100), `name`(255). Partner data appears only inside templated description. | ⚠ Soft — smoke-test recommended |
| `iris_investigation_history` | `case_id`(80), `action_type`(20), `description`(500), `old_value`/`new_value` jsonb | ⚠ Same as above |
| `iris_investigation_comments` | `comment`(500) written by ops, not partner | ❌ Skip |
| `iris_investigation_related_resource` | `resource_type`(20), `resource_id`(120) server-generated | ❌ Skip |
| `demeter_webhook_outbound_event_log` | `event_payload` text blob holds `BeneficiaryWebhookEvent`. Partner-visible JSON contract inherited from event class. | ⚠ Soft |
| `hathor_signzy`, `hathor_ofac`, `hathor_ofac_address` | jsonb/text blobs (same shape as Sets 1–2) | ❌ Skip |
| `iris_account_review` | Operator-written `review_comment` text | ❌ Skip |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /beneficiary` | `fields.phone` → service maps to `BeneficiaryField{fieldName: "phone"}` → `dionysus_beneficiary_personal_detail.phone` (varchar 14 in live DB) | ✅ Confirmed truncation risk for international numbers. |
| `POST /beneficiary` | `country` → `dionysus_beneficiary_personal_detail.country` varchar(4) | ✅ Confirmed odd column width. |
| `PATCH /beneficiary/account` | `channel` → `dionysus_beneficiary_account_detail.channel` varchar(30) (no @Size at gateway/downstream) | ✅ Confirmed gap. |
| `PUT /beneficiary/{referenceId}` | Same DTO as create → confirmed identical handling. | ✅ |
| `PUT /beneficiary/account/{accountNumber}` | Confirmed leading-space typo in controller path. | ✅ |
| `POST /beneficiary/{referenceId}/link-card` | `clientId`/`businessId` XOR check is service-level only, no `@Valid`. | ✅ |
| `dionysus_beneficiary_personal_detail` schema drift | `account_numbers` text + `direction` varchar(20) present in DB, missing on entity. | ✅ Confirmed drift. |
| (async) `dionysus_beneficiary_transaction_records.beneficiary_account_number` varchar(35) vs `dionysus_beneficiary_account_detail.account_number` varchar(30) | ✅ Confirmed — same logical field, two widths. |
| (async) `iris_investigation_case.description` varchar(500) | ✅ Server-formatted — partner data may appear in templated text, low truncation risk for typical names/addresses but worth a smoke test. |

---

## Recommended next-step actions (for the consistency pass — not in scope of this doc)

1. Fix downstream `CreateBeneficiaryRequest` bugs: `@NotBlank` → `@NotNull` on `currencyId` (Long) and `beneficiaryType` (enum). Bean Validation silently ignores both today.
2. Add `@NotBlank` (not `@NotNull`) and `@Size` to all string wrapper fields in `BeneficiaryCreationRequest` and `BeneficiaryAccountCreationRequest`: `country` (@Size max=4 or =3), `channel` (@Size max=30), `referenceId` (@Size max=60), `clientIpAddress` (@Size max=45).
3. Add `@Valid` to the `@RequestBody VaultSessionRequest request` parameter in `BeneficiaryController.linkCard`.
4. Add XOR `clientId`/`businessId` validation as a class-level constraint on `VaultSessionRequest`, `BeneficiaryCreationRequest`, `BeneficiaryAccountCreationRequest` (matches the manual `validateAccess` check).
5. Decide a canonical length for `first_name`/`last_name` columns and apply across `dionysus_beneficiary_personal_detail` (50), `hathor_client` (32), `khonsu_prospect` (20), `khonsu_business_prospect` (20). 50 is probably the right target (international names).
6. Widen `dionysus_beneficiary_address` columns (`address_one`/`address_two`/`city`/`state`/`postal_code`) to match the canonical sizes used in `hathor_client_address` (80/80/128/64/10). Currently the narrowest table in the codebase.
7. Widen `dionysus_beneficiary_personal_detail.phone` from 14 to 20 to accommodate full international numbers + separators.
8. Decide canonical width for `country` columns — `dionysus_beneficiary_personal_detail.country` (varchar 4) is the only 4-char outlier; pick ISO-3 (3) and align.
9. Surface partner-facing per-field constraints (regex/min/max) consistently — make sure every row in `dionysus_validation` is wired up via `dionysus_beneficiary_field_map.validation_key` for every active field.
10. Fix the leading-space path bug in `BeneficiaryController.updateBeneficiaryAccount` (`" external/beneficiary/account/{accountNumber}"` → `"external/..."`).
11. Pull `dionysus_beneficiary_personal_detail.account_numbers` (text) and `direction` (varchar 20) into `BeneficiaryPersonalDetailEntity` or remove via migration — schema drift.
12. Cap `dionysus_beneficiary_account_detail.card_token` (currently unbounded varchar) to something sensible like varchar(255).
13. Reconcile `dionysus_beneficiary_account_detail.account_number` (varchar 30) with `osiris_client_account.account_id` (varchar 20) — pick one.
14. Add `clientIpAddress` storage on beneficiary entities (currently parameter is accepted but never persisted) — or drop it from the DTOs.
15. **(async, cross-table)** Reconcile `beneficiary_account_number` width across `dionysus_beneficiary_account_detail.account_number` (30), `dionysus_international_transaction.beneficiary_account_number` (35), `dionysus_beneficiary_transaction_records.beneficiary_account_number` (35) — pick one canonical width.
16. **(async, cross-table)** Reconcile `client_account_number` / `from_account` width across `dionysus_international_transaction.client_account_number` (30), `dionysus_beneficiary_transaction_records.client_account_number` (20), `osiris_client_account.account_id` (20). Same logical field at two widths.
17. (async) Spot-check `iris_investigation_case.description` (varchar 500) under realistic OFAC-failure scenarios — if business names exceed 100 chars and a templated description includes them verbatim, the audit text may truncate. Either widen to text (unbounded) or shorten the template.
18. (async) Confirm the JSON payload contract on `demeter_webhook_outbound_event_log.event_payload` for `BeneficiaryWebhookEvent` — partner-visible shape inherits from this event class. If beneficiary DTO widths change in the consistency pass, update the event class too.
