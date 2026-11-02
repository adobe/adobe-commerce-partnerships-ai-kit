# MODULE_INDEX.md — adobe-commerce-partnerships-ref-app

## Overview

adobe-commerce-partnerships-ref-app is a stateless Next.js backend-for-frontend for Adobe's partner/reseller commerce
experience. It owns no data of its own: every write and most reads pass through to the Adobe
Commerce Partner API, authenticated via a cached, service-level Adobe IMS OAuth token. adobe-commerce-partnerships-ref-app's
job is to validate/shape requests and responses (Zod), multiplex several logical operations onto
a small set of physical Next.js API routes, and normalize error handling and structured logging
across all of them. There are no async events, no database, and no inbound end-user
authentication at the API layer — see `CONTRACTS.md` and `CONNECTORS.md` for detail.

## Source Structure

| Layer | Path | File count |
|---|---|---|
| API Routes | `pages/api/` | 9 |
| Controllers | `controllers/` | 7 |
| Models (Zod schemas + types) | `models/` | 11 |
| Utilities | `utils/` | 19 (backend-relevant: `apiError.ts`, `commonUtils.ts`, `constants.ts`, `imsTokenService.ts`, `logger.ts`; remainder are frontend-only helpers, out of scope for this card set) |
| Types (ambient/shared) | `types/` | 7 (frontend-oriented; not consumed by controllers) |
| Tests (backend-relevant) | `tests/` root (integration) | 13 files; `tests/unit/` (unit) 1 file |

Frontend layers (`pages/*.tsx`, `components/`, `hooks/`, `contexts/`, `styles/`) exist in this
repo but are out of scope for backend service cards.

## Capability List

| Capability | Primary Handler | Stability |
|---|---|---|
| Get Customer | `customerController::getCustomer` | STABLE |
| List Customers for Reseller | `customerController::getAllCustomers` | STABLE |
| Create Customer | `customerController::createCustomer` | STABLE |
| Update Customer | `customerController::updateCustomer` | STABLE |
| Search Customers | `pages/api/search.ts::searchCustomers` | STABLE |
| Search Resellers | `pages/api/search.ts::searchResellers` | STABLE |
| Get Order | `orderController::getOrders` | STABLE |
| Get Orders History | `orderController::getOrdersHistoryForCustomer` | STABLE |
| Create New Order | `orderController::createNewOrder` | STABLE |
| Create Return Order | `orderController::createReturnOrder` | STABLE |
| Create Preview Order | `orderController::createPreviewOrder` | STABLE |
| Create Preview Renewal Order | `orderController::createPreviewRenewalOrder` | STABLE |
| Create Renewal Order | `orderController::createRenewalOrder` | STABLE |
| Update Order (external reference) | `orderController::updateOrder` | STABLE (not wired to any `pages/api` route — see note below) |
| Get Price List | `priceController::getPriceList` | STABLE |
| Get Recommendations | `recommendationController::getRecommendations` | IN FLUX (validation failure falls back to unvalidated raw data instead of erroring — see Key Decisions) |
| Get Partner Details / Market Segments / Currencies | `partnerDetailsController::getPartnerDetails` (+2 derived) | STABLE |
| Get Reseller Details | `resellerController::getResellerDetails` | STABLE |
| List All Resellers | `resellerController::getAllResellers` | STABLE |
| Create Reseller | `resellerController::createReseller` | IN FLUX (no response-schema validation — returns raw upstream JSON) |
| Get Subscription Details | `subscriptionController::getSubscriptionDetails` | STABLE |
| List Customer Subscriptions | `subscriptionController::getCustomerSubscriptions` | STABLE |
| Create Subscription | `subscriptionController::createSubscription` | IN FLUX (no response-schema validation) |
| Update Subscription | `subscriptionController::updateSubscription` | IN FLUX (no response-schema validation) |
| Get Runtime Env Config | `pages/api/env.ts` (no controller) | IN FLUX (public env var allowlist is currently an empty placeholder object) |

> **Note:** `orderController::updateOrder` (PATCH order, sets `externalReferenceId`) is exported
> but not called from any `pages/api/*.ts` route or elsewhere in `controllers/`/`utils/` — dead
> code from the API surface's perspective, or a capability awaiting a route. Documented as found.

---

## Capability Catalogue

### Get Customer
- **Handler:** `pages/api/customers.ts` (`GET ?type=getCustomer`) → `customerController::getCustomer`
- **Services:** none (single controller function)
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/customers/{id}`)
- **Domain:** `CustomerDetailsSchema` (response)
- **Validation:** response validated against `CustomerDetailsSchema`; no request-body validation (path param only)
- **Events:** none
- **Key Decisions:**
  - System-of-record: Partner API owns customer data; adobe-commerce-partnerships-ref-app is a pass-through/cache-less proxy. **Enforcement:** no persistence layer exists (confirmed via dependency scan — no DB/ORM).
  - Idempotency: safe to retry (GET). **Enforcement:** no side effects in code path.
  - Fail-closed on upstream error/parse/validation failure. **Enforcement:** `customerController.ts:60-88` throws `ApiError` on non-OK, non-JSON, or schema-mismatch responses.

### List Customers for Reseller
- **Handler:** `pages/api/customers.ts` (`GET ?type=getAllCustomers`) → `customerController::getAllCustomers`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/resellers/{id}/customers?offset&limit`)
- **Domain:** `CustomerMinimalSchema` (per item), `PaginatedCustomersResponse`
- **Validation:** each item parsed individually; invalid items are logged and dropped (`filter(Boolean)`), not fatal
- **Key Decisions:**
  - Fail-open at the item level, fail-closed at the request level: a malformed customer record is skipped; a non-OK/non-JSON upstream response still throws. **Enforcement:** `customerController.ts:349-358` (per-item try/catch) vs. `:308-326` (request-level throw).
  - Idempotent (GET).

### Create Customer
- **Handler:** `pages/api/customers.ts` (`POST ?type=createCustomer`) → `customerController::createCustomer`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/customers`)
- **Domain:** `CreateCustomerSchema` (request), `CustomerDetailsSchema` (response)
- **Validation:** request body validated pre-flight (400 on failure, no upstream call made); response re-validated post-flight
- **Key Decisions:**
  - Not idempotent — no idempotency key or duplicate-detection sent upstream. **Enforcement:** not enforced in code — relies on caller discipline (e.g. UI-generated `externalReferenceId` uniqueness is never checked here).
  - Fail-closed. **Enforcement:** `customerController.ts:142-151`.

### Update Customer
- **Handler:** `pages/api/customers.ts` (`PATCH ?type=updateCustomer`) → `customerController::updateCustomer`
- **Connector(s):** Adobe Commerce Partner API (`PATCH /v3/customers/{id}`)
- **Domain:** `UpdateCustomerSchema` (request), `CustomerDetailsSchema` (response)
- **Key Decisions:**
  - Idempotent by HTTP semantics (PATCH with full target state), not verified in code. **Enforcement:** not enforced — relies on upstream API's own idempotency guarantees.
  - Fail-closed on validation or upstream error.

### Search Customers
- **Handler:** `pages/api/search.ts::searchCustomers`
- **Services:** fans out to `customerController::findCustomerByName` and, if the query is numeric (`/^\d{3,}$/`), also `customerController::getCustomer`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/resellers/{id}/customers?company-name=*query*`, `GET /v3/customers/{id}`)
- **Domain:** merged `{ customers, totalCount, count, offset, limit, hasMore }`
- **Validation:** delegated to the two underlying controller functions; merge logic itself is unvalidated
- **Key Decisions:**
  - Fail-open across the fan-out: `Promise.allSettled` means either branch failing independently does not fail the whole search. **Enforcement:** `pages/api/search.ts:94` + result-status checks at `:96-103`.
  - Dedup by `customerId` when merging name-search and numeric-lookup results; numeric result only appended if `combinedResults.length < limit`. **Enforcement:** `pages/api/search.ts:105-51`.
- **Call graph:**
  1. `handler` validates `type`/`query`/`resellerId`/pagination params
  2. `handlePrerequisites` → IMS token
  3. `searchCustomers(resellerId, query, accessToken, offset, limit)`
  4. → `findCustomerByName` (always) and, if numeric, `getCustomer` — both via `Promise.allSettled`
  5. merge + dedupe → response

### Search Resellers
- **Handler:** `pages/api/search.ts::searchResellers`
- **Services:** fans out to `resellerController::findResellerByName` and, if numeric, `resellerController::getResellerDetails`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/resellers?company-name=*query*`, `GET /v3/resellers/{id}`)
- **Key Decisions:** identical fail-open/dedupe pattern to Search Customers, mirrored in `pages/api/search.ts:12-65`.
- **Call graph:** same shape as Search Customers, substituting reseller functions.

### Get Order
- **Handler:** `pages/api/orders.ts` (`GET ?type=getOrders`) → `orderController::getOrders`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/customers/{customerId}/orders/{orderId}`)
- **Domain:** `PaginatedResponse` (order + `links`) — response not passed through `OrderSchema.parse` (unlike other order operations)
- **Key Decisions:**
  - No response-schema validation on this specific read path. **Enforcement:** not enforced — `orderController.ts:98` returns `data as PaginatedResponse` directly, a cast rather than a Zod parse.
  - Fail-closed on non-OK/non-JSON.

### Get Orders History
- **Handler:** `pages/api/orders.ts` (`GET ?type=getOrdersHistory`) → `orderController::getOrdersHistoryForCustomer`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/customers/{id}/orders?offset&limit&fetch-price=true`)
- **Domain:** `OrdersHistoryResponseSchema` + computed `hasMore`
- **Key Decisions:** fail-closed with full response validation (`orderController.ts:423-441`); idempotent (GET).

### Create New / Return / Preview / Preview-Renewal / Renewal Order
- **Handlers:** `orderController::createNewOrder` / `createReturnOrder` / `createPreviewOrder` / `createPreviewRenewalOrder` / `createRenewalOrder` — all delegate to the shared `orderController::createOrder(body, accessToken, queryParams?)`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/customers/{customerId}/orders`)
- **Domain:** one request schema per order type (`NewOrderSchema`, `ReturnOrderSchema`, `PreviewOrderSchema`, `PreviewRenewalOrderSchema`, `RenewalOrderSchema`), shared `OrderSchema` response
- **Validation:** each entry function validates against its own schema *before* calling the shared `createOrder`; `createOrder` itself does not re-validate the `orderType` discriminant, so a caller invoking `createOrder` directly could bypass the type-specific schema.
- **Key Decisions:**
  - Not idempotent — no dedupe on `externalReferenceId` before the upstream POST. **Enforcement:** not enforced in code.
  - Fail-closed; on upstream error, the raw upstream JSON body is stringified into `ApiError.message` (`orderController.ts:150`) so `pages/api/orders.ts` can re-parse and forward it verbatim.
- **Call graph (representative — Create New Order):**
  1. `pages/api/orders.ts` handler (`POST ?type=NEW`) extracts `{ customerId, externalReferenceId, currencyCode, lineItems }`
  2. `createNewOrder` validates against `NewOrderSchema`
  3. → `createOrder(body, accessToken)` builds URL, POSTs to Partner API
  4. response parsed → `OrderSchema.parse`
  5. `BackendResult` returned up through the route to the client

### Update Order (external reference)
- **Handler:** `orderController::updateOrder` — **not invoked by any `pages/api` route**
- **Connector(s):** Adobe Commerce Partner API (`PATCH /v3/customers/{customerId}/orders/{orderId}`)
- **Domain:** `{ externalReferenceId }` request, `OrderSchema` response
- **Key Decisions:** same fail-closed pattern as other order writes. Flagged as unreferenced — confirm with the team whether this is intentionally unexposed or a missing route before building on top of it.

### Get Price List
- **Handler:** `pages/api/pricelist.ts` → `priceController::getPriceList`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/pricelist`)
- **Domain:** `PriceListRequestSchema` (request), `PriceListResponseSchema` (response)
- **Validation:** Zod schema + explicit non-whitespace currency guard (see `CODE_PATTERNS.md → ## 4`)
- **Key Decisions:**
  - Fail-closed before any network call if currency is empty/whitespace — cheaper rejection path. **Enforcement:** `priceController.ts:39-42`.
  - `hasMore` is computed heuristically as `data.items.length === limit` (not from a total-count field) — an exact-limit-size page that happens to be the last page will be incorrectly reported as having more. **Enforcement:** not enforced/guarded — documented behavior, `priceController.ts:102`.

### Get Recommendations
- **Handler:** `pages/api/recommendation.ts` → `recommendationController::getRecommendations`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/recommendations`)
- **Domain:** `RecommendationRequestSchema` (request), `RecommendationResponseSchema` (response)
- **Key Decisions:**
  - **Fail-open on response validation** — unique among this service's read paths: if `RecommendationResponseSchema.parse(data)` throws, the function logs the error and returns the *raw, unvalidated* upstream `data` instead of throwing `ApiError`. **Enforcement:** `recommendationController.ts:111-121`. Callers of this capability must defensively handle a response that may not match `RecommendationResponseSchema`.
  - Idempotent (effectively a GET-like POST, no state mutation upstream).

### Get Partner Details / Market Segments / Currencies
- **Handler:** `pages/api/partnerDetails.ts` → `partnerDetailsController::getPartnerDetails` (base) / `getPartnerDetailsMarketSegments` / `getPartnerDetailsCurrencies` (both derive from the base call)
- **Connector(s):** none — sourced entirely from environment variables (`PARTNER_NAME`, `MARKET_SEGMENTS`, `CURRENCIES`, `REGION`)
- **Domain:** `PartnerDetailsSchema`
- **Key Decisions:**
  - System-of-record is **environment configuration**, not the Partner API — this is a deliberate exception to the "Partner API is system of record" pattern used everywhere else in this service. **Enforcement:** `partnerDetailsController.ts:19-64` — no `fetch` call present.
  - Fail-closed at startup-config level: missing/malformed env vars throw `ApiError(500)` on every request (not cached), rather than failing fast at process boot. **Enforcement:** `parseEnvArray` (`partnerDetailsController.ts:6-17`).

### Get Reseller Details
- **Handler:** `pages/api/resellers.ts` (`GET ?type=getResellerDetails`) → `resellerController::getResellerDetails`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/resellers/{id}`)
- **Domain:** `ResellerDetailsSchema`
- **Key Decisions:** fail-closed with full response validation (`resellerController.ts:177-190`); idempotent.

### List All Resellers
- **Handler:** `pages/api/resellers.ts` (`GET ?type=getAllResellers`) → `resellerController::getAllResellers`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/resellers?offset&limit`)
- **Domain:** `ResellerMinimalSchema` (per item)
- **Key Decisions:** unlike `getAllCustomers`, per-item parsing here is **not** wrapped in try/catch (`resellerController.ts:92`) — a single malformed reseller record throws and fails the entire list request. **Enforcement:** none — this is a divergence from the customer-list pattern, worth reconciling if it causes production incidents.

### Create Reseller
- **Handler:** `pages/api/resellers.ts` (`POST`, `type=createReseller` or omitted) → `resellerController::createReseller`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/resellers`)
- **Domain:** `ResellerSchema` (request only)
- **Key Decisions:**
  - No response-schema validation — `data` is returned raw (`resellerController.ts:254-257`). **Enforcement:** none; contract with the frontend relies on the upstream API not changing shape.
  - Not idempotent.

### Get Subscription Details
- **Handler:** `pages/api/subscriptions.ts` (`GET ?type=getSubscriptionDetails`) → `subscriptionController::getSubscriptionDetails`
- **Connector(s):** Adobe Commerce Partner API (`GET {baseUrl}/customers/{customerId}/subscriptions/{subscriptionId}` — **note:** missing `/v3` prefix, see `CONNECTORS.md#1`)
- **Domain:** `SubscriptionSchema`
- **Key Decisions:** fail-closed with response validation.

### List Customer Subscriptions
- **Handler:** `pages/api/subscriptions.ts` (`GET ?type=getCustomerSubscriptions`) → `subscriptionController::getCustomerSubscriptions`
- **Connector(s):** Adobe Commerce Partner API (`GET /v3/customers/{id}/subscriptions`)
- **Domain:** `SubscriptionSchema[]`
- **Key Decisions:** throws if `data.items` is missing/not an array (`subscriptionController.ts:309-311`) rather than defaulting to an empty list — stricter than the analogous customer/reseller list endpoints.

### Create Subscription
- **Handler:** `pages/api/subscriptions.ts` (`POST`) → `subscriptionController::createSubscription`
- **Connector(s):** Adobe Commerce Partner API (`POST /v3/customers/{customerId}/subscriptions`)
- **Domain:** `CreateSubscriptionRequestSchema` (request); response returned raw (no schema)
- **Key Decisions:** not idempotent; no response validation — same posture as Create Reseller.

### Update Subscription
- **Handler:** `pages/api/subscriptions.ts` (`PATCH`) → `subscriptionController::updateSubscription`
- **Connector(s):** Adobe Commerce Partner API (`PATCH /v3/customers/{customerId}/subscriptions/{subscriptionId}[?reset-flex-discount-codes=true]`)
- **Domain:** `UpdateSubscriptionRequestSchema` (request, wrapped as `{ autoRenewal: body }`); response returned raw
- **Key Decisions:** the `reset-flex-discount-codes` query flag changes the upstream URL but is not itself validated/typed — a plain string comparison (`req.query['reset-flex-discount-codes'] === 'true'`) in `pages/api/subscriptions.ts:73`.

### Get Runtime Env Config
- **Handler:** `pages/api/env.ts` (no controller layer)
- **Connector(s):** none
- **Domain:** none (untyped `Record<string, string | undefined>`)
- **Key Decisions:** the allowlisted public env vars object is currently empty (`// Add any safe public environment variables here`) — the endpoint is scaffolded but not populated. **Enforcement:** none; flagged as IN FLUX.

---

## Cross-Reference: Connector → Capabilities

| Connector | Capabilities that use it |
|---|---|
| Adobe Commerce Partner API | Get/List/Create/Update Customer, Search Customers, Search Resellers, Get/List Order(s) History, Create New/Return/Preview/Preview-Renewal/Renewal Order, Update Order, Get Price List, Get Recommendations, Get/List/Create Reseller, Get/List/Create/Update Subscription |
| Adobe IMS OAuth Token Service | every capability above (via `handlePrerequisites`, invoked once per request in each `pages/api/*.ts` route except `partnerDetails.ts` and `env.ts`) |
| *(none)* | Get Partner Details / Market Segments / Currencies (env-var only), Get Runtime Env Config (static) |

## Type Inventory

| Type | Kind | Used by capability |
|---|---|---|
| `CustomerMinimal` | Domain | List Customers for Reseller, Search Customers |
| `Customer` | Domain | declared (`models/Customer.ts`), not directly referenced by any controller |
| `CustomerDetails` | Response | Get/Create/Update Customer |
| `CreateCustomer` | Request | Create Customer |
| `UpdateCustomer` | Request | Update Customer |
| `Order` | Response | Create New/Return/Preview/Preview-Renewal/Renewal Order, Update Order |
| `LineItem` | Domain | embedded in all Order request/response schemas |
| `Pricing` | Domain | embedded in `LineItem` |
| `OrdersHistoryResponse` / `OrdersHistoryOrder` | Response | Get Orders History |
| `OrdersHistoryResponseWithPagination` | Response | Get Orders History (adds `hasMore`) |
| `Reseller` | Request | Create Reseller (validates input; response is unvalidated raw JSON) |
| `ResellerMinimal` | Domain | List All Resellers, Search Resellers |
| `ResellerDetails` | Response | Get Reseller Details |
| `Subscription` | Response | Get Subscription Details, List Customer Subscriptions |
| `PriceListRequest` | Request | Get Price List |
| `PriceListResponse` / `PriceListOffer` | Response | Get Price List |
| `PartnerDetails` / `Currency` | Response | Get Partner Details / Currencies |
| `RecommendationRequest` | Request | Get Recommendations |
| `RecommendationResponse` / `RecommendationItem` / `RecommendationProduct` | Response | Get Recommendations; `RecommendationItem` also embedded (optional) in `OrderSchema.recommendations` |
| `BackendResult<T>` | Wrapper | return type for most controller functions — `{ data: T; requestId?: string }` |
| `CatalogProduct` / `CatalogFilters` | Domain | declared in `models/Catalog.ts`; no controller/API-route reference found — appears to serve frontend catalog features (`lib/product_merchandising.json`, `utils/prefetchProducts.ts`), out of scope for this backend card set |