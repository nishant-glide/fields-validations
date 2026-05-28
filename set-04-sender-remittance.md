# Set 4 — Sender & Remittance Initiation

5 endpoints across `SenderController` + `RemittanceController`. The actual money-movement entry points for international payments and the FX pricing override.

Downstream surfaces:
- **glide-remittance** — `dionysus_partner_sender_details`, `dionysus_international_transaction`, `dionysus_fx_pricing_base_rates`
- For `POST /payments/transfer-fund` — branches by `paymentType`: WIRE → `transaction-api` artifact (🔒 deferred), ACH → **glide-extpaymt** (will be detailed in Set 5; here we cover only the gateway DTO)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`SenderDetailsDTO` (gateway) has ZERO Bean Validation annotations** | No `@NotNull`, `@NotBlank`, `@Size`, `@Pattern`, `@Email`, `@Past` anywhere across 20 fields. `@Valid` on the controller has nothing to validate. | ❌ Worst-validated DTO encountered so far. Service-layer manually checks XOR (`clientId`/`businessId`) and `firstName`/`lastName` per `type`. |
| 2 | `dionysus_partner_sender_details` column widths | `first_name`/`last_name`/`business_name`/`city`/`state`/`document_id` = **varchar(150)** — **far wider** than the equivalent columns in every other table (`hathor_client.first_name`=32, `khonsu_prospect.first_name`=20, `dionysus_beneficiary_personal_detail.first_name`=50). | ❌ Storage bloat + zero alignment across modules. |
| 3 | **`PartnerSenderDetailsEntity.dateOfBirth` is `LocalDateTime`** in the entity, but the DB column is **`date`** | Type mismatch — Hibernate coerces, time portion discarded. Gateway DTO is correctly `LocalDate`. | ❌ |
| 4 | **`PartnerSenderDetailsEntity.type` field is missing `@Enumerated(EnumType.STRING)`** | Enum will serialize as ordinal int. The DB column is varchar(30) NOT NULL — INSERT will fail unless converter is registered elsewhere. | ❌ Likely a bug. (`SenderDetailsEntity` may handle it via converter — sibling entity exists at `dionysus_sender_details`.) |
| 5 | `RemittanceController.updateRate` accepts **`com.glidetech.remittance.api.v1.domain.pricing.UpdateBasePricingRequest` directly** (downstream DTO) | Same anti-pattern as `CreateSignzyJourneyRequest` in Set 1. No gateway DTO. | ⚠ Violates "Always Use DTOs and Mappers". |
| 6 | Currency code lengths | `dionysus_international_transaction.from_currency`/`to_currency` = **varchar(3)**, but `dionysus_fx_pricing_base_rates.from_currency`/`to_currency` = **varchar(10)** | ❌ Same logical field, two widths. |
| 7 | `dionysus_international_transaction.beneficiary_account_number` = **varchar(35) NOT NULL** vs `dionysus_beneficiary_account_detail.account_number` = **varchar(30) NOT NULL** | The transaction stores a wider beneficiary account number than the beneficiary table itself uses. | ❌ |
| 8 | `CreatePaymentRequest` (gateway) missing @Size | `beneficiaryAccount`, `description`, `senderIdentifier`, `purpose`, `sourceOfFund`, `relationship`, `couponCode`, `customOne/Two/Three`, `clientIpAddress`, `fromAccount`, `quoteId` — none have `@Size`. Several map to DB varchar(30–150) columns. | ❌ Truncation risks. |
| 9 | `CreatePaymentRequest.description` annotated `@NotNull` (no `@NotBlank`, no `@Size`); DB `description` = varchar(255) | Empty string passes gateway; can also send 10K chars and fail at INSERT. | ❌ |
| 10 | `CreatePaymentRequest.amount` `@NotNull` but **not `@Positive` or `@DecimalMin(0)`** | A zero or negative amount passes gateway. Downstream service does not pre-check. | ❌ |
| 11 | `PaymentTransferRequest.beneficiaryAccount` `@NotNull` (no `@NotBlank`/`@Size`) | Empty string passes; no length cap. | ❌ |
| 12 | `PaymentTransferRequest.companyEntryDescription` `@Size(max=10)` | Aligned with NACHA "Company Entry Description" 10-char limit ✅ | ✅ Good example. |
| 13 | `PaymentTransferRequest.settlementPriority` `@Pattern("^(STANDARD\|SAME_DAY)$")` | ✅ Strict pattern. Could use enum + `@NotNull` but functional. | ✅ |
| 14 | `UpdateBasePricingRequest.toCurrencyCode` `@NotBlank` (no `@Size`); DB `to_currency` = varchar(10) | No length cap at gateway. Most ISO codes are 3 chars; 10-char column allows for vendor-prefixed codes (e.g. `XAU-2`). | ⚠ |
| 15 | `UpdateBasePricingRequest` lacks `fromCurrencyCode` parameter | Service updates `to_currency` by partner key; can't override per-from-currency. Constraint on `dionysus_fx_pricing_base_rates` is composite (`from_currency_id`, `to_currency_id`, `channel`) so the update may be ambiguous. | ⚠ |
| 16 | `PricingBaseRatesEntity` declares `fromCurrencyId`/`toCurrencyId` as `nullable=false` but DB columns are **nullable** | ❌ Schema drift between entity contract and DB. |
| 17 | `PartnerSenderDetailsDTO` (downstream) — only `type` has `@NotNull`; no other validation | Mirrors gateway's emptiness. | ⚠ |
| 18 | `CreatePaymentRequest.correlationId` `@Size(max=80)` | DB `correlation_id` varchar(80) | ✅ Aligned. |
| 19 | `dionysus_international_transaction.purpose_code` = varchar(150) — gateway `purpose` has no `@Size` | ❌ |
| 20 | `dionysus_international_transaction.source_of_fund` = varchar(150) — gateway `sourceOfFund` no `@Size` | ❌ |
| 21 | `dionysus_international_transaction.relationship` = varchar(30) — gateway `relationship` no `@Size` | ❌ |
| 22 | `dionysus_international_transaction.coupon_code` = varchar(40) — gateway `couponCode` no `@Size` | ❌ |
| 23 | `dionysus_international_transaction.custom_one/two/three` = varchar(50) each — gateway `customOne/Two/Three` no `@Size` | ❌ |
| 24 | `dionysus_international_transaction.sender_identifier` = varchar(80) — `dionysus_partner_sender_details.identifier` = varchar(80) | ✅ Aligned across the join. |
| 25 | `dionysus_international_transaction.channel` = varchar(100) — `dionysus_fx_pricing_base_rates.channel` = varchar(50) | ❌ Same logical field, two widths. |

---

## 1. `POST /admin/senders` — Create sender

- **Controller:** `SenderController.createSender` (`@RequestMapping("admin/senders")` — admin-only; **no `/external` variant**)
- **Request DTO (gateway):** `SenderDetailsDTO` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/sender/SenderDetailsDTO.java))
- **Service flow:** `SenderService.createSender` → manual XOR + first/last/businessName checks → `BeneficiaryMapper.toPartnerSenderDetailsDTO` → `PartnerSenderManager.createSender(PartnerSenderDetailsDTO)` (**glide-remittance**)
- **Downstream DTO:** `com.glidetech.remittance.api.v1.domain.sender.PartnerSenderDetailsDTO`
- **Persisted to:** `dionysus_partner_sender_details`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `identifier` | String | (no validation) | (no validation) | `identifier` varchar(80) NOT NULL | ⚠ | Server-generated on create. |
| 2 | `businessId` | Long | (no validation; manual XOR in service) | — | `business_id` bigint | ⚠ | |
| 3 | `clientId` | Long | (no validation; manual XOR) | — | `client_id` bigint | ⚠ | |
| 4 | `firstName` | String | (no @Size; service checks blank only when type=INDIVIDUAL) | (no validation) | `first_name` **varchar(150)** | ❌ | Wide column, no gateway cap. |
| 5 | `lastName` | String | (no @Size; service checks blank only when type=INDIVIDUAL) | (no validation) | `last_name` **varchar(150)** | ❌ | Same. |
| 6 | `businessName` | String | (no @Size; service checks blank only when type=BUSINESS) | (no validation) | `business_name` **varchar(150)** | ❌ | Same. |
| 7 | `phone` | String | (no @Size, no @Pattern) | (no validation) | `phone` varchar(20) | ❌ | Same as gateway phone gaps elsewhere — no `\d{1,15}` pattern. |
| 8 | `type` | enum `SenderType` | (no @NotNull) | @NotNull on `PartnerSenderType` | `type` varchar(30) NOT NULL | ⚠ | Gateway misses `@NotNull`; downstream enforces. **Also `PartnerSenderDetailsEntity.type` field is missing `@Enumerated(EnumType.STRING)` annotation** — see headline #4. |
| 9 | `dateOfBirth` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | @JsonFormat("yyyy-MM-dd") (no @Past) | `date_of_birth` date | ❌ | Entity declares it as `LocalDateTime` — type drift. |
| 10 | `email` | String | (no @Email, no @Size) | (no validation) | `email` varchar(100) | ❌ | |
| 11 | `addressOne` | String | (no @Size) | (no validation) | `address_one` **varchar(100)** | ❌ | 100 vs 80 (hathor_client_address) vs 50 (dionysus_beneficiary_address) vs 30 (khonsu_business_prospect). 4 different widths. |
| 12 | `addressTwo` | String | (no @Size) | (no validation) | `address_two` **varchar(100)** | ❌ | Same. |
| 13 | `city` | String | (no @Size) | (no validation) | `city` **varchar(150)** | ❌ | 150 vs 128/30 elsewhere. |
| 14 | `state` | String | (no @Size) | (no validation) | `state` **varchar(150)** | ❌ | 150 vs 64/25 elsewhere. |
| 15 | `postalCode` | String | (no @Size, no @Pattern) | (no validation) | `postal_code` varchar(20) | ❌ | 20 vs 10/7. |
| 16 | `countryCode` | String | (no @Size, no @Pattern) | (no validation) | `country_code` varchar(3) | ✅ | DB width aligned with hathor_client_address. |
| 17 | `documentType` | String | (no @Size) | (no validation) | `document_type` varchar(50) | ⚠ | |
| 18 | `documentId` | String | (no @Size) | (no validation) | `document_id` **varchar(150)** | ❌ | |
| 19 | `clientIpAddress` | String | (no @Size) | (not persisted on partner-sender entity) | — | ⚠ | |
| 20 | `createdOn`/`updatedOn` | LocalDateTime | response-side only | response-side only | timestamps | ✅ | Server-set on @PrePersist. |

**Service-level rules** (not annotations): clientId XOR businessId required; firstName+lastName required if `type=INDIVIDUAL`; businessName required if `type=BUSINESS`.

---

## 2. `PUT /admin/senders/{identifier}` — Update sender

- **Controller:** `SenderController.updateSender`
- **Request DTO (gateway):** `SenderDetailsDTO` (re-used)
- **Service flow:** `SenderService.updateSender` → `BeneficiaryMapper.toPartnerSenderDetailsDTO` → `PartnerSenderManager.updateSender(identifier, PartnerSenderDetailsDTO)` (**glide-remittance**)
- **Downstream DTO:** Same as §1
- **Persisted to:** UPDATE on `dionysus_partner_sender_details`

All field rows from §1 apply.

---

## 3. `POST /admin/payments` | `POST /external/payments` — Initiate international payment

- **Controller:** `RemittanceController.initiatePayment`
- **Request DTO (gateway):** `CreatePaymentRequest` ([file](../../glide-api-gateway/api/src/main/java/com/glidetech/api_gateway/api/v1/domain/remittance/CreatePaymentRequest.java))
- **Service flow:** `RemittanceService.initiatePayment` → `PaymentMapper.toMap` → `RemittanceManager.initiatePayment(InternationalPaymentsRequest)` (**glide-remittance**)
- **Downstream DTO:** `com.glidetech.remittance.api.v1.domain.payment.InternationalPaymentsRequest`
- **Persisted to:** `dionysus_international_transaction`

| # | Field | Type | Gateway DTO | Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `businessId` | Long | (no validation) | — | `business_id` bigint | ⚠ | |
| 2 | `clientId` | Long | (no validation) | — | `client_id` bigint NOT NULL | ⚠ | |
| 3 | `quoteId` | String | (no @Size) | (no @Size) | `quote_id` varchar(50) | ❌ | |
| 4 | `fromAccount` | String | (no @Size) | (no @Size) | `client_account_number` varchar(30) | ❌ | |
| 5 | `beneficiaryAccount` | String | @NotBlank (no @Size) | @NotBlank (no @Size) | `beneficiary_account_number` **varchar(35) NOT NULL** | ❌ | DB allows 35 but `dionysus_beneficiary_account_detail.account_number` is varchar(30) — mismatch. |
| 6 | `senderIdentifier` | String | (no @Size) | (no @Size) | `sender_identifier` varchar(80) | ✅ | DB width matches `dionysus_partner_sender_details.identifier`. Gateway has no cap. |
| 7 | `fromCurrencyId` | Long | (no validation) | (no validation) | `from_currency_id` integer | ⚠ | |
| 8 | `toCurrencyId` | Long | (no validation) | (no validation) | `to_currency_id` integer | ⚠ | |
| 9 | `amountCurrencyId` | Long | (no validation) | (no validation) | `amount_currency_id` integer | ⚠ | |
| 10 | `amountType` | String | (no validation, no @Pattern) | enum `AmountType` (no @NotNull) | derived | ⚠ | Gateway is plain String; downstream is enum. Mapper does the enum lookup — invalid values fail late. |
| 11 | `fundSource` | String | (no validation) | enum `FundSource` (no @NotNull) | `fund_source` varchar(50) | ⚠ | Same gateway/downstream type drift. |
| 12 | `amount` | BigDecimal | @NotNull (no @Positive, no @DecimalMin) | @NotNull | `amount` numeric(19,5) NOT NULL | ❌ | Negative/zero passes gateway. |
| 13 | `rate` | BigDecimal | (no validation) | (no validation) | `rate` numeric(19,5) | ⚠ | |
| 14 | `fees` | BigDecimal | (no validation) | (no validation) | `fees` numeric(19,5) | ⚠ | |
| 15 | `purpose` | String | (no @Size) | (no @Size) | `purpose_code` varchar(150) | ❌ | |
| 16 | `sourceOfFund` | String | (no @Size) | (no @Size) | `source_of_fund` varchar(150) | ❌ | |
| 17 | `relationship` | String | (no @Size) | (no @Size) | `relationship` varchar(30) | ❌ | |
| 18 | `couponCode` | String | (no @Size) | (no @Size) | `coupon_code` varchar(40) | ❌ | |
| 19 | `description` | String | @NotNull (no @NotBlank, no @Size) | @NotNull (no @Size) | `description` varchar(255) | ❌ | Empty string passes; 10K-char passes gateway. |
| 20 | `clientIpAddress` | String | (no @Size) | mapped to `ipAddress` (no @Size) | `ip_address` varchar(50) | ❌ | |
| 21 | `correlationId` | String | @Size(max=80) | (no @Size) | `correlation_id` varchar(80) | ✅ | Aligned. |
| 22 | `customOne` | String | (no @Size) | (no @Size) | `custom_one` varchar(50) | ❌ | |
| 23 | `customTwo` | String | (no @Size) | (no @Size) | `custom_two` varchar(50) | ❌ | |
| 24 | `customThree` | String | (no @Size) | (no @Size) | `custom_three` varchar(50) | ❌ | |

`InternationalPaymentsRequest` also carries `channel` (server-resolved from beneficiary account) and `quoteId` (echoed). DB also has `provider_*` columns (server-only).

---

## 4. `POST /admin/payments/transfer-fund` | `POST /external/payments/transfer-fund` — Transfer funds (ACH or WIRE)

- **Controller:** `RemittanceController.transferFund`
- **Request DTO (gateway):** `PaymentTransferRequest`
- **Service flow:** `TransactionService.transferFund` → branches by `paymentType`:
  - `WIRE` → `TransactionMapper.toExternalPaymentSystem` → `ExternalPaymentManager.transferFund(...)` (artifact: `transaction-api/v2` — 🔒 deferred, downstream DTO not in repo)
  - `ACH` → `TransactionMapper.toODFIPaymentRequest` → `ODFIPaymentManager.transfer(...)` (**glide-extpaymt** — full DB trace deferred to Set 5)
- **Downstream DTOs:** `com.glidetech.transaction.api.v2.domain.extpayment.ExternalPaymentRequest` 🔒 / `com.glidetech.extpaymt.api.v1.odfi.ODFIPaymentRequest` (covered in Set 5)
- **Persisted to:** Set 5 will cover the `ACH` path; the WIRE path persists to an artifact-managed table.

| # | Field | Type | Gateway DTO | Downstream (per branch) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `paymentType` | enum `PaymentType` | @NotNull | — | ✅ | Switch-case in service. |
| 2 | `clientId` | Long | (no validation, XOR with businessId) | — | ⚠ | Validated by `validateAccess`. |
| 3 | `businessId` | Long | (no validation, XOR) | — | ⚠ | |
| 4 | `fromAccount` | String | (no @Size) | — | ⚠ | If blank, server fills from `CompanyUserAccountEntity`. |
| 5 | `beneficiaryAccount` | String | @NotNull (no @NotBlank, no @Size) | — | ❌ | |
| 6 | `amount` | BigDecimal | @NotNull @Positive | — | ✅ | Best-validated amount field in this set. |
| 7 | `description` | String | (no validation) | — | ❌ | |
| 8 | `correlationId` | String | (no @Size) | — | ❌ | Same field as in `CreatePaymentRequest` has @Size(80) there — inconsistent. |
| 9 | `reference` | String | (no @Size) | — | ⚠ | |
| 10 | `companyEntryDescription` | String | @Size(max=10) | — | ✅ | NACHA 10-char limit. |
| 11 | `achPaymentType` | enum `ExternalPaymentType` | (no @NotNull) | — | ⚠ | |
| 12 | `clientIpAddress` | String | (no @Size) | — | ⚠ | |
| 13 | `instantPayment` | Boolean | (no validation) | — | ⚠ | |
| 14 | `settlementPriority` | String | @Pattern("^(STANDARD\|SAME_DAY)$") | — | ✅ | Could be enum but pattern is correct. |

---

## 5. `POST /admin/pricing/update-rate` | `POST /external/pricing/update-rate` — Update FX base pricing

- **Controller:** `RemittanceController.updateRate`
- **Request DTO (gateway = downstream):** `com.glidetech.remittance.api.v1.domain.pricing.UpdateBasePricingRequest`
  - **Gateway controller accepts the downstream DTO directly** — no gateway DTO.
- **Service flow:** `RemittanceService.updateRate` → `RemittanceManager.updateBasePricing(...)` (**glide-remittance**) → `PricingBaseRatesService.update` → UPDATE on `dionysus_fx_pricing_base_rates`
- **Downstream DTO:** Same as gateway.
- **Persisted to:** `dionysus_fx_pricing_base_rates`

| # | Field | Type | Gateway/Downstream DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `toCurrencyCode` | String | @NotBlank (no @Size, no @Pattern) | `dionysus_fx_pricing_base_rates.to_currency` varchar(10) NOT NULL | ❌ | No @Size at gateway; DB caps at 10. ISO-4217 is 3 chars normally — column is 10 for vendor-prefixed codes. |
| 2 | `channel` | String | @NotBlank (no @Size) | `channel` varchar(50) — but `dionysus_international_transaction.channel` is varchar(100) | ❌ | Cross-table channel length inconsistent. |
| 3 | `rate` | BigDecimal | @NotNull @DecimalMin("0.0", exclusive=true) | `base_rate` numeric(19,10) NOT NULL | ✅ | Strict positive. |
| 4 | `fees` | BigDecimal | @DecimalMin("0.0", inclusive=true) (no @NotNull) | `fee` numeric(19,10) (entity declares NOT NULL with default `BigDecimal.ZERO`, DB allows NULL) | ⚠ | Entity says NOT NULL with default; DB column is nullable. |
| — | (missing) `fromCurrencyCode` | — | not in DTO | `from_currency` varchar(10) NOT NULL | ❌ | DTO has no way to disambiguate which `(from, to, channel)` row to update. Implementation likely picks first match — risk of updating the wrong row. |

**Schema drift:** `PricingBaseRatesEntity.fromCurrencyId`/`toCurrencyId` are declared `nullable=false` but the DB columns are nullable (`is_nullable=YES`). Either widen the entity or tighten the DB.

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /senders` | `firstName` → `dionysus_partner_sender_details.first_name` varchar(150) | ✅ Confirmed extreme width. |
| `POST /senders` | `type` enum → entity field has no `@Enumerated`; DB column varchar(30) NOT NULL | ✅ Confirmed annotation gap. |
| `POST /senders` | `dateOfBirth` LocalDate in DTO → `LocalDateTime` in entity → `date` in DB | ✅ Confirmed type drift. |
| `POST /payments` | `beneficiaryAccount` → `dionysus_international_transaction.beneficiary_account_number` varchar(35) vs `dionysus_beneficiary_account_detail.account_number` varchar(30) | ✅ Confirmed cross-table mismatch. |
| `POST /payments` | `description` `@NotNull` only → `dionysus_international_transaction.description` varchar(255) | ✅ Confirmed gap. |
| `POST /payments/transfer-fund` | `paymentType` switch routes to WIRE/ACH | ✅ Code path confirmed. |
| `POST /pricing/update-rate` | `to_currency` varchar(10) gateway no @Size | ✅ |
| `dionysus_fx_pricing_base_rates` | `from_currency_id` nullable in DB but entity says NOT NULL | ✅ Drift confirmed. |

---

## Recommended next-step actions (for the consistency pass — not in scope of this doc)

1. **Critical:** Add `@Enumerated(EnumType.STRING)` to `PartnerSenderDetailsEntity.type` — current code likely fails at insert or stores ordinal ints (verify by inspecting sample row data).
2. **Critical:** Change `PartnerSenderDetailsEntity.dateOfBirth` from `LocalDateTime` to `LocalDate` to match the DB `date` column and the gateway DTO.
3. Add `@NotNull` + `@Size` + `@Pattern` (where appropriate) to all 20 fields of `SenderDetailsDTO`. At minimum:
   - `clientId`/`businessId` — XOR class-level constraint
   - `firstName`/`lastName`/`businessName` — `@Size(max=...)` with canonical width
   - `phone` — `@Pattern("\\d{1,15}")`
   - `type` — `@NotNull`
   - `dateOfBirth` — `@Past` + `@JsonFormat`
   - `email` — `@Email @Size(max=100)`
   - address fields — `@Size` per column width
   - `countryCode` — `@Size(min=2,max=3) @Pattern("^[A-Z]{2,3}$")`
4. Pick canonical widths for sender address fields and align — currently `dionysus_partner_sender_details` is the widest (150) in the codebase. Suggest 100 for first/last/business names and city/state.
5. Add `@Size` to every `CreatePaymentRequest` field that maps to a varchar column (rows #3–24 in §3): `quoteId(50)`, `fromAccount(30)`, `beneficiaryAccount(35)`, `senderIdentifier(80)`, `purpose(150)`, `sourceOfFund(150)`, `relationship(30)`, `couponCode(40)`, `description(255)`, `clientIpAddress(50)`, `customOne/Two/Three(50)`.
6. Add `@NotBlank` to `CreatePaymentRequest.description` (currently `@NotNull` only).
7. Add `@Positive` or `@DecimalMin("0.01")` to `CreatePaymentRequest.amount`.
8. Reconcile `beneficiary_account_number` length across `dionysus_international_transaction` (35) and `dionysus_beneficiary_account_detail` (30) — pick one.
9. Reconcile currency-code column lengths: `dionysus_international_transaction.from_currency/to_currency` (3) vs `dionysus_fx_pricing_base_rates.from_currency/to_currency` (10) — pick canonical width.
10. Reconcile `channel` column lengths: 50 (pricing) vs 100 (transaction). Suggest 50 with `@Size(max=50)` everywhere.
11. Introduce a gateway-layer `UpdateBasePricingRequest` DTO that does **not** mirror the downstream class. Add `fromCurrencyCode` to disambiguate updates.
12. Resolve `PricingBaseRatesEntity` ↔ DB schema drift: entity says `from_currency_id`/`to_currency_id` are NOT NULL; DB says nullable. Pick one (recommend NOT NULL — these are FX pair keys).
13. Add `@NotBlank` + `@Size` to `PaymentTransferRequest.beneficiaryAccount` (currently `@NotNull` only).
14. Consider replacing `PaymentTransferRequest.settlementPriority` String + @Pattern with an enum.
15. `SenderController` is admin-only — confirm with product whether `/external/senders` should also exist; if so, add the dual path.
