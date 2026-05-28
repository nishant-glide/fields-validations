# Set 8 — Coin / Crypto

6 endpoints under `CoinController` (admin-only — `@RequestMapping("admin/coin")`). All downstream calls go to the **coin-api artifact** (`com.glidetech.coin`) — 🔒 deferred since the source is not in this repo. We document gateway DTOs and the live DB schema, which is sufficient for the consistency pass.

Downstream surfaces:
- **coin-api artifact** — managed by `CoinManager` (🔒 not in repo)
- DB tables: `shu_coin_account`, `shu_coin_account_address`, `shu_coin_balances`, `shu_coin_operations`, `osiris_external_wallet` (for linked external wallets)

Mythology mapping confirmed:
- `shu_*` → **coin-api artifact** (NEW for this set; 12 tables) — service source not in repo

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`AddCoinAccountAddressRequest` has only one field, `List<String> networks`, with NO validation** | No `@NotNull`/`@NotEmpty`/`@Size` on the list. Empty list passes; null passes. | ❌ |
| 2 | **`CoinController.depositCoins` is missing `@Valid` on `@RequestBody StableCoinMintRequest`** | `@Valid` is present on `withdrawCoins` for the sibling Burn DTO. Inconsistent. | ❌ |
| 3 | All coin DTOs use **`String amount` with `@Pattern("^\\d+(\\.\\d{1,6})?$")`** rather than `BigDecimal @Positive @Digits` | ⚠ Works, but the string-amount + regex pattern allows leading zeros and silently parses; better Java pattern would be `BigDecimal` with `@Positive` + `@Digits(integer=...,fraction=6)`. The 6-decimal cap is reasonable for crypto (BTC = 8 decimals, USDC = 6). |
| 4 | `CoinAccountCreateRequest.networks` is `List<String>` with **no `@Size` on each element** and no `@NotEmpty` on the list | ⚠ |
| 5 | `CoinAccountCreateRequest.accountType` `@Pattern("^(COIN\|DIGITAL_COIN)$")` defaults to `"COIN"` — but the field has no `@NotBlank` | ⚠ Default applied at field initialization; partner can send `""` and Pattern will reject. OK. |
| 6 | `ExternalCoinAccountRequest`: `name`, `address`, `network` all `@NotBlank` ✅; but **no `@Size`** anywhere → DB `osiris_external_wallet.address` = varchar(255), `name` = varchar(255) | ⚠ |
| 7 | `StableCoinMintRequest`/`StableCoinBurnRequest` `symbol` and `network` fields are **NOT marked `@NotBlank`** — partner can send a mint request without symbol/network and only fail downstream | ❌ |
| 8 | `CoinTransferCreateRequest` is the **best-validated DTO** in Set 8 — `@NotBlank` on `symbol`, `amount`, `fromCoinAccount`, `toCoinAccount`, `network`, plus `@Pattern` on `amount` | ✅ |
| 9 | `shu_coin_balances.symbol` = **varchar(10)**, `shu_coin_account.symbol` = **varchar(20)** — same logical field, two widths | ❌ |
| 10 | `shu_coin_account_address.address` = **varchar(500)** but `osiris_external_wallet.address` = **varchar(255)** | ❌ Inconsistent for the same logical concept (a blockchain address). Some chains (Solana) use up to 44 chars; ETH = 42; XRP = 25-35. 500 is over-budget; 255 is fine. |
| 11 | `shu_coin_operations.fiat_currency` = varchar(4) — gateway DTOs don't expose `fiatCurrency` directly | ⚠ |
| 12 | `shu_coin_operations.transaction_hash` = varchar(100) — most chains' hashes are 64 hex chars (Ethereum) or 64 bytes (BTC) | ✅ Tight but OK. |
| 13 | `shu_coin_operations.symbol` = varchar(10) — same field is varchar(20) on `shu_coin_account` | ❌ |
| 14 | `shu_coin_operations.operation_id` = varchar(50) | Server-generated. |
| 15 | `osiris_external_wallet.tag` = varchar(200) — for memo/tag chains (XRP, XLM); reasonable | ✅ |
| 16 | `osiris_external_wallet.asset` = varchar(100) — partner sends `network` field but DB stores `asset`. Mapper-side rename. | ⚠ |
| 17 | `osiris_external_wallet.external_wallet_id` = varchar(20) (NOT NULL? — checked: it's nullable) — server-generated identifier | ✅ |
| 18 | `shu_coin_account.session_id` = varchar(255) — wider than 80 used elsewhere | ⚠ |
| 19 | `shu_coin_account.networks` and `requested_networks` are varchar(500) — comma-separated string instead of normalized table | ⚠ Design choice. |
| 20 | All `clientId` / `businessId` pairs across 6 DTOs: no XOR validation | ❌ |

---

## 1. `POST /admin/coin/accounts` — Create coin account

- **Controller:** `CoinController.createCoinAccount`
- **Request DTO (gateway):** `CoinAccountCreateRequest`
- **Service flow:** `CoinService.createCoinAccount` → `CoinManager.createAccount(...)` (coin-api artifact 🔒)
- **Persisted to:** `shu_coin_account` (+ `shu_coin_account_address` per network)

| # | Field | Type | Gateway | DB column (`shu_coin_account`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `name` | String | @NotBlank (no @Size) | `name` varchar(200) NOT NULL | ⚠ | |
| 2 | `clientId` | Long | (no validation) | `client_id` bigint | ⚠ | |
| 3 | `businessId` | Long | (no validation) | `business_id` bigint | ⚠ | |
| 4 | `networks` | List<String> | (no @NotEmpty, no per-element @Size) | `networks` varchar(500) (comma-separated) | ❌ | |
| 5 | `description` | String | (no @Size) | (not directly on `shu_coin_account`) | ⚠ | |
| 6 | `correlationId` | String | (no @Size) | `correlation_id` varchar(100) | ❌ | |
| 7 | `clientIpAddress` | String | (no @Size) | (not persisted on coin account) | ⚠ | |
| 8 | `symbol` | String | (no @NotBlank, no @Size) | `symbol` varchar(20) | ❌ | |
| 9 | `accountType` | String | @Pattern("^(COIN\|DIGITAL_COIN)$") default="COIN" | `account_type` varchar(20) NOT NULL | ✅ | |

---

## 2. `POST /admin/coin/accounts/link-external-wallet` — Register external coin wallet

- **Controller:** `CoinController.registerExternalCoinAccount`
- **Request DTO:** `ExternalCoinAccountRequest`
- **Service flow:** `CoinService.registerExternalCoinAccount` → `CoinManager.linkExternalWallet(...)` (🔒) — also persists `osiris_external_wallet`
- **Persisted to:** `osiris_external_wallet`

| # | Field | Type | Gateway | DB column (`osiris_external_wallet`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `name` | String | @NotBlank (no @Size) | `name` varchar(255) | ⚠ | |
| 2 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 3 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 4 | `address` | String | @NotBlank (no @Size, no @Pattern) | `address` varchar(255) | ❌ | Should validate per chain format (ETH = 42 hex, BTC = base58, etc.). |
| 5 | `network` | String | @NotBlank (no @Size) | mapped to `asset` varchar(100) | ⚠ | |
| 6 | `description` | String | (no @Size) | (not on external wallet) | ⚠ | |
| 7 | `correlationId` | String | (no @Size) | — | ⚠ | |
| 8 | `clientIpAddress` | String | (no @Size) | — | ⚠ | |

---

## 3. `POST /admin/coin/accounts/{coinAccountNumber}/address` — Add address to coin account

- **Controller:** `CoinController.addAddressToCoinAccount`
- **Request DTO:** `AddCoinAccountAddressRequest`
- **Service flow:** `CoinService.addAddressToCoinAccount` → `CoinManager.addAddress(...)` (🔒) → fetches wallet addresses from PrimeVault → INSERTs into `shu_coin_account_address`

| # | Field | Type | Gateway | DB column (`shu_coin_account_address`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `networks` | List<String> | (no validation) | `network` varchar(150) NOT NULL (per-row) | ❌ | List can be `null` or empty — controller passes through unchecked. Each element becomes a row. |

Path variable: `coinAccountNumber` — links to `shu_coin_account.account_id` varchar(30).

---

## 4. `POST /admin/coin/deposit` (alias `/admin/coin/mint`) — Deposit (mint) coins

- **Controller:** `CoinController.depositCoins`
- **Request DTO:** `StableCoinMintRequest`
- **⚠ Controller missing `@Valid`** on the @RequestBody — DTO validation does not fire.
- **Service flow:** `CoinService.depositCoins` → `CoinManager.mint(...)` (🔒)
- **Persisted to:** new row in `shu_coin_operations` (operation_type = MINT/DEPOSIT)

| # | Field | Type | Gateway | DB column (`shu_coin_operations`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `symbol` | String | (no @NotBlank, no @Size) | `symbol` varchar(10) | ❌ | |
| 2 | `network` | String | (no @NotBlank, no @Size) | `network` varchar(20) NOT NULL | ❌ | |
| 3 | `amount` | String | @NotBlank @Pattern("^\\d+(\\.\\d{1,6})?$") | `amount` numeric | ✅ | Pattern is reasonable. |
| 4 | `coinAccountNumber` | String | @NotBlank (no @Size) | `coin_account_number` varchar(50) | ⚠ | |
| 5 | `primaryAccountNumber` | String | (no validation) | (not directly on `shu_coin_operations`) | ⚠ | Fiat source account. |
| 6 | `clientIpAddress` | String | (no validation) | — | ⚠ | |
| 7 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 8 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 9 | `correlationId` | String | (no @Size) | (not on `shu_coin_operations`) | ⚠ | |

**Critical:** `@Valid` is missing on the controller — even the `@NotBlank` constraints won't fire.

---

## 5. `POST /admin/coin/withdraw` (alias `/admin/coin/burn`) — Withdraw (burn) coins

- **Controller:** `CoinController.withdrawCoins`
- **Request DTO:** `StableCoinBurnRequest`
- Controller has `@Valid` ✅ (unlike depositCoins).
- **Service flow:** `CoinService.withdrawCoins` → `CoinManager.burn(...)` (🔒)
- **Persisted to:** `shu_coin_operations` (operation_type = BURN/WITHDRAW)

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `symbol` | String | (no @NotBlank, no @Size) | `symbol` varchar(10) | ❌ | Same as deposit gap. |
| 2 | `network` | String | (no @NotBlank, no @Size) | `network` varchar(20) NOT NULL | ❌ | |
| 3 | `amount` | String | @NotBlank @Pattern("^\\d+(\\.\\d{1,6})?$") | `amount` numeric | ✅ | |
| 4 | `coinAccountNumber` | String | @NotBlank (no @Size) | `coin_account_number` varchar(50) | ⚠ | |
| 5 | `primaryAccountNumber` | String | (no validation) | — | ⚠ | Fiat destination account. |
| 6 | `clientIpAddress` | String | (no validation) | — | ⚠ | |
| 7 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 8 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 9 | `correlationId` | String | (no @Size) | — | ⚠ | |

---

## 6. `POST /admin/coin/transfers` — Transfer coins (between coin accounts)

- **Controller:** `CoinController.createTransfer`
- **Request DTO:** `CoinTransferCreateRequest`
- **Service flow:** `CoinService.createTransfer` → `CoinManager.transfer(...)` (🔒)
- **Persisted to:** `shu_coin_operations` (operation_type = TRANSFER)

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `symbol` | String | @NotBlank (no @Size) | `symbol` varchar(10) | ✅ | |
| 2 | `amount` | String | @NotBlank @Pattern("^\\d+(\\.\\d{1,6})?$") | `amount` numeric | ✅ | |
| 3 | `fromCoinAccount` | String | @NotBlank (no @Size) | `coin_account_number` varchar(50) | ✅ | |
| 4 | `toCoinAccount` | String | @NotBlank (no @Size) | (destination — server-resolved or as parameter on dest) | ✅ | |
| 5 | `network` | String | @NotBlank (no @Size) | `network` varchar(20) NOT NULL | ✅ | |
| 6 | `description` | String | (no @Size) | — | ⚠ | |
| 7 | `clientIpAddress` | String | (no validation) | — | ⚠ | |
| 8 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 9 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 10 | `correlationId` | String | (no @Size) | — | ⚠ | |

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /coin/accounts` | `networks` list — DB stores comma-separated `varchar(500)` | ✅ |
| `POST /coin/accounts/link-external-wallet` | `address` no @Pattern → DB `varchar(255)` | ✅ |
| `POST /coin/accounts/{n}/address` | `networks` list no validation | ✅ |
| `POST /coin/deposit` | Controller missing `@Valid` | ✅ Confirmed inconsistency vs `/withdraw`. |
| `POST /coin/withdraw` | `symbol`/`network` no @NotBlank | ✅ |
| `POST /coin/transfers` | All required fields properly `@NotBlank` | ✅ Best DTO. |
| Schema drift | `shu_coin_account.symbol` (20) vs `shu_coin_operations.symbol` (10) vs `shu_coin_balances.symbol` (10) | ✅ |
| `shu_coin_account_address.address` varchar(500) vs `osiris_external_wallet.address` varchar(255) | ✅ |

---

## Recommended next-step actions

1. **Critical:** Add `@Valid` to `CoinController.depositCoins`'s `@RequestBody StableCoinMintRequest request` (sibling `withdrawCoins` has it).
2. Add `@NotBlank` to `StableCoinMintRequest.symbol`/`network` and `StableCoinBurnRequest.symbol`/`network`. Currently the only mint-required fields are `amount` and `coinAccountNumber`.
3. Add `@Size(min=2,max=20)` and `@Pattern("[A-Z0-9]+")` to `symbol` fields across all 6 DTOs (and reconcile DB widths — `shu_coin_balances.symbol`=10 vs `shu_coin_account.symbol`=20).
4. Add `@Size` to `AddCoinAccountAddressRequest.networks` (`@NotEmpty @Size(max=20)`) and per-element `@Pattern`.
5. Add chain-specific address-format validators or generic `@Size(max=255)` `@Pattern` to `ExternalCoinAccountRequest.address`.
6. Reconcile `address` column width: `shu_coin_account_address.address`=500 vs `osiris_external_wallet.address`=255. Pick canonical 255.
7. Reconcile `symbol` width across the three `shu_*` tables (10 vs 20).
8. Replace `String amount` + `@Pattern` with `BigDecimal amount` + `@Positive @Digits(integer=20,fraction=6)` for type safety. (Current pattern allows leading zeros.)
9. Add XOR class-level constraint on `clientId`/`businessId` for all 6 DTOs.
10. Cap `shu_coin_account.networks`/`requested_networks` varchar(500) usage by normalizing into a separate `shu_coin_account_network` table (current comma-separated string is hard to query).
11. Narrow `shu_coin_account.session_id` from varchar(255) to varchar(80) to match other tables.
12. Add `@NotBlank` to `CoinAccountCreateRequest.symbol` (since it determines the chain) — currently only `accountType` defaults to "COIN".
13. Tighten `shu_coin_operations.symbol` varchar(10) — if you support 4-char symbols, fine; otherwise widen to match `shu_coin_account.symbol` varchar(20).
