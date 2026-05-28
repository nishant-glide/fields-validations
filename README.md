# API Field Validation Inventory

End-to-end inventory of every input field on every input-bearing API exposed by **glide-api-gateway**, tracing each field through three layers:

1. **Gateway DTO** — `glide-api-gateway/api/.../v1/domain/*` (Bean Validation: `@Size`, `@Pattern`, `@NotBlank`, `@Past`, `@Min`/`@Max`, `@JsonFormat`).
2. **Downstream DTO** — DTO sent over Feign to the relevant domain service (`glide-business`, `glide-client`, `glide-client-account`, `glide-remittance`, `glide-cardpymt`, `glide-extpaymt`, `glide-prospects`, `glide-user`, `glide-webhook`). Third-party artifact DTOs (`cards-api`, `coin-api`, `trading-api`, `transaction-api`, `deposit-api`, `portal-api`) are flagged as **deferred — source not in repo**.
3. **DB column** — JPA `@Column(name, length, nullable)` and the authoritative `information_schema.columns` row in Postgres `bivo_sandbox` (schema `public`).

This document exists so we can produce a consistency pass: where layers disagree, we will align them in a later change set.

---

## Purpose

> "We have to make validations consistent across the db to services to downstream services involved."

For every field, we will be able to answer:
- What format / length is accepted at the gateway?
- What format / length does the downstream service expect?
- What length is the column physically able to store?
- Are these three answers the same? If not, where's the gap?

---

## Methodology

**Per endpoint:**
1. Read the controller method → identify request DTO.
2. Open the gateway DTO; record every field's annotations and Java type.
3. Walk controller → service → mapper to identify the actual downstream DTO sent over Feign.
4. Open the downstream DTO (in the relevant `<module>/api/`); record annotations.
5. Identify the persisted entity (`<module>/service/.../entity/`); record `@Column(...)`.
6. Query live DB:
   ```sql
   SELECT column_name, data_type, character_maximum_length,
          numeric_precision, numeric_scale, is_nullable, column_default
   FROM information_schema.columns
   WHERE table_schema = 'public' AND table_name = '<table>'
   ORDER BY ordinal_position;
   ```
7. Render the per-field table; mark each row `✅` / `❌` / `⚠`.

**Read-only.** No code edits. No DB writes.

---

## Legend

| Symbol | Meaning |
|---|---|
| ✅ aligned | Gateway, downstream, and DB all agree on a length/format that's enforceable. |
| ⚠ partial | One layer enforces, others don't (e.g. gateway has `@Pattern` but downstream and DB do not) — not strictly wrong, but inconsistent. |
| ❌ inconsistent | Layers actively disagree on length/type — risk of truncation, 500s, or downstream rejects. |
| 🔒 deferred | Downstream DTO lives in a third-party artifact JAR not present in this repo. Will be inspected later if we decide to decompile from the Gradle cache. |
| — n/a | Field not present at this layer (e.g. enriched server-side, derived from path param). |

---

## Database

- Host: `bivotech-core-postgres.cluster-cx6i6a4kahit.ap-south-1.rds.amazonaws.com`
- DB: `bivo_sandbox`, schema: `public`
- Table prefix convention (per Flyway scripts across modules — mythological names):

| Prefix | Likely service | Table count |
|---|---|---:|
| `dionysus_` | **glide-remittance** (beneficiary, transactions, FX, vendor mappings) | 94 |
| `hathor_` | **glide-client** (Client, Address, Signzy, Ofac, etc.) | 41 |
| `osiris_` | **glide-client-account** (ClientAccount, SpendingAccount, BusinessAccount, ExternalAccount) | 36 |
| `hades_` | **trading-api artifact** (AUM, sweep, positions, securities, stocks, T-bills) | 33 |
| `cronus_` | **glide-business** (BusinessProfile, BusinessAdditionalInfo) | 24 |
| `demeter_` | **glide-webhook** (outbound config, event log, vendor-specific webhook events) | 23 |
| `hermes_` | **transaction-api / wire-instructions** (wire transactions, wire instructions, auto-approval config) | 23 |
| `iris_` | **glide-portal** (document req config, document requirements, support-user, dynamic templates, investigations) | 20 |
| `khonsu_` | **glide-prospects** (Prospect, BusinessProspect, BuProspectOwner) | 7 |
| `zeus_` | **glide-extpaymt** (ACH txn details, ACH file info) | 9 |
| `shu_` | **coin-api artifact** (coin account, balances, operations) | 12 |
| `horus_` | **glide-cardpymt** (card account, card txn details, card attributes, card activity) | 14 |
| `helios_` | **external payment / integration** (external payment, beneficiary account, provider integration logs) | 13 |
| `hera_` | (TBD) | 12 |
| `hestia_` | **glide-api-gateway (local)** (company profile, company product access, api user) | 4 |
| (others) | misc | — |

Prefix → service mapping is filled in per-set as the relevant Flyway scripts are opened.

Read-only query helper:
```bash
PGPASSWORD='***' /opt/homebrew/opt/libpq/bin/psql \
  -h bivotech-core-postgres.cluster-cx6i6a4kahit.ap-south-1.rds.amazonaws.com \
  -U nishant -d bivo_sandbox \
  -c "SELECT column_name, data_type, character_maximum_length, is_nullable
      FROM information_schema.columns
      WHERE table_schema='public' AND table_name='<table>' ORDER BY ordinal_position;"
```

---

## Set Index

| # | File | Domain | Endpoints | Status |
|---|---|---|---:|---|
| 1 | [set-01-onboarding-individual.md](./set-01-onboarding-individual.md) | Individual onboarding (account create / KYC / identity) | 7 | ✅ done |
| 2 | [set-02-onboarding-business.md](./set-02-onboarding-business.md) | Business onboarding (prospect, owners, account) | 9 | ✅ done |
| 3 | [set-03-beneficiary.md](./set-03-beneficiary.md) | Beneficiary CRUD + account link | 5 | ✅ done |
| 4 | [set-04-sender-remittance.md](./set-04-sender-remittance.md) | Sender + remittance initiation + pricing | 5 | ✅ done |
| 5 | [set-05-ach-external-account.md](./set-05-ach-external-account.md) | ACH movement + Plaid / external account linking | 8 | ✅ done |
| 6 | [set-06-card-operations.md](./set-06-card-operations.md) | Card activate / PIN / replace / status + transfers | 8 | ✅ done |
| 7 | [set-07-secure-card.md](./set-07-secure-card.md) | Secure card (credit) account, payment, statement | 7 | ✅ done |
| 8 | [set-08-coin-crypto.md](./set-08-coin-crypto.md) | Coin account, deposit/withdraw, transfers | 6 | ✅ done |
| 9 | [set-09-wire-currency-misc.md](./set-09-wire-currency-misc.md) | Wire, currency conversion, banking fee, trading, funding | 11 | ✅ done |
| 10 | [set-10-partner-webhook-internal.md](./set-10-partner-webhook-internal.md) | Internal partner registration, document upload, webhook | 7 | ✅ done |
| — | — | **Total** | **73** | — |

---

## Per-API Section Template

Each endpoint within a set file follows this layout:

```
### <METHOD> <path-admin> | <path-external>
**Controller:** <Controller>.<method>
**Request DTO (gateway):** <ClassName>
**Service / Mapper:** <Service>.<method> → <Mapper>.<method>
**Downstream call:** <Manager>.<method>  (module: <module>)
**Downstream DTO:** <ClassName>  (or 🔒 deferred)
**Persisted entity / table:** <Entity> / <public.tablename>

| # | Field | Type | Gateway DTO (validation) | Downstream DTO (validation) | DB column (type / length / null) | Aligned | Notes |
|---|---|---|---|---|---|---|---|
| 1 | businessName | String | @NotBlank @Size(max=100) | @Size(max=255) | varchar(120) NOT NULL | ❌ | Pick one max — recommend 120 |
| 2 | ein | String | @Pattern(\d{9}) | none | varchar(20) NOT NULL | ⚠ | Add @Pattern downstream |
```

A **Set-level summary** at the top of each set file calls out the headline gaps so a reviewer doesn't have to read every row.

---

## Exclusions

The following endpoints are intentionally **not** in scope (none have input fields that need cross-layer alignment):
- All pure `GET` endpoints across all 28 controllers.
- `DELETE`-by-id endpoints (only path-param UUIDs).
- `TestSupportAPIController` — sandbox/test-only.
- `InternalServiceController` — internal GET-only lookups.
- `StatementController` / `AccountStatementController` — file downloads.
- `ExternalAccountController.createLinkToken` — no body.
- `BusinessAccountController.acceptBusinessAgreement` — no body.
- `BusinessAccountController.addCurrencyAccountToBusiness` / `AccountController.addCurrencyAccountToClient` / `AccountController.createAccountForClient` — query/path-param only, no body.
