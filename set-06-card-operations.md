# Set 6 — Card Operations & Transfers

8 endpoints across `CardController` (4), `CardTokenController` (1), and `CardTransferController` (3). Cards-related operations split between **glide-cardpymt** (locally available) and the **cards-api artifact** (🔒 deferred — JARs not in repo).

Downstream surfaces:
- **glide-cardpymt** — `horus_card_account`, `horus_card_txn_details`, `horus_card_activity`, `horus_card_attributes` (in-repo)
- **cards-api artifact** (`com.glidetech.cards`) — `CardApiManager` (activate/pin/replace/status DTOs) — 🔒 deferred
- **glide-client-account** / cards module — `ExternalCardJpa` for `link-card` and `delete-card`

Mythology mapping confirmed:
- `horus_*` → **glide-cardpymt** (NEW for this set; 14 tables)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`CardPinChangeRequestDto` has ZERO validation** | No `@NotNull` on `cardToken`, `newPin`, `cvv2`, `last4Digits`. PIN values are unbounded — a partner could send a 10K-char `newPin`. | ❌ |
| 2 | **`CardReplacementRequestDto` has ZERO validation** | No `@NotNull` on `cardToken`, `reason`. Both `sendPhysicalCard` and `generateNewCardNumber` are primitive booleans (default false — silently). | ❌ |
| 3 | **`CardStatusChangeRequestDto` has ZERO validation** | No `@NotNull` on `cardToken`, no `@NotNull` on `newStatus` enum, no `@NotBlank` on `reason`. | ❌ |
| 4 | `CardActivationRequestDto`: `cardToken` `@NotNull`, but **`@JsonFormat(pattern = "YYYY-MM")` on `cardExpiryDate`** uses `YYYY` (week-of-year) instead of `yyyy` (year) | Java SimpleDateFormat treats `YYYY` as week-year. Output strings near year boundaries will be wrong. | ❌ Bug |
| 5 | `CardActivationRequestDto.last4Digits` `@Size(max=4)` (no `@Pattern("\\d{4}")`) | Allows non-digit content up to 4 chars. | ⚠ |
| 6 | `CardActivationRequestDto.cvv2` `@Size(max=3)` (no `@Pattern("\\d{3}")`) | Same. Also, 4-digit Amex CVV won't fit (but Visa/MC contract only). | ⚠ |
| 7 | `CardTransferRequestDto` validation is **the best in Set 6** — `@NotNull` on `type`/`amount`, `@NotBlank` on `cardToken`/`currency`, `@Positive` on `amount`. **Still missing `@Size` everywhere**, no `@Pattern` on `currency` (ISO-4217). | ✅/⚠ |
| 8 | `CaptureTransactionRequestDto.transactionId` `@NotBlank` (no `@Size`); DB `transaction_id` = varchar(50) | ⚠ |
| 9 | **`CardTransactionQueryRequest` (downstream-defined in glide-cardpymt) is accepted directly by the gateway controller** — no gateway DTO | Same anti-pattern as Sets 1, 4. Has default `fromDate`/`toDate` of "today minus 1 month" / "today" — could surprise partners who don't pass them. | ⚠ |
| 10 | `horus_card_account.card_token` = **varchar(255)**, `account_id` = **varchar(255)** | Card token formats vary by provider; 255 is liberal. Inconsistent with `horus_card_txn_details.card_token` = varchar(100). | ❌ Same logical field two widths. |
| 11 | `horus_card_account.last4_digit` = varchar(4) | ✅ tight. Note column name is `last4_digit` (singular); other tables use `last4` / `card_last4`. | ⚠ Naming drift. |
| 12 | `horus_card_account.status` enum-as-string varchar(20) NOT NULL — `@Enumerated(EnumType.STRING)` ✅ | | ✅ |
| 13 | `horus_card_txn_details.transaction_id` (50), `transaction_gid` (50) — vs `dionysus_international_transaction.payment_identifier` (60), `dionysus_international_transaction.transaction_gid` (60) | ❌ Cross-table transaction-id widths inconsistent (50 vs 60). |
| 14 | `horus_card_txn_details.description` = varchar(500), but gateway DTOs have **no `@Size` on `description`** | ❌ |
| 15 | `horus_card_txn_details.amount` is `numeric` with **no precision/scale set** in the schema | Entity declares `precision=19, scale=4` but DB just shows `numeric`. Hibernate-DDL not applied via migration. | ⚠ Schema drift. |
| 16 | **`VaultSessionRequest` for card link has same gap as Set 3** — zero validation on `clientId`/`businessId` | ❌ |
| 17 | `CardPinChangeRequestDto.newPin` is a **plaintext String** with no validation | ⚠ Security: should be at least `@Size(min=4,max=12)` and ideally tokenized/encrypted in transit. Beyond this doc's scope but worth noting. |
| 18 | `CardActivationRequestDto.last4Digits`, `cvv2` are declared `public` (not `private`) on the DTO | ⚠ Code style — Lombok `@Data` still generates getters/setters; not a runtime bug. |
| 19 | All 4 `CardController` mutating endpoints go through `CardApiManager` (cards-api artifact, **🔒 deferred** — downstream DTO sources not in repo) | Cannot trace downstream length/validation. |
| 20 | `CardTransferController` endpoints go through `CardTransferManager` (glide-cardpymt) — fully traceable. | ✅ |

---

## 1. `POST /admin/cards/activate` — Activate card

- **Controller:** `CardController.activateCard`
- **Request DTO (gateway):** `CardActivationRequestDto`
- **Service flow:** `CardService.activateCard` → `CardApiManager.activateCard(...)` (**cards-api artifact** — 🔒 deferred)
- **Downstream DTO:** 🔒 deferred (cards-api JAR — `CardActivationRequest` likely)
- **Persisted to:** `horus_card_account` (state transition via downstream service in artifact)

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | 🔒 | `horus_card_account.client_id` bigint | ⚠ | |
| 2 | `businessId` | Long | (no validation) | 🔒 | `horus_card_account.business_id` bigint | ⚠ | |
| 3 | `cardToken` | String | @NotNull (no @Size) | 🔒 | `horus_card_account.card_token` varchar(255) · `horus_card_txn_details.card_token` varchar(100) | ❌ | Cross-table inconsistency. |
| 4 | `last4Digits` | String | @Size(max=4) (no @Pattern) | 🔒 | `horus_card_account.last4_digit` varchar(4) · `horus_card_activity.card_last4` varchar(10) | ⚠ | No `@Pattern("\\d{4}")`. |
| 5 | `cardExpiryDate` | String | `@JsonFormat(pattern="YYYY-MM")` | 🔒 | (not stored on these tables; provider-side) | ❌ | `YYYY` is week-year (Java SimpleDateFormat). Should be `yyyy-MM`. |
| 6 | `cvv2` | String | @Size(max=3) (no @Pattern) | 🔒 | (not persisted — sent to provider only) | ⚠ | No `@Pattern("\\d{3,4}")`. |

---

## 2. `POST /admin/cards/pin` — Change card PIN

- **Controller:** `CardController.changePin`
- **Request DTO (gateway):** `CardPinChangeRequestDto`
- **Service flow:** `CardService.changePin` → `CardApiManager.changePin(...)` (🔒 deferred)
- **Persisted to:** `horus_card_activity` (audit row)

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | 🔒 | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | 🔒 | `business_id` | ⚠ | |
| 3 | `correlationId` | String | (no validation) | 🔒 | — | ❌ | |
| 4 | `cardToken` | String | (no validation) | 🔒 | `horus_card_activity.card_token` varchar(255) | ❌ | Missing `@NotNull`. |
| 5 | `newPin` | String | (no validation) | 🔒 | (not persisted plaintext) | ❌ | **No `@Size`, no `@Pattern("\\d{4,12}")`** — sensitive field unbounded. |
| 6 | `last4Digits` | String | (no validation) | 🔒 | — | ⚠ | |
| 7 | `cvv2` | String | (no validation) | 🔒 | — | ⚠ | |
| 8 | `cardExpiryDate` | String | (no validation, no @JsonFormat) | 🔒 | — | ⚠ | |
| 9 | `operation` | enum `PinOperation` | (no @NotNull) | 🔒 | — | ⚠ | |

---

## 3. `POST /admin/cards/replace` — Replace card

- **Controller:** `CardController.replaceCard`
- **Request DTO (gateway):** `CardReplacementRequestDto`
- **Service flow:** `CardService.replaceCard` → `CardApiManager.replaceCard(...)` (🔒 deferred)
- **Persisted to:** new row in `horus_card_account` (new card token), prior row marked inactive

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | 🔒 | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | 🔒 | `business_id` | ⚠ | |
| 3 | `correlationId` | String | (no validation) | 🔒 | — | ⚠ | |
| 4 | `cardToken` | String | (no validation) | 🔒 | `card_token` varchar(255) | ❌ | Should be `@NotBlank`. |
| 5 | `reason` | String | (no validation) | 🔒 | (audit only) | ❌ | |
| 6 | `sendPhysicalCard` | boolean | (primitive; default false) | 🔒 | — | ⚠ | |
| 7 | `generateNewCardNumber` | boolean | (primitive; default false) | 🔒 | — | ⚠ | |

---

## 4. `POST /admin/cards/status` — Change card status

- **Controller:** `CardController.changeCardStatus`
- **Request DTO (gateway):** `CardStatusChangeRequestDto`
- **Service flow:** `CardService.changeCardStatus` → `CardApiManager.changeCardStatus(...)` (🔒 deferred)
- **Persisted to:** `horus_card_account.status` UPDATE

| # | Field | Type | Gateway DTO | Downstream | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | 🔒 | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | 🔒 | `business_id` | ⚠ | |
| 3 | `correlationId` | String | (no validation) | 🔒 | — | ⚠ | |
| 4 | `cardToken` | String | (no validation) | 🔒 | `card_token` varchar(255) | ❌ | |
| 5 | `newStatus` | enum `CardStatus` | (no @NotNull) | 🔒 | `horus_card_account.status` varchar(20) NOT NULL | ❌ | |
| 6 | `reason` | String | (no validation) | 🔒 | (audit) | ❌ | |

---

## 5. `POST /admin/cards/link-card` — Generate vault session token for card linking

- **Controller:** `CardTokenController.linkCard`
- **Request DTO (gateway):** `VaultSessionRequest`
- **Service flow:** `CardTokenService.createVaultSession` → service-level access validation → outbound to card vault provider (Cybersource/Tabapay)
- **Persisted to:** No DB row at this step; subsequent card registration creates `horus_card_attributes` + `horus_card_account`

| # | Field | Type | Gateway DTO | Notes |
|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | Same gap as `BeneficiaryController.linkCard` in Set 3. |
| 2 | `businessId` | Long | (no validation) | XOR is service-level. |

**Same DTO as Beneficiary linkCard.** Controller has `@Valid` here but DTO has no constraints, so it's a no-op.

---

## 6. `POST /admin/card-transfers/transfer` — Execute card transfer (AFT / OCT)

- **Controller:** `CardTransferController.transfer`
- **Request DTO (gateway):** `CardTransferRequestDto`
- **Service flow:** `CardTransferService.transfer` → `CardTransferMapper.toMap` → `CardTransferManager.transfer(...)` (**glide-cardpymt**)
- **Downstream DTO:** `com.glidetech.cardpymt.api.v1.domain.transfer.CardTransferRequest` (similar shape)
- **Persisted to:** `horus_card_txn_details`

| # | Field | Type | Gateway DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `type` | enum `CardTransferType` (PUSH/PULL) | @NotNull | `horus_card_txn_details.transaction_type` varchar(10) | ✅ | |
| 2 | `cardToken` | String | @NotBlank (no @Size) | `card_token` varchar(100) NOT NULL | ⚠ | No length cap at gateway; DB caps at 100. |
| 3 | `amount` | BigDecimal | @NotNull @Positive | `amount` numeric (no precision/scale set in DB) | ⚠ | Best amount validation. DB column lacks precision/scale per migration. |
| 4 | `primaryAccountNumber` | String | (no validation) | `primary_account` varchar(50) NOT NULL | ❌ | No `@NotBlank` or `@Size`. |
| 5 | `currency` | String | @NotBlank (no @Size, no @Pattern) | `currency_code` varchar(3) (default "USD") | ❌ | Should be `@Size(min=3,max=3) @Pattern("[A-Z]{3}")`. |
| 6 | `clientId` | Long | (no validation) | `client_id` bigint | ⚠ | |
| 7 | `businessId` | Long | (no validation) | `business_id` bigint | ⚠ | |
| 8 | `merchantReference` | String | (no validation) | (not persisted as-is — typically maps to `retrieval_reference_number` varchar(50)) | ❌ | |
| 9 | `description` | String | (no validation) | `description` varchar(500) | ❌ | |
| 10 | `correlationId` | String | (no validation) | `correlation_id` varchar(100) | ❌ | |
| 11 | `capture` | Boolean | (no @NotNull; default false) | derived from `transaction_status` | ⚠ | |

---

## 7. `POST /admin/card-transfers/capture` — Capture authorized transaction

- **Controller:** `CardTransferController.captureTransaction`
- **Request DTO (gateway):** `CaptureTransactionRequestDto`
- **Service flow:** `CardTransferService.captureTransaction` → `CardTransferManager.captureTransaction(...)` (**glide-cardpymt**)
- **Persisted to:** UPDATE on `horus_card_txn_details` (status transition)

| # | Field | Type | Gateway DTO | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `transactionId` | String | @NotBlank (no @Size) | `horus_card_txn_details.transaction_id` varchar(50) | ⚠ | |
| 2 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 3 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |

---

## 8. `POST /admin/card-transfers/search` — Search card transactions

- **Controller:** `CardTransferController.searchCardTransactions`
- **Request DTO (gateway = downstream):** `com.glidetech.cardpymt.api.v1.domain.transfer.CardTransactionQueryRequest`
  - **The gateway controller accepts the downstream DTO directly** — no gateway-side wrapper.
- **Service flow:** `CardTransferService.searchCardTransactions` → `CardTransferManager.searchCardTransactions(...)` (**glide-cardpymt**)
- **Persisted to:** Read-only query against `horus_card_txn_details`

| # | Field | Type | Gateway/Downstream DTO | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `fromDate` | LocalDate | default = today − 1 month | ⚠ | Default could surprise partners. |
| 2 | `toDate` | LocalDate | default = today | ⚠ | |
| 3 | `daysAgo` | Long | (no validation) | ⚠ | |
| 4 | `pageNumber` | Integer | default 1 (no `@Min(1)`) | ⚠ | |
| 5 | `pageSize` | Integer | default 50 (no `@Max`) | ❌ | Partner can send `pageSize=1000000`. |
| 6 | `filter` | nested object | (no validation) | ⚠ | Free-form search criteria; each filter sub-field is `@JsonProperty` only. |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /cards/activate` | `cardExpiryDate` `@JsonFormat("YYYY-MM")` (week-year typo) | ✅ Confirmed bug. |
| `POST /cards/pin` | All 9 fields unvalidated; `newPin` plaintext | ✅ Confirmed. |
| `POST /cards/replace` | `cardToken` no @NotNull | ✅ |
| `POST /cards/status` | `newStatus` enum no @NotNull | ✅ |
| `POST /cards/link-card` | VaultSessionRequest reused from Set 3, zero validation | ✅ |
| `POST /card-transfers/transfer` | `cardToken` gateway @NotBlank, DB varchar(100) — `horus_card_account.card_token` is varchar(255). Cross-table mismatch. | ✅ |
| `POST /card-transfers/capture` | `transactionId` @NotBlank, DB varchar(50). No `@Size`. | ✅ |
| `POST /card-transfers/search` | Gateway accepts downstream DTO directly | ✅ |

---

## Recommended next-step actions

1. Fix `CardActivationRequestDto.cardExpiryDate` `@JsonFormat(pattern="YYYY-MM")` → `pattern="yyyy-MM"`. The `YYYY` token is week-year in Java.
2. Add `@Pattern("\\d{4}")` to `last4Digits` and `@Pattern("\\d{3,4}")` to `cvv2` at gateway.
3. Add `@NotBlank` + `@Size(max=255)` to `cardToken` on every Card*RequestDto (currently missing on Pin/Replace/Status).
4. Add `@NotNull` to `CardPinChangeRequestDto.operation`, `CardStatusChangeRequestDto.newStatus`, `CardReplacementRequestDto.reason`.
5. Add `@Size(min=4,max=12)` and `@Pattern("\\d+")` to `CardPinChangeRequestDto.newPin`. Strongly consider routing PIN changes through a tokenized vault session (same pattern as `link-card`) rather than gateway-accepted plaintext.
6. Reconcile `card_token` width across `horus_card_account` (varchar 255) and `horus_card_txn_details` (varchar 100). Pick one canonical width.
7. Add `@Size` to `CardTransferRequestDto.description` (gateway → DB varchar 500), `correlationId` (→ DB varchar 100), `merchantReference`.
8. Add `@Pattern("[A-Z]{3}")` and `@Size(min=3,max=3)` to `CardTransferRequestDto.currency`.
9. Add `@NotBlank` to `CardTransferRequestDto.primaryAccountNumber` (currently optional).
10. Introduce a gateway-layer wrapper DTO for `CardTransactionQueryRequest` (currently leaks downstream contract — same as Sets 1, 4).
11. Add `@Min(1)` to `pageNumber` and `@Max(200)` to `pageSize` on `CardTransactionQueryRequest`.
12. Set `precision=19, scale=4` on the `amount` column in a Flyway migration so it matches the entity (currently DB shows just `numeric`).
13. Rename `horus_card_account.last4_digit` → `last4_digit_s`? or align with `card_last4` used in `horus_card_activity`/`horus_card_attributes`. Naming drift.
14. Decide whether to inspect the cards-api artifact (out of scope per the user's directive); the four mutating cards/* endpoints have unverified downstream DTOs.
