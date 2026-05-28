# Set 10 — Partner Registration / Webhook / Document

7 endpoints across 3 controllers:
- `InternalPartnerIntegrationController` (4): partner register, portal-user create, partner profile update, enable product. Path prefix `/internal/partner` — service-to-service only.
- `DocumentController` (2): list document requirements, upload document (multipart).
- `WebhookController` (1): search webhook events.

Downstream surfaces:
- **api-gateway-local** (yes — same service) — `hestia_company_profile`, `hestia_company_product_access`, `hestia_company_api_user`. These are the api-gateway's own JPA entities.
- **glide-portal** — `iris_document_req_config`, `iris_document_requirements`
- **glide-webhook** / glide-portal-relayed — `demeter_webhook_outbound_configuration`, `demeter_webhook_outbound_event_log`
- **glide-client** — `KycProviderConfig` and KYC waterfall steps may be referenced by the partner-registration flow

Mythology mapping confirmed:
- `hestia_*` → **api-gateway's own DB module** (company profile / company product access / api user — 4 tables). NEW for this set.
- `demeter_*` → **glide-webhook** (23 tables — confirmed in Set 5 hint). NEW for this set.
- `iris_*` → glide-portal (already confirmed in Set 1)

---

## Headline findings (consolidated)

| # | Field / Issue | Layer divergence | Risk |
|---|---|---|---|
| 1 | **`InternalPartnerIntegrationController.registerCompany` missing `@Valid`** on `@RequestBody`. DTO `CompanyRegistrationRequest` has **zero validation** anyway. | ❌ |
| 2 | **`InternalPartnerIntegrationController.createPartnerPortalUser` missing `@Valid`** on `@RequestBody`. DTO has `@NotNull` on `companyId`, `email`, `clientId`, `groups`. None will fire. | ❌ |
| 3 | **`InternalPartnerIntegrationController.updateCompanyProfile` missing `@Valid`** on `@RequestBody`. DTO has zero validation. | ❌ |
| 4 | **`InternalPartnerIntegrationController.enableProduct` missing `@Valid`** on `@RequestBody`. DTO has `@NotNull` on `productType` only. | ❌ |
| 5 | `CompanyRegistrationRequest` has **zero validation** across 12 fields | ❌ |
| 6 | `UpdateCompanyRequest` has **zero validation** across 9 fields | ❌ |
| 7 | `PartnerPortalUserRequest`: `@NotNull` on `companyId`, `email`, `clientId`, `groups` — but **no `@Email` on email, no `@Size` on any string** | ⚠ |
| 8 | `EnableProductRequest`: `@NotNull` on `productType` enum ✅; `Currency` and `provider` no `@Size`/`@Pattern` | ⚠ |
| 9 | `DocumentRequirementListRequest` has **zero validation** across 6 fields | ❌ Service-level validation expected. |
| 10 | `WebhookEventSearchRequest`: `@Min(0)` on `page`, `@Min(1)` on `size` — but **no `@Max` on size** (could request 1M results) | ⚠ |
| 11 | `hestia_company_profile.name` varchar(100) — `CompanyRegistrationRequest.name` no `@Size` | ❌ |
| 12 | `hestia_company_profile.support_email` varchar(100) (no `@Email` anywhere) | ❌ |
| 13 | `hestia_company_profile.ip_address` = varchar(**255**) — but IPv6 maxes at 45 chars. 5× over-budget. Inconsistent with `khonsu_prospect.client_ip_address` (50) | ❌ |
| 14 | `hestia_company_profile.address` = **text** (unbounded) | ⚠ Reasonable for partner business addresses. |
| 15 | `hestia_company_profile.redirect_url` = varchar(255) | ✅ |
| 16 | `hestia_company_profile.default_product_id` varchar(100) vs `osiris_business_account.product_id` varchar(100) | ✅ Aligned. |
| 17 | `hestia_company_api_user.identifier` = varchar(100) NOT NULL — partner's API client identifier | ✅ |
| 18 | `hestia_company_api_user.phone_number` = varchar(20) — no `@Pattern` anywhere | ⚠ |
| 19 | `hestia_company_product_access.assigned_product_id`/`product_id` = varchar(100) NOT NULL — gateway DTO passes through (no Size cap) | ⚠ |
| 20 | `iris_document_requirements.requirement_uuid` = varchar(36) NOT NULL — UUID v4 format | ✅ |
| 21 | `iris_document_requirements.document_url` = varchar(500); `mime_type` = varchar(100) | ✅ |
| 22 | `demeter_webhook_outbound_event_log.event_id` = varchar(255) — pathvar `eventId` from `retryWebhookByEventId` has no `@Size`/`@Pattern` | ⚠ |
| 23 | `demeter_webhook_outbound_event_log.signature` = varchar(300) | ✅ |
| 24 | `DocumentController.uploadDocument` (multipart) has **no `@Size`/`@Pattern` on `requirementUuid` request-param** and no max-file-size check at controller level (relies on Spring `spring.servlet.multipart.max-file-size` config) | ⚠ |
| 25 | `WebhookController.retryWebhookByEventId` `@PathVariable("eventId")` String — no `@Size`/`@Pattern` constraint | ⚠ |

---

## 1. `POST /internal/partner/register` — Register a new partner (internal service-to-service)

- **Controller:** `InternalPartnerIntegrationController.registerCompany` — **⚠ missing `@Valid`**
- **Request DTO:** `CompanyRegistrationRequest`
- **Service flow:** `CompanyProfileService.registerCompany` → uses `CompanyMapper.toEntity` → saves `CompanyProfileEntity`
- **Persisted to:** `hestia_company_profile` + (downstream) `hestia_company_api_user`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `name` | String | (no validation) | `hestia_company_profile.name` varchar(100) NOT NULL · `hestia_company_api_user.name` varchar(255) NOT NULL | ❌ Two widths. |
| 2 | `apiClientId` | String | (no validation) | mapped to `hestia_company_api_user.identifier` varchar(100) NOT NULL | ⚠ | |
| 3 | `apiSecret` | String | (no validation) | (server-hashed and stored elsewhere) | ❌ | Sensitive field, no `@Size`. |
| 4 | `description` | String | (no validation) | `description` text | ⚠ | |
| 5 | `website` | String | (no validation) | `hestia_company_profile.website` varchar(100) · `hestia_company_api_user.website` varchar(255) | ❌ Two widths. |
| 6 | `supportEmail` | String | (no validation, no @Email) | `support_email` varchar(100) · varchar(255) | ❌ |
| 7 | `redirectUrl` | String | (no validation) | `redirect_url` varchar(255) | ⚠ | |
| 8 | `address` | String | (no validation) | `address` text | ⚠ | |
| 9 | `ipAddress` | String | (no validation) | `ip_address` **varchar(255)** | ❌ | IPv6 maxes at 45. |
| 10 | `defaultProductId` | String | (no validation) | `default_product_id` varchar(100) | ⚠ | |
| 11 | `settlementApiAccess` | Boolean | (no validation) | `settlement_api_access` bool | ✅ | |
| 12 | `transactionLimit` | BigDecimal | (no validation) | `transaction_limit` numeric | ⚠ | No `@PositiveOrZero`. |
| 13 | `achHoldDays` | Integer | (no validation) | `ach_hold_days` integer NOT NULL | ❌ | No `@Min(0)`. Partner could send negative. |

---

## 2. `POST /internal/partner/portal-user/create` — Create partner portal user

- **Controller:** `InternalPartnerIntegrationController.createPartnerPortalUser` — **⚠ missing `@Valid`**
- **Request DTO:** `PartnerPortalUserRequest`
- **Service flow:** `CompanyProfileService.createPartnerPortalUser` → calls glide-portal `SupportUserService` to create the portal user
- **Persisted to:** glide-portal `iris_support_user` (out of file scope)

| # | Field | Type | Gateway | Aligned | Notes |
|---|---|---|---|---|---|
| 1 | `companyId` | Long | @NotNull | ✅ | Wouldn't fire due to missing `@Valid`. |
| 2 | `email` | String | @NotNull (no @Email, no @Size) | ❌ | |
| 3 | `phone` | String | (no validation, no @Pattern) | ⚠ | |
| 4 | `clientId` | String | @NotNull | ⚠ | Confusingly typed as String here. |
| 5 | `groups` | Set<Long> | @NotNull (no @NotEmpty) | ⚠ | Empty set passes `@NotNull`. |
| 6 | `userType` | String | (no validation) | ⚠ | |
| 7 | `firstName` | String | (no validation) | ⚠ | |
| 8 | `lastName` | String | (no validation) | ⚠ | |
| 9 | `clientIpAddress` | String | (no validation) | ⚠ | |

---

## 3. `PUT /internal/partner/{companyId}` — Update partner profile

- **Controller:** `InternalPartnerIntegrationController.updateCompanyProfile` — **⚠ missing `@Valid`**
- **Request DTO:** `UpdateCompanyRequest`
- **Service flow:** `CompanyProfileService.updateCompany` → `CompanyMapper.updateEntityFromRequest` (cumulative update) → save `CompanyProfileEntity`
- **Persisted to:** `hestia_company_profile`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `name` | String | (no validation) | `name` varchar(100) NOT NULL | ❌ | |
| 2 | `description` | String | (no validation) | `description` text | ⚠ | |
| 3 | `website` | String | (no validation) | `website` varchar(100) | ⚠ | |
| 4 | `supportEmail` | String | (no validation, no @Email) | `support_email` varchar(100) | ❌ | |
| 5 | `redirectUrl` | String | (no validation) | `redirect_url` varchar(255) | ⚠ | |
| 6 | `address` | String | (no validation) | `address` text | ⚠ | |
| 7 | `ipAddress` | String | (no validation) | `ip_address` varchar(255) | ❌ | |
| 8 | `defaultProductId` | String | (no validation) | `default_product_id` varchar(100) | ⚠ | |
| 9 | `settlementApiAccess` | Boolean | (no validation) | `settlement_api_access` bool | ✅ | |
| 10 | `transactionLimit` | BigDecimal | (no validation) | `transaction_limit` numeric | ⚠ | |
| 11 | `achHoldDays` | Integer | (no validation) | `ach_hold_days` integer NOT NULL | ❌ | |

---

## 4. `POST /internal/partner/{companyId}/product/enable` — Enable product for company

- **Controller:** `InternalPartnerIntegrationController.enableProduct` — **⚠ missing `@Valid`**
- **Request DTO:** `EnableProductRequest`
- **Service flow:** `CompanyProfileService.enableProductForCompany` → `DepositAccountManager.enableProduct(...)` (deposit-api artifact 🔒) + writes `CompanyProductAccessEntity`
- **Persisted to:** `hestia_company_product_access`

| # | Field | Type | Gateway | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `productType` | enum `ProductType` | @NotNull | (server-resolved into `assigned_product_id` + `group_name`) | ✅ | |
| 2 | `categoryDefaultProduct` | Boolean | (no validation; default true) | `category_default_product` bool NOT NULL | ✅ | |
| 3 | `companyDefaultProduct` | Boolean | (no validation; default false) | (used server-side to set `hestia_company_profile.default_product_id`) | ✅ | |
| 4 | `sourceCompanyId` | Long | (no validation) | (server-resolved) | ⚠ | |
| 5 | `currency` | String | (no @Size, no @Pattern) | `hestia_company_product_access.currency` varchar(8) NOT NULL (default "USD") | ❌ | |
| 6 | `provider` | String | (no @Size, no @Pattern) | `provider` varchar(50) NOT NULL | ❌ | |

---

## 5. `POST /admin/documents/list` — Fetch document requirements

- **Controller:** `DocumentController.getDocumentRequirements` — `@Valid` ✅ (but DTO has no validation)
- **Request DTO:** `DocumentRequirementListRequest`
- **Service flow:** `DocumentService.getDocumentRequirements` → `PortalManager.fetchDocumentRequirements(...)` (**glide-portal**)
- **Persisted to:** Read-only on `iris_document_requirements`

| # | Field | Type | Gateway | DB column (`iris_document_requirements`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `clientId` | Long | (no validation) | `client_id` | ⚠ | |
| 2 | `businessId` | Long | (no validation) | `business_id` | ⚠ | |
| 3 | `identifier` | String | (no validation) | `identifier` varchar(100) | ⚠ | |
| 4 | `accountType` | String | (no validation) | `account_type` varchar(20) NOT NULL | ⚠ | |
| 5 | `statuses` | List<String> | (no validation) | filter against `status` varchar(30) | ⚠ | |
| 6 | `clientIpAddress` | String | (no validation) | — | ⚠ | |

---

## 6. `POST /admin/documents/upload` — Upload document (multipart)

- **Controller:** `DocumentController.uploadDocument`
- **Request:** multipart form params (no `@Valid` DTO — direct request params)
- **Service flow:** `DocumentService.uploadDocument` → `PortalManager.uploadDocument(...)` (portal-api artifact 🔒) → S3 upload + update `iris_document_requirements`
- **Persisted to:** `iris_document_requirements` UPDATE (sets `document_url`, `file_path`, `mime_type`, `file_size`, `submitted_date`, `status='SUBMITTED'`)

| # | Field | Type | Gateway (request-param) | DB column | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `requirement_Id` | String | @RequestParam (no @Size, no @Pattern) | `iris_document_requirements.requirement_uuid` varchar(36) NOT NULL | ❌ | UUID format not validated. |
| 2 | `file` | MultipartFile | @RequestParam (no `@NotNull`) | `iris_document_requirements.document_url`/`file_path` varchar(500), `file_size` bigint, `mime_type` varchar(100) | ⚠ | No per-controller max-file-size or `@Pattern` on allowed mime types. Relies on Spring config. |
| 3 | `client_id` | Long | @RequestParam (no @Min) | `client_id` | ⚠ | |
| 4 | `business_id` | Long | @RequestParam (no @Min) | `business_id` | ⚠ | |

---

## 7. `POST /admin/webhooks/events/search` — Search webhook events

- **Controller:** `WebhookController.searchWebhookEvents` — `@Valid` ✅
- **Request DTO:** `WebhookEventSearchRequest`
- **Service flow:** `WebhookService.searchWebhookEvents` → `WebhookOutBoundAPIManager.searchEvents(...)` (**glide-webhook**)
- **Persisted to:** Read-only on `demeter_webhook_outbound_event_log`

| # | Field | Type | Gateway | DB column (`demeter_webhook_outbound_event_log`) | Aligned | Notes |
|---|---|---|---|---|---|---|
| 1 | `eventType` | String | (no validation, no @Pattern) | `event_type` varchar(100) NOT NULL | ⚠ | |
| 2 | `status` | enum `EventStatus` | (no @NotNull — optional filter) | `status` varchar(50) NOT NULL | ✅ | |
| 3 | `startDate` | LocalDate | @JsonFormat("yyyy-MM-dd") (no @Past) | filter on `created_on` | ⚠ | |
| 4 | `endDate` | LocalDate | @JsonFormat("yyyy-MM-dd") (no relative-order check) | filter on `created_on` | ⚠ | |
| 5 | `page` | Integer | @Min(0) (default 0) | (pagination) | ✅ | |
| 6 | `size` | Integer | @Min(1) (default 20) — **no @Max** | (pagination) | ❌ | Partner can request 1M results. |
| 7 | `clientIpAddress` | String | (no validation) | — | ⚠ | |

(Plus path-variable `eventId` on `retryWebhookByEventId` — no `@Size`/`@Pattern`. DB column `event_id` is varchar(255).)

---

## Verification

| Endpoint | Field traced | Result |
|---|---|---|
| `POST /internal/partner/register` | DTO zero validation + controller missing `@Valid` | ✅ |
| `POST /internal/partner/portal-user/create` | `@NotNull` on 4 fields but controller missing `@Valid` | ✅ |
| `PUT /internal/partner/{id}` | DTO zero validation + controller missing `@Valid` | ✅ |
| `POST /internal/partner/{id}/product/enable` | `@NotNull` on `productType`; controller missing `@Valid` | ✅ |
| `POST /documents/list` | DTO zero validation; controller has `@Valid` (no-op) | ✅ |
| `POST /documents/upload` | Multipart params unbounded; no MIME-type check | ✅ |
| `POST /webhooks/events/search` | `size` no @Max → DoS surface | ✅ |
| `hestia_company_profile.name` varchar(100) vs `hestia_company_api_user.name` varchar(255) | ✅ Cross-table mismatch confirmed. |
| `hestia_company_profile.ip_address` varchar(255) | ✅ Way over IPv6 max. |

---

## Recommended next-step actions

1. **Critical:** Add `@Valid` to all 4 `InternalPartnerIntegrationController` endpoints' `@RequestBody` parameters. Otherwise no DTO validation can fire.
2. Add field-level validation to all 12 fields of `CompanyRegistrationRequest`. At minimum:
   - `name` `@NotBlank @Size(max=100)`
   - `apiClientId` `@NotBlank @Size(max=100)`
   - `apiSecret` `@NotBlank @Size(min=32,max=255)`
   - `supportEmail` `@Email @Size(max=100)`
   - `website` `@URL @Size(max=255)`
   - `redirectUrl` `@URL @Size(max=255)`
   - `ipAddress` `@Size(max=45) @Pattern` (IPv4/IPv6)
   - `transactionLimit` `@PositiveOrZero`
   - `achHoldDays` `@Min(0) @Max(30)`
3. Mirror validation on `UpdateCompanyRequest` (most fields are optional but should still have `@Size`/`@Pattern`).
4. Add `@Email`, `@Size(max=100)`, `@Pattern("\\d{1,15}")` to `PartnerPortalUserRequest.email`/`phone`. Add `@NotEmpty` to `groups`.
5. Add `@Size(min=3,max=3) @Pattern("[A-Z]{3}")` to `EnableProductRequest.currency`. `@Size(max=50)` to `provider`.
6. Add `@Max(200)` to `WebhookEventSearchRequest.size`.
7. Add custom `@AssertTrue` validator on `WebhookEventSearchRequest` ensuring `startDate <= endDate`.
8. Reconcile column widths: `hestia_company_profile.name` (100) vs `hestia_company_api_user.name` (255). Pick canonical 100.
9. Reconcile `website` widths: `hestia_company_profile.website` (100) vs `hestia_company_api_user.website` (255). Pick 255.
10. Reconcile `support_email` widths: same — pick canonical 100.
11. Narrow `hestia_company_profile.ip_address` from varchar(255) to varchar(45) (IPv6 max).
12. Add `@Pattern` UUID validator and `@Size(min=36,max=36)` to `requirement_Id` request-param on `uploadDocument`.
13. Validate MIME type and file size in `DocumentController.uploadDocument` before reaching the service layer (allow PDF / JPEG / PNG / DOC).
14. Add `@Pattern` UUID to `eventId` path-variable on `retryWebhookByEventId` (or whatever ID format webhooks use).
15. Rename `PartnerPortalUserRequest.clientId` (currently `String`) to match the Long type used elsewhere, or document why it's a String here.
