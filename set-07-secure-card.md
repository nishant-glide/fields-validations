# Set 7 — Secure Card (Credit)

7 endpoints under `SecureCardController` (admin-only — `@RequestMapping("admin/cards")`). Manages the secure (credit) card lifecycle: account creation, card linking, payment settings, payments, statements, and transactions.

Downstream surfaces:
- **glide-client-account** / `cards` module — `osiris_secure_card_account`, `osiris_secure_card_mapping`, `osiris_secure_card_payment`, `osiris_secure_card_payment_history`, `osiris_secure_card_setting`, `osiris_secure_card_statement`
- **cards-api artifact** for some downstream calls — 🔒 deferred

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`CreateSecureCardAccountRequestDto`**: only `nameOnCard` has `@NotNull`. `clientId`/`businessId` no XOR; `currencyCode` no `@Size`/`@Pattern`; `correlationId` no `@Size`. | ❌ |
| 2 | **`MakePaymentRequestDto.amount` has no `@NotNull` and no `@Positive`** — partner can send `null` amount (or negative); only `paymentType` (string) gates which branch is taken. | ❌ |
| 3 | **`MakePaymentRequestDto.paymentType` is a plain `String`** with no `@Pattern`/`@NotBlank` — no enum and no enumerated values. Critical control field for payment branching (full/min/partial). | ❌ |
| 4 | **`MakePaymentRequestDto.clientId` `@NotNull`** but `businessId` is nullable — looks like client-only flow despite the controller advertising both. | ⚠ Inconsistent with sibling DTOs. |
| 5 | `UpdatePaymentSettingsRequestDto.paymentPeriod` is `Integer` with no `@Min`/`@Max` | DB `payment_period` is `smallint` (max 32767). Reasonable values are 1–28 (days). | ❌ |
| 6 | `LinkCardToSecureAccountRequestDto.nameOnCard` no `@NotBlank`, no `@Size`; `cardType` and `type` have **good** `@Pattern` (CREDIT/DEBIT, PHYSICAL/VIRTUAL) | ⚠ / ✅ |
| 7 | `osiris_secure_card_account.account_id` = varchar(20) (matches `osiris_client_account.account_id`) | ✅ |
| 8 | `osiris_secure_card_account.correlation_id` = varchar(255) vs partner DTO has no `@Size`; other tables use 50/80 | ❌ Cross-table inconsistency. |
| 9 | `osiris_secure_card_account.account_name` = varchar(255) — gateway DTO `nameOnCard` has no `@Size`. Card names typically embossed at 26 chars max. | ❌ |
| 10 | `osiris_secure_card_account.currency_code` = varchar(10) — partner DTO `currencyCode` has no `@Pattern`/`@Size`. Should be 3-char ISO. | ❌ |
| 11 | `osiris_secure_card_mapping.card_token` = varchar(64) vs `horus_card_account.card_token` = varchar(255) vs `horus_card_txn_details.card_token` = varchar(100) | ❌ Three widths for the same logical field. |
| 12 | `osiris_secure_card_mapping.card_nickname` = varchar(50) (no gateway field maps to this) | ⚠ |
| 13 | `osiris_secure_card_payment_history.account_id`, `primary_account_number` both NOT NULL varchar(20) | ✅ Aligned. |
| 14 | `osiris_secure_card_statement.pdf_url` = varchar(500), `bucket_name` = varchar(100) | ✅ Reasonable. |
| 15 | `GenerateStatementRequestDto.recreate` is a primitive `boolean` (default false). Idempotency-relevant flag. | ⚠ |
| 16 | `GenerateStatementPdfRequestDto.startDate`/`endDate` no `@Past`/`@PastOrPresent`, no relative-order validation (`startDate < endDate`) | ⚠ |
| 17 | `SecureCardTransactionRequestDto.startDate`/`endDate` no validation; controller-level note says "Max 1 month range" — enforced in service, not DTO | ⚠ |
| 18 | All `Long clientId` / `Long businessId` pairs across the 7 DTOs: no XOR constraint — partner can send both or neither | ❌ Service-level fallback only. |
| 19 | `osiris_secure_card_setting.payment_period`/`statement_day` are both `smallint` (16-bit). Gateway DTO uses `Integer` — type drift but values fit. | ⚠ |
| 20 | `osiris_secure_card_account` has DB column `session_id` varchar(255) — wider than other tables (40/80) | ⚠ Drift. |

---

## 1. `POST /admin/cards/account/create` — Create secure card account

- **Controller:** `SecureCardController.createSecureCardAccount`
- **Request DTO:** `CreateSecureCardAccountRequestDto`
- **Service flow:** `SecureCardService.createSecureCardAccount` → `SecureCardAccountApiManager.createAccount(...)` (cards-api artifact — 🔒 deferred)
- **Persisted to:** `osiris_secure_card_account`

| # | Field | Type | Gateway | DB column (`osiris_secure_card_account`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` bigint NOT NULL | ⚠ | No XOR. |
| 2 | `businessId` | Long | (no validation) | `business_id` bigint | ⚠ | |
| 3 | `correlationId` | String | (no @Size) | `correlation_id` varchar(255) | ❌ | |
| 4 | `currencyCode` | String | (no @Size, no @Pattern) | `currency_code` varchar(10) | ❌ | Should be `@Size(min=3,max=3) @Pattern("[A-Z]{3}")`. |
| 5 | `nameOnCard` | String | @NotNull (no @Size) | `account_name` varchar(255) | ❌ | Card-embossing standard is 26 chars; column allows 255 with no gateway cap. |

---

## 2. `POST /admin/cards/account/add-card` — Add card to secure account

- **Controller:** `SecureCardController.linkCardToSecureAccount`
- **Request DTO:** `LinkCardToSecureAccountRequestDto`
- **Service flow:** `SecureCardService.linkCardToSecureAccount` → `SecureCardAccountApiManager.linkCard(...)` (🔒 deferred)
- **Persisted to:** `osiris_secure_card_mapping`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | `secure_account_id` varchar(20) NOT NULL | ⚠ | |
| 4 | `cardType` | String | @Pattern("^(CREDIT\|DEBIT)$") | `card_type` varchar(20) | ✅ | Best-validated field in Set 7 (along with `type`). |
| 5 | `type` | String | @Pattern("^(PHYSICAL\|VIRTUAL)$") | (not mapped directly; influences provider call) | ✅ | |
| 6 | `nameOnCard` | String | (no validation) | (server-side; influences card-embossing) | ❌ | |

---

## 3. `POST /admin/cards/settings/update` — Update payment settings

- **Controller:** `SecureCardController.updatePaymentSettings`
- **Request DTO:** `UpdatePaymentSettingsRequestDto`
- **Service flow:** `SecureCardService.updatePaymentSettings` → `SecureCardAccountApiManager.updateSettings(...)` (🔒 deferred)
- **Persisted to:** `osiris_secure_card_setting` UPDATE

| # | Field | Type | Gateway | DB column (`osiris_secure_card_setting`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | (not stored on setting; key via account_id) | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` bigint | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank | `account_id` varchar(20) NOT NULL | ⚠ | No `@Size`. |
| 4 | `hassleFreePay` | Boolean | (no validation) | `hassle_free_pay` bool | ✅ | |
| 5 | `paymentPeriod` | Integer | (no @Min/@Max) | `payment_period` smallint | ❌ | No bounds; smallint allows up to 32767 — partner could send 9999 "days". |

---

## 4. `POST /admin/cards/payment` — Make payment

- **Controller:** `SecureCardController.makePayment`
- **Request DTO:** `MakePaymentRequestDto`
- **Service flow:** `SecureCardService.makePayment` → routes by `paymentType` (FULL / MINIMUM / PARTIAL) → corresponding manager call (🔒 deferred)
- **Persisted to:** `osiris_secure_card_payment_history` (new row); `osiris_secure_card_payment` UPDATE

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | @NotNull | `client_id` | ✅ | Sole field with `@NotNull`. |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | Asymmetric vs `clientId`. |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | `account_id` varchar(20) NOT NULL · `primary_account_number` varchar(20) NOT NULL | ⚠ | |
| 4 | `sourceAccountNumber` | String | @NotBlank (no @Size) | (server-resolved, debit account for the payment) | ⚠ | |
| 5 | `amount` | BigDecimal | (no @NotNull, no @Positive) | `payment_amount` numeric NOT NULL on history | ❌ | **Critical:** money-movement field with no validation. |
| 6 | `paymentType` | String | (no @NotBlank, no @Pattern) | `payment_type` varchar(30) on history; varchar(25) on payment | ❌ | Control field — should be enum + `@NotBlank`. |

---

## 5. `POST /admin/cards/statement/generate` — Generate statement

- **Controller:** `SecureCardController.generateStatement`
- **Request DTO:** `GenerateStatementRequestDto`
- **Service flow:** `SecureCardService.generateStatement` → `SecureCardAccountApiManager.generateStatement(...)` (🔒 deferred)
- **Persisted to:** new row in `osiris_secure_card_statement`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | `account_id` varchar(20) NOT NULL | ⚠ | |
| 4 | `recreate` | boolean | (primitive) | (idempotency flag — affects whether existing statement is replaced) | ⚠ | |

---

## 6. `POST /admin/cards/statement/pdf/generate` — Generate statement PDF (date range)

- **Controller:** `SecureCardController.generateStatementPdf`
- **Request DTO:** `GenerateStatementPdfRequestDto`
- **Service flow:** `SecureCardService.generateStatementPdf` → `SecureCardAccountApiManager.generatePdf(...)` (🔒 deferred). PDF stored at `osiris_secure_card_statement.pdf_url`.

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | ⚠ | |
| 2 | `businessId` | Long | (no validation) | — | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | `account_id` varchar(20) | ⚠ | |
| 4 | `startDate` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past, no required) | `statement_period_start` date | ⚠ | |
| 5 | `endDate` | LocalDate | @JsonFormat("yyyy-MM-dd") (no required, no order check) | `statement_period_end` date | ⚠ | |

---

## 7. `POST /admin/cards/transactions` — Get secure card transactions

- **Controller:** `SecureCardController.getSecureCardTransactions`
- **Request DTO:** `SecureCardTransactionRequestDto`
- **Service flow:** `SecureCardService.getSecureCardTransactions` → routes through `TransactionsManager` (transaction-api artifact — 🔒 deferred). Service caps date range to 1 month (per `@Operation` description).
- **Persisted to:** Read-only.

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | ⚠ | |
| 2 | `businessId` | Long | (no validation) | — | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | `account_id` varchar(20) | ⚠ | |
| 4 | `startDate` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | — | ⚠ | |
| 5 | `endDate` | LocalDate | @JsonFormat("yyyy-MM-dd") | — | ⚠ | Service enforces ≤1 month range. Belongs in DTO. |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /cards/account/create` | `nameOnCard` `@NotNull` only — DB `account_name` varchar(255) | ✅ |
| `POST /cards/account/add-card` | `cardType`/`type` `@Pattern` enums | ✅ |
| `POST /cards/settings/update` | `paymentPeriod` no @Min/@Max — DB smallint | ✅ |
| `POST /cards/payment` | `amount` no @NotNull, no @Positive | ✅ Confirmed critical gap. |
| `POST /cards/statement/generate` | `recreate` primitive | ✅ |
| `POST /cards/statement/pdf/generate` | `startDate`/`endDate` no relative-order check | ✅ |
| `POST /cards/transactions` | Date-range cap enforced in service, not DTO | ✅ |

---

## Recommended next-step actions

1. **Critical:** Add `@NotNull` and `@Positive` (or `@DecimalMin("0.01")`) to `MakePaymentRequestDto.amount`.
2. Add `@NotBlank` and `@Pattern("^(FULL\|MINIMUM\|PARTIAL)$")` to `MakePaymentRequestDto.paymentType` (or convert to enum).
3. Add XOR `@AssertTrue` class-level constraint for `clientId`/`businessId` on all 7 DTOs.
4. Add `@Size(min=3,max=3) @Pattern("[A-Z]{3}")` to `CreateSecureCardAccountRequestDto.currencyCode`.
5. Add `@Size(max=26)` to `CreateSecureCardAccountRequestDto.nameOnCard` (embossing standard) and `LinkCardToSecureAccountRequestDto.nameOnCard`.
6. Add `@Min(1) @Max(28)` to `UpdatePaymentSettingsRequestDto.paymentPeriod`.
7. Add `@Size(max=20)` to every `accountNumber` field across the 7 DTOs.
8. Add class-level `@AssertTrue` validator on `GenerateStatementPdfRequestDto` ensuring `startDate <= endDate` and both required.
9. Add `@PastOrPresent` to all `startDate`/`endDate` fields.
10. Move the 1-month range cap from `SecureCardService` into the DTO as a custom class-level constraint.
11. Reconcile `card_token` width across `osiris_secure_card_mapping` (64), `horus_card_account` (255), `horus_card_txn_details` (100). Pick canonical.
12. Reconcile `correlation_id` width: `osiris_secure_card_account` (255) vs other tables (50/80). Pick canonical 80.
13. Narrow `osiris_secure_card_account.session_id` from 255 to 80 (matches `osiris_spending_account.session_id`).
14. Add `@Pattern` for `paymentType` consistency between `osiris_secure_card_payment` (varchar 25) and `osiris_secure_card_payment_history` (varchar 30) — currently column widths differ.
