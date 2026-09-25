# CONTRACTS.md — adobe-commerce-partnerships-ref-app

## 1. Overview

adobe-commerce-partnerships-ref-app exposes its backend surface as Next.js API Routes under `pages/api/`. Most resources are
**multiplexed**: one physical route file handles several logical operations, selected by a `type`
query-string parameter (see `utils/constants.ts` for the `*_API_TYPE` enums). There are no async
events (no Kafka/SQS/Rabbit/etc. consumers or publishers) — omit that section entirely.

No inbound authentication/authorization check exists at the API-route layer itself (no session,
JWT, or API-key check on the incoming request). Every route calls `handlePrerequisites(req)`
(`utils/commonUtils.ts:370`), which unconditionally fetches a service-to-service IMS access token
via `imsTokenService.getAccessToken()` — this token authenticates adobe-commerce-partnerships-ref-app *to* the upstream Adobe
Commerce Partner API, not the caller *to* adobe-commerce-partnerships-ref-app.

---

## 2. HTTP Operations Exposed

### GET /api/customers — Fetch a single customer — `type=getCustomer`
- **Handler:** `pages/api/customers.ts` → `controllers/customerController.ts::getCustomer`
- **Query params:** `type=getCustomer` (required), `customerId` (required, single value)
- **Auth:** none inbound; outbound Bearer token + `x-api-key` to Partner API
- **Response:** `CustomerDetailsSchema` (`models/CustomerDetails.ts`) — 200
- **Errors:** 400 missing `customerId`; upstream status passed through as `{ error }`; 500 on validation/parse failure

### GET /api/customers — List customers for a reseller — `type=getAllCustomers`
- **Handler:** `customerController::getAllCustomers`
- **Query params:** `type=getAllCustomers`, `resellerId` (required), `offset` (default 0), `limit` (default 50)
- **Response:** `{ customers: CustomerMinimal[], totalCount, count, offset, limit, hasMore }` — 200
- **Errors:** 400 missing/array `resellerId`

### POST /api/customers — Create a customer — `type=createCustomer`
- **Handler:** `customerController::createCustomer`
- **Body:** `CreateCustomerSchema` (`models/CustomerDetails.ts`)
- **Response:** `CustomerDetailsSchema` — 201
- **Errors:** 400 body fails `CreateCustomerSchema`; upstream error status passed through

### PATCH /api/customers — Update a customer — `type=updateCustomer`
- **Handler:** `customerController::updateCustomer`
- **Query params:** `customerId` (required); **Body:** `UpdateCustomerSchema`
- **Response:** `CustomerDetailsSchema` — 200

---

### GET /api/orders — Fetch a single order — `type=getOrders`
- **Handler:** `pages/api/orders.ts` → `controllers/orderController.ts::getOrders`
- **Query params:** `customerId`, `orderId` (both required)
- **Response:** `PaginatedResponse` (order + pagination `links`) — 200

### GET /api/orders — Fetch order history for a customer — `type=getOrdersHistory`
- **Handler:** `orderController::getOrdersHistoryForCustomer`
- **Query params:** `customerId` (required), `offset` (default 0), `limit` (default 25)
- **Response:** `OrdersHistoryResponseSchema` + `hasMore` (`models/Order.ts`) — 200

### POST /api/orders — Create a new order — `type=NEW`
- **Handler:** `orderController::createNewOrder` → `createOrder` (shared upstream POST)
- **Body:** `{ customerId, externalReferenceId, currencyCode, lineItems }`, validated by `NewOrderSchema`
- **Response:** `OrderSchema` — 201

### POST /api/orders — Return/cancel an order — `type=Return`
- **Handler:** `orderController::createReturnOrder`
- **Body:** `{ customerId, referenceOrderId, externalReferenceId, currencyCode, lineItems }`, validated by `ReturnOrderSchema`
- **Response:** `OrderSchema` — 201

### POST /api/orders — Preview an order — `type=Preview`
- **Handler:** `orderController::createPreviewOrder`
- **Body:** `{ customerId, externalReferenceId, currencyCode, lineItems }`, validated by `PreviewOrderSchema`; forwards remaining query params (minus `type`) to upstream
- **Response:** `OrderSchema` — 201

### POST /api/orders — Preview a renewal order — `type=PreviewRenewal`
- **Handler:** `orderController::createPreviewRenewalOrder`
- **Body:** `{ customerId, currencyCode, lineItems? }`, validated by `PreviewRenewalOrderSchema`
- **Response:** `OrderSchema` — 201

### POST /api/orders — Create a renewal order — `type=RenewalOrder`
- **Handler:** `orderController::createRenewalOrder`
- **Body:** `{ customerId, externalReferenceId, currencyCode, lineItems }`, validated by `RenewalOrderSchema`
- **Response:** `OrderSchema` — 201

**Errors (all `/api/orders` operations):** 400 invalid `type` or missing required fields; upstream
non-JSON body triggers `ApiError(500)`; on upstream error, `pages/api/orders.ts` attempts to
`JSON.parse(error.message)` and forwards the parsed backend error body verbatim, falling back to
`{ error: message }` if parsing fails.

---

### GET /api/search — Search resellers — `type=reseller`
- **Handler:** `pages/api/search.ts::searchResellers` → `resellerController::findResellerByName` (+ `getResellerDetails` if `query` is a 3+ digit numeric string)
- **Query params:** `type=reseller`, `query` (required, wrapped in `*query*` for name search), `offset` (default 0), `limit` (default 50)
- **Response:** `{ resellers, totalCount, count, offset, limit, hasMore }` — 200
- **Behavior:** issues name-search and (if numeric) exact-ID lookup via `Promise.allSettled`, dedupes by ID, merges results

### GET /api/search — Search customers — `type=customer`
- **Handler:** `searchCustomers` → `customerController::findCustomerByName` (+ `getCustomer` if numeric)
- **Query params:** `type=customer`, `query` (required), `resellerId` (required), `offset`, `limit`
- **Response:** `{ customers, totalCount, count, offset, limit, hasMore }` — 200
- **Errors:** 400 missing `type`/`query`/`resellerId`, invalid `offset`/`limit`, or unrecognized `type`

---

### POST /api/pricelist — Fetch a price list
- **Handler:** `pages/api/pricelist.ts` → `controllers/priceController.ts::getPriceList`
- **Body:** `PriceListRequestSchema` (`models/PriceList.ts`) — `region`, `marketSegment`, `priceListType`, `priceListMonth`, `currency` (required, non-empty), `offset`/`limit` (default 0/100), optional `filters`, `useMock`, `includeOfferAttributes`
- **Response:** `PriceListResponseSchema` + computed `hasMore` — 200
- **Errors:** 405 non-POST; 400 schema failure or empty/whitespace `currency` (explicit guard beyond Zod `.min(1)`, rejects before the upstream call)

---

### GET /api/recommendation — Fetch product recommendations
- **Handler:** `pages/api/recommendation.ts` → `controllers/recommendationController.ts::getRecommendations`
- **Query params:** `customerId` (required, coerced to number), `recommendationContext`, `country`, `language` (all optional)
- **Response:** `RecommendationResponseSchema` (`models/recommendation.ts`) — 200; if response validation fails, the raw upstream `data` is returned instead of throwing
- **Errors:** 400 `RecommendationRequestSchema` failure; 405 non-GET

---

### GET /api/partnerDetails — Fetch partner contract details
- **Handler:** `pages/api/partnerDetails.ts` → `controllers/partnerDetailsController.ts::getPartnerDetails`
- **Source of truth:** environment variables only (`PARTNER_NAME`, `MARKET_SEGMENTS`, `CURRENCIES`, `REGION`) — **no upstream call**
- **Response:** `PartnerDetailsSchema` (`models/PartnerDetails.ts`) wrapped in `BackendResult` — 200
- **Errors:** 500 if any required env var is missing or `MARKET_SEGMENTS`/`CURRENCIES` isn't valid JSON array

### GET /api/partnerDetails?type=marketSegments — Fetch distinct market segments
- **Handler:** `partnerDetailsController::getPartnerDetailsMarketSegments` (derives from `getPartnerDetails`)

### GET /api/partnerDetails?type=currencies — Fetch supported currencies
- **Handler:** `partnerDetailsController::getPartnerDetailsCurrencies` (derives from `getPartnerDetails`)

---

### GET /api/resellers — Fetch reseller details — `type=getResellerDetails`
- **Handler:** `pages/api/resellers.ts` → `controllers/resellerController.ts::getResellerDetails`
- **Query params:** `id` (required)
- **Response:** `ResellerDetailsSchema` (`models/ResellerDetails.ts`) — 200

### GET /api/resellers — List all resellers — `type=getAllResellers`
- **Handler:** `resellerController::getAllResellers`
- **Query params:** `offset` (default 0), `limit` (default 50)
- **Response:** `{ resellers: ResellerMinimal[], totalCount, count, offset, limit, hasMore }` — 200

### POST /api/resellers — Create a reseller — `type=createReseller` (or omitted, for backward compatibility)
- **Handler:** `resellerController::createReseller`
- **Body:** `ResellerSchema` (`models/Reseller.ts`)
- **Response:** raw upstream JSON — 201

**Cross-cutting:** `pages/api/resellers.ts` also handles `OPTIONS` (CORS preflight, returns 200 empty body) and catches `ZodError` explicitly, returning `{ error: error.errors }` at 400.

---

### GET /api/subscriptions — Fetch subscription details — `type=getSubscriptionDetails`
- **Handler:** `pages/api/subscriptions.ts` → `controllers/subscriptionController.ts::getSubscriptionDetails`
- **Query params:** `customerId`, `subscriptionId` (both required)
- **Response:** `SubscriptionSchema` (`models/Subscription.ts`) — 200

### GET /api/subscriptions — List a customer's subscriptions — `type=getCustomerSubscriptions`
- **Handler:** `subscriptionController::getCustomerSubscriptions`
- **Query params:** `customerId` (required)
- **Response:** `SubscriptionSchema[]` — 200

### POST /api/subscriptions — Create a subscription — `type=createSubscription` (or omitted)
- **Handler:** `subscriptionController::createSubscription`
- **Body:** `{ customerId (required), ...CreateSubscriptionRequestSchema }`
- **Response:** raw upstream JSON — 201

### PATCH /api/subscriptions — Update auto-renewal on a subscription
- **Handler:** `subscriptionController::updateSubscription`
- **Body:** `{ customerId, subscriptionId, autoRenewal }` (all required), validated by `UpdateSubscriptionRequestSchema`
- **Query params:** `reset-flex-discount-codes=true` (optional) appends `?reset-flex-discount-codes=true` upstream
- **Response:** raw upstream JSON — 200

---

### GET /api/env — Runtime public environment variables
- **Handler:** `pages/api/env.ts` (no controller layer — self-contained)
- **Response:** object of whitelisted public env vars (currently empty placeholder), in-memory cached 60s (`Cache-Control: public, max-age=60`, `X-Cache: HIT|MISS`)
- **Errors:** 405 non-GET
- **Note:** no upstream call, no auth — pure static/cached config endpoint

---

## 3. Events Consumed

None — this service has no async message consumers.

## 4. Events Published

None — this service has no async message publishers.