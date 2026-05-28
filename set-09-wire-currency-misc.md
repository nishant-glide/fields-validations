# Set 9 — Wire / Currency / Trading / Fees / Misc

11 endpoints spread across 7 controllers (the heaviest mixed-bag set):
- `WireInstructionsController` (2): create wire instructions, withdraw fund
- `CurrencyConversionController` (1): execute currency conversion
- `PaymentController` (1): card move-fund
- `PartnerBankingFeeController` (2): banking fee, partner fee
- `TradingController` (3): business sweep settings, client sweep settings, AUM fee
- `TransactionController` (1): funding payment
- `ExternalWalletController` (1): create external wallet

Downstream surfaces:
- **glide-cardpymt** (card move-fund — partially)
- **glide-remittance** — currency conversion + wire withdraw (some artifact paths)
- **transaction-api artifact** — wire and funding payment downstream DTOs 🔒 deferred
- **trading-api artifact** — sweep, AUM fee 🔒 deferred
- **glide-client-account** — external wallet (`ExternalWalletManager`)

Mythology mapping confirmed:
- `hermes_*` → **transaction-api / wire-instructions** module (23 tables — covers wire flow)
- `hades_*` → **trading-api artifact** (33 tables — AUM, sweep, positions, securities)
- `helios_*` → **external payment / integration** (13 tables)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`WireInstructionsController.createWireInstructions` missing `@Valid`** on `@RequestBody WireInstructionsRequest` | Even though the DTO has `@NotBlank` on 7 fields, none of them will fire. | ❌ |
| 2 | **`ExternalWalletController.createExternalWallet` missing `@Valid`** on `@RequestBody CreateExternalWalletRequest` | Downstream DTO has zero validation anyway. | ❌ |
| 3 | **`TradingController.updateClientSweepSettings` missing `@Valid`** on `@RequestBody SweepSettings` | `updateBusinessSweepSettings` has `@Valid` (inconsistent). | ❌ |
| 4 | **`TradingController.setAumFee` missing `@Valid`** on `@RequestBody AumFeeRequest` | DTO has zero validation anyway. | ❌ |
| 5 | **`CurrencyConversionController.initiateCurrencyConversion` accepts downstream DTO directly** (`com.glidetech.remittance.api.v1.domain.payment.CurrencyConversionRequest`) | Same anti-pattern as Sets 1/4/8. | ⚠ |
| 6 | **`PaymentController.moveFund` re-uses the parent `PaymentRequest` DTO** (also used for ACH). Path-variable `paymentMethod` has `@Pattern("^card$")` — strange (regex forced lowercase). Service ignores the URL value anyway and always calls `cardMoveFund`. | ⚠ |
| 7 | `WireInstructionsRequest`: 7 `@NotBlank` fields ✅; **no `@Size` anywhere**; **no `@Pattern("\\d{9}")` on `wireRoutingNumber`** (US Fed routing = 9 digits); no `@Pattern` on `accountNumber` | ❌ |
| 8 | `hermes_wire_instructions.zip_code` = varchar(10) but `hermes_wire_instructions.street_address` = varchar(255) — gateway DTO has neither `@Size` | ⚠ |
| 9 | `hermes_wire_instructions.business_name` = **varchar(255)** but `hathor_business.company` = varchar(100) NOT NULL | ❌ Same logical field, two widths. |
| 10 | `hermes_wire_instructions.first_name`/`last_name` = **varchar(255)** — far wider than every other first-name column (20/32/50/150) | ❌ Adds a 5th width for the same logical field. |
| 11 | `hermes_wire_instructions.wire_routing_number`/`ach_routing_number`/`account_number` all varchar(20) — should be `@Pattern("\\d{9}")` (routing) and tighter (account) | ❌ |
| 12 | `BankingFeeRequest` and `PartnerFeeRequest` have **good validation** — `@NotBlank` on `accountNumber`, `@NotNull @Positive` on `amount`, `@Pattern` on `direction` | ✅ Best Set 9 DTOs. Still missing `@Size` everywhere. |
| 13 | `BankingFeeRequest.amount` has **`@NotNull` twice** (line 21 and 22) — harmless duplication | ⚠ Code smell. |
| 14 | `FundingPaymentRequest` has solid validation (`@NotBlank` on fromAccount/toAccount, `@NotNull @Positive` on amount) — but **no `@Size`** | ⚠ |
| 15 | `WirePaymentRequest.wireInstructionsId` `@NotNull` only (no `@NotBlank`, no `@Size`) | ❌ |
| 16 | `WirePaymentRequest`: `description`, `correlationId`, `clientIpAddress`, `fromAccount` — no `@Size` | ❌ |
| 17 | `CurrencyConversionRequest.amount` `@NotNull` but **no `@Positive`** | ❌ |
| 18 | `CurrencyConversionRequest.fromAccountNumber`/`toAccountNumber` `@NotBlank` (no `@Size`) | ⚠ |
| 19 | `SweepSettings.accountNumber` `@NotBlank`; `settings` list `@Valid` ✅; `marginEnable`/`stockLending` Boolean primitives default false | ⚠ |
| 20 | `AumFeeRequest` has **zero validation** — `fee` BigDecimal nullable, no `@Positive`/`@DecimalMin`/`@DecimalMax`; `clientId`/`businessId` no XOR | ❌ |
| 21 | `CreateExternalWalletRequest` (downstream, no gateway wrapper) has **zero validation** on all 6 fields | ❌ |
| 22 | `dionysus_currency_exchange_transactions.from_account`/`to_account` = **varchar(255)** vs `dionysus_international_transaction.client_account_number` = varchar(30) | ❌ Same logical concept, two widths. |
| 23 | `dionysus_currency_exchange_transactions.payment_identifier`/`idempotent_id` = **varchar(255)** vs `dionysus_international_transaction.payment_identifier` = varchar(60) | ❌ |
| 24 | `osiris_fund_advance_fees` exists but isn't covered by any of the 11 endpoints — separate flow. | — |
| 25 | `PaymentRequest` (used by `PaymentController.moveFund` AND `ACHController.moveFund`) — **dual-use DTO**. The `@Pattern` validation on the path variable `paymentMethod` is dead code (forced lowercase 'card' only, regex can't match path values from typical clients). | ⚠ |

---

## 1. `POST /admin/wire/instructions` — Create wire instructions

- **Controller:** `WireInstructionsController.createWireInstructions` — **⚠ missing `@Valid`** on `@RequestBody`
- **Request DTO:** `WireInstructionsRequest`
- **Service flow:** `WireInstructionsService.createWireInstructions` → `PayeeAccountManager.create(...)` (transaction-api artifact 🔒)
- **Persisted to:** `hermes_wire_instructions`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `identifier` | String | (no validation) | `identifier` varchar(20) NOT NULL | ⚠ | Server-generated. |
| 2 | `businessName` | String | (no validation) | `business_name` varchar(255) | ❌ | Inconsistent with `hathor_business.company` varchar(100). |
| 3 | `firstName` | String | (no validation) | `first_name` varchar(255) | ❌ | 5th width for "first_name" across the codebase. |
| 4 | `lastName` | String | (no validation) | `last_name` varchar(255) | ❌ | Same. |
| 5 | `businessId` | Long | (no validation) | `business_id` bigint | ⚠ | |
| 6 | `clientId` | Long | (no validation) | (not stored on wire_instructions) | ⚠ | |
| 7 | `accountNickname` | String | @NotBlank | `account_nickname` varchar(60) NOT NULL | ⚠ | No `@Size`. |
| 8 | `streetAddress` | String | @NotBlank | `street_address` varchar(255) | ⚠ | |
| 9 | `city` | String | @NotBlank | `city` varchar(100) | ⚠ | |
| 10 | `state` | String | @NotBlank | `state` varchar(100) | ⚠ | |
| 11 | `zipCode` | String | @NotBlank | `zip_code` varchar(10) | ⚠ | |
| 12 | `accountNumber` | String | @NotBlank | `account_number` varchar(20) NOT NULL | ⚠ | |
| 13 | `wireRoutingNumber` | String | (no validation, no @Pattern) | `wire_routing_number` varchar(20) | ❌ | Should be `@Pattern("\\d{9}") @Size(min=9,max=9)`. |
| 14 | `referenceId` | String | (no validation) | `reference_id` varchar(70) | ⚠ | |
| 15 | `clientIpAddress` | String | (no validation) | (not stored) | ⚠ | |

**Note:** Even if every `@NotBlank` were correct, the controller is missing `@Valid` — so none of them fire.

---

## 2. `POST /admin/wire/withdraw-fund` — Wire withdraw

- **Controller:** `WireInstructionsController.withdrawFund` — `@Valid` present ✅
- **Request DTO:** `WirePaymentRequest`
- **Service flow:** `TransactionService.wireWithdrawFund` → `ExternalPaymentManager.withdrawFund(...)` (transaction-api artifact 🔒)
- **Persisted to:** Wire transactions persist in artifact-managed tables and `hermes_wire_transactions`.

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | — | ⚠ | |
| 2 | `businessId` | Long | (no validation) | — | ⚠ | |
| 3 | `fromAccount` | String | (no validation) | — | ❌ | Service falls back to `CompanyUserAccountEntity.accountId` if blank. |
| 4 | `wireInstructionsId` | String | @NotNull (no @NotBlank, no @Size) | links to `hermes_wire_instructions.identifier` varchar(20) | ❌ | |
| 5 | `amount` | BigDecimal | @NotNull @Positive | (artifact-managed) | ✅ | |
| 6 | `description` | String | (no validation) | (artifact) | ❌ | |
| 7 | `correlationId` | String | (no validation) | (artifact) | ❌ | |
| 8 | `clientIpAddress` | String | (no validation) | — | ⚠ | |

---

## 3. `POST /admin/currency/conversion` | `POST /external/currency/conversion` — Initiate currency conversion

- **Controller:** `CurrencyConversionController.initiateCurrencyConversion`
- **Request DTO (gateway = downstream):** `com.glidetech.remittance.api.v1.domain.payment.CurrencyConversionRequest`
  - **Gateway accepts downstream DTO directly** — no wrapper.
- **Service flow:** Forwarded to `RemittanceManager.initiateCurrencyConversion(...)` (**glide-remittance**)
- **Persisted to:** `dionysus_currency_exchange_transactions`

| # | Field | Type | Gateway/Downstream | DB column (`dionysus_currency_exchange_transactions`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `fromAccountNumber` | String | @NotBlank (no @Size) | `from_account` **varchar(255)** | ❌ | DB is 8× wider than other client_account_number columns (20–35). |
| 2 | `toAccountNumber` | String | @NotBlank (no @Size) | `to_account` varchar(255) | ❌ | Same. |
| 3 | `fromCurrencyId` | Long | @NotNull | (key) | ✅ | |
| 4 | `toCurrencyId` | Long | @NotNull | (key) | ✅ | |
| 5 | `amount` | BigDecimal | @NotNull (no @Positive) | `amount` numeric | ❌ | |
| 6 | `amountCurrencyId` | Long | (no validation) | (server-resolved default to fromCurrencyId) | ⚠ | |
| 7 | `rate` | BigDecimal | (no validation) | (server-resolved or override) | ⚠ | |
| 8 | `fees` | BigDecimal | (no validation) | (server-resolved or override) | ⚠ | |
| 9 | `description` | String | (no @Size) | — | ⚠ | |
| 10 | `correlationId` | String | (no @Size) | `idempotent_id` varchar(255) (mapped) | ⚠ | |
| 11 | `quoteId` | String | (no @Size) | — | ⚠ | |
| 12 | `ipAddress` | String | (no @Size) | — | ⚠ | |

---

## 4. `POST /admin/move-fund/{paymentMethod}` — Card move-fund

- **Controller:** `PaymentController.moveFund`
- **Request DTO:** `PaymentRequest` (parent of `ODFIPaymentRequest` from Set 5)
- Path variable `paymentMethod` has `@Pattern("^card$")` — but the service ignores it and always routes to `cardService.cardMoveFund`. The `@Pattern` is effectively dead.
- **Service flow:** `CardService.cardMoveFund(PaymentRequest)` → routes via `CardApiManager` to a card-vendor (Visa/MC) (🔒 cards-api artifact)
- **Persisted to:** `horus_card_txn_details` (Set 6 traced this table)

Same field-by-field validation as `ODFIPaymentRequest` in Set 5 §1 — `PaymentRequest` is the parent. `fromAccount`/`toAccount` `@NotNull`, `amount` `@NotNull @Positive`, `companyEntryDescription` `@Size(max=10)`, `settlementPriority` `@Pattern`. Rest unvalidated.

**Note:** `PaymentRequest` is a *dual-use* DTO for both ACH (Set 5) and CARD (this set). NACHA-specific `companyEntryDescription` `@Size(10)` is meaningless on card transactions but harmless.

---

## 5. `POST /admin/banking/fees` — Charge banking fee

- **Controller:** `PartnerBankingFeeController.chargeBankingFee` — `@Valid` ✅
- **Request DTO:** `BankingFeeRequest`
- **Service flow:** `TransactionService.applyBankingFee` → debit/credit transaction via internal transaction module (artifact-managed)
- **Persisted to:** transaction-api / `transactions-v1_*` tables (🔒 deferred)

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | ⚠ | |
| 2 | `businessId` | Long | (no validation) | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | ⚠ | |
| 4 | `amount` | BigDecimal | @NotNull (×2 — duplicate) @Positive | ✅ | Has duplicate `@NotNull`. |
| 5 | `feeType` | String | @NotBlank (no @Size, no @Pattern) | ⚠ | Should be enum or `@Pattern`. |
| 6 | `direction` | String | @Pattern("^(CREDIT\|DEBIT)$") | ✅ | |
| 7 | `description` | String | (no @Size) | ⚠ | |
| 8 | `clientIpAddress` | String | (no @Size) | ⚠ | |
| 9 | `referenceId` | String | (no @Size) | ⚠ | |
| 10 | `correlationId` | String | (no @Size) | ⚠ | |

---

## 6. `POST /admin/partner/fees` — Charge partner fee

- **Controller:** `PartnerBankingFeeController.chargePartnerFee` — `@Valid` ✅
- **Request DTO:** `PartnerFeeRequest`
- Same shape as `BankingFeeRequest` minus `feeType`.

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | ⚠ | |
| 2 | `businessId` | Long | (no validation) | ⚠ | |
| 3 | `accountNumber` | String | @NotBlank (no @Size) | ⚠ | |
| 4 | `amount` | BigDecimal | @NotNull @Positive | ✅ | |
| 5 | `description` | String | (no @Size) | ⚠ | |
| 6 | `direction` | String | @Pattern("^(CREDIT\|DEBIT)$") | ✅ | |
| 7 | `referenceId` | String | (no @Size) | ⚠ | |
| 8 | `clientIpAddress` | String | (no @Size) | ⚠ | |
| 9 | `correlationId` | String | (no @Size) | ⚠ | |

---

## 7. `POST /admin/trading/sweep/settings/business/{businessId}` — Update business sweep settings

- **Controller:** `TradingController.updateBusinessSweepSettings` — `@Valid` ✅
- **Request DTO:** `SweepSettings`
- **Service flow:** `TradingService.updateBusinessSweepSettings` → `TradingManager.updateBusinessSweepSettings(...)` (trading-api artifact 🔒)
- **Persisted to:** `hades_business_trading_setting`, `hades_auto_allocation_setting`, `hades_auto_sweep_*` tables

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `accountNumber` | String | @NotBlank (no @Size) | ⚠ | |
| 2 | `settings` | List<SweepSetting> | @Valid (cascades) | ⚠ | Nested SweepSetting validation depends on its own annotations. |
| 3 | `marginEnable` | Boolean | (default false) | ⚠ | |
| 4 | `stockLending` | Boolean | (default false) | ⚠ | |
| 5 | `clientIpAddress` | String | (no validation) | ⚠ | |

---

## 8. `POST /admin/trading/sweep/settings/client/{clientId}` — Update client sweep settings

- **Controller:** `TradingController.updateClientSweepSettings` — **⚠ missing `@Valid`** on `@RequestBody`
- **Request DTO:** `SweepSettings` (re-used)
- Same field-by-field as §7.

---

## 9. `POST /admin/trading/aum-fee` — Set AUM fee

- **Controller:** `TradingController.setAumFee` — **⚠ missing `@Valid`** on `@RequestBody`
- **Request DTO:** `AumFeeRequest`
- **Service flow:** `TradingService.setAumFee` → `TradingManager.setAumFee(...)` (trading-api artifact 🔒)
- **Persisted to:** `hades_aum_contract_fee_override` / `hades_aum_fee_structure` (artifact-managed)

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | ❌ | |
| 2 | `businessId` | Long | (no validation) | ❌ | No XOR. |
| 3 | `fee` | BigDecimal | (no @NotNull, no @Positive, no @DecimalMax) | ❌ | Null fee passes. Negative or 1000 (1000%) also passes. |
| 4 | `clientIpAddress` | String | (no validation) | ⚠ | |

---

## 10. `POST /admin/funding-payments` — Process funding payment

- **Controller:** `TransactionController.processFundingPayment` — `@Valid` ✅
- **Request DTO:** `FundingPaymentRequest`
- **Service flow:** `TransactionService.processFundingPayment` → routes via `TransactionsManager` (transaction-api artifact 🔒)
- **Persisted to:** transaction-api tables (`transactions-v1_*`)

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `fromAccount` | String | @NotBlank (no @Size) | ⚠ | |
| 2 | `toAccount` | String | @NotBlank (no @Size) | ⚠ | |
| 3 | `amount` | BigDecimal | @NotNull @Positive | ✅ | |
| 4 | `reference` | String | (no @Size) | ⚠ | |
| 5 | `description` | String | (no @Size) | ⚠ | |
| 6 | `correlationId` | String | (no @Size) | ⚠ | |
| 7 | `clientIpAddress` | String | (no @Size) | ⚠ | |

---

## 11. `POST /admin/external-wallets` — Create external wallet (crypto)

- **Controller:** `ExternalWalletController.createExternalWallet` — **⚠ missing `@Valid`** on `@RequestBody`
- **Request DTO (gateway = downstream):** `com.glidetech.clientaccount.api.v1.domain.externalwallet.CreateExternalWalletRequest`
  - **Gateway accepts downstream DTO directly** — no gateway wrapper.
- **Service flow:** `ExternalWalletService.createExternalWallet` → `ExternalWalletManager.create(...)` (**glide-client-account**)
- **Persisted to:** `osiris_external_wallet`

| # | Field | Type | Gateway/Downstream | DB column (`osiris_external_wallet`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 3 | `name` | String | (no validation) | `name` varchar(255) | ❌ | |
| 4 | `asset` | String | (no validation) | `asset` varchar(100) | ❌ | |
| 5 | `address` | String | (no validation, no @Pattern) | `address` varchar(255) | ❌ | |
| 6 | `tag` | String | (no validation) | `tag` varchar(200) | ❌ | |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /wire/instructions` | Controller missing `@Valid` despite DTO `@NotBlank` | ✅ Confirmed. |
| `POST /wire/withdraw-fund` | `wireInstructionsId` `@NotNull` only | ✅ |
| `POST /currency/conversion` | `amount` `@NotNull` no `@Positive` | ✅ |
| `POST /banking/fees` | Duplicate `@NotNull` on amount | ✅ |
| `POST /partner/fees` | Well-validated | ✅ |
| `POST /trading/sweep/settings/business/{id}` | `@Valid` present | ✅ |
| `POST /trading/sweep/settings/client/{id}` | **`@Valid` missing** | ✅ Confirmed. |
| `POST /trading/aum-fee` | DTO has zero validation + controller missing `@Valid` | ✅ Confirmed. |
| `POST /funding-payments` | Well-validated | ✅ |
| `POST /external-wallets` | Controller missing `@Valid` + downstream-DTO leakage | ✅ |
| `dionysus_currency_exchange_transactions.from_account` varchar(255) vs `dionysus_international_transaction.client_account_number` varchar(30) | ✅ |
| `hermes_wire_instructions.business_name` varchar(255) vs `hathor_business.company` varchar(100) | ✅ |

---

## Recommended next-step actions

1. **Critical:** Add `@Valid` to 4 controllers' `@RequestBody` parameters: `WireInstructionsController.createWireInstructions`, `ExternalWalletController.createExternalWallet`, `TradingController.updateClientSweepSettings`, `TradingController.setAumFee`.
2. Introduce gateway-layer wrappers for `CurrencyConversionRequest` and `CreateExternalWalletRequest` (both currently leak downstream contract).
3. Add `@Pattern("\\d{9}") @Size(min=9,max=9)` to `WireInstructionsRequest.wireRoutingNumber` (US Fed routing).
4. Add `@Size` to every `WireInstructionsRequest` field that maps to a varchar column (`businessName` ≤ 100 to match `hathor_business`, address fields per DB widths).
5. Add `@Positive` (or `@DecimalMin("0.01")`) to `CurrencyConversionRequest.amount`.
6. Add `@NotNull @Positive @DecimalMax("100.00")` to `AumFeeRequest.fee` (fee as a percentage with sane bounds).
7. Add XOR class-level constraint on `clientId`/`businessId` to all DTOs in this set.
8. Remove duplicate `@NotNull` on `BankingFeeRequest.amount` (lines 21 and 22).
9. Reconcile name-column widths: `hermes_wire_instructions.first_name`/`last_name` (255) vs `hathor_client.first_name` (32), `dionysus_partner_sender_details.first_name` (150). Pick canonical.
10. Reconcile account-number column widths: `dionysus_currency_exchange_transactions.from_account`/`to_account` (255) vs `dionysus_international_transaction.client_account_number` (30). Pick canonical 30.
11. Reconcile `payment_identifier` widths: `dionysus_currency_exchange_transactions.payment_identifier` (255) vs `dionysus_international_transaction.payment_identifier` (60). Pick canonical 60.
12. Replace `feeType` String + `@NotBlank` in `BankingFeeRequest` with an enum + `@NotNull`.
13. Replace `direction` String + `@Pattern` with `Direction` enum.
14. Remove dead `@Pattern("^card$")` from `PaymentController.moveFund` path variable (service ignores it). Or wire it up properly to route to non-card methods.
15. Add `@Size` (≤ 50) and `@NotBlank` to `WirePaymentRequest.wireInstructionsId`.
