# Module Index — Bridge

## Overview

Bridge is a stateless Next.js BFF (Backend-For-Frontend) for Adobe VIP Marketplace partners. It owns no data store. Every API route authenticates via IMS, then proxies validated requests to the VIP Marketplace Partner API. Business logic lives in controllers; Zod schemas in `models/` enforce request/response shapes.

---

## Source Structure

| Layer | Path | File count |
|-------|------|-----------|
| API route handlers | `pages/api/` | 10 |
| Controllers | `controllers/` | 8 |
| Zod models / schemas | `models/` | 12 |
| TypeScript types | `types/` | 6 |
| Utilities | `utils/` | 17 |
| Client-side fetch hooks | `hooks/` | 8 |
| React components | `components/` | ~30 |
| Tests | `tests/` | 15 |

---

## Capability List

| Capability | Primary Handler | Controller | Stability |
|------------|----------------|-----------|-----------|
| Customer Management | `pages/api/customers.ts` | `customerController.ts` | STABLE |
| Reseller Management | `pages/api/resellers.ts` | `resellerController.ts` | STABLE |
| Search | `pages/api/search.ts` | `resellerController.ts` + `customerController.ts` | STABLE |
| Order Operations | `pages/api/orders.ts` | `orderController.ts` + `switchOrderController.ts` | STABLE |
| Subscription Management | `pages/api/subscriptions.ts` | `subscriptionController.ts` | STABLE |
| Price List | `pages/api/pricelist.ts` | `priceController.ts` | STABLE |
| Offer Switch Paths | `pages/api/offer-switch-paths.ts` | `switchOrderController.ts` | STABLE |
| Partner Details | `pages/api/partnerDetails.ts` | `partnerDetailsController.ts` | STABLE |
| Recommendations | `pages/api/recommendation.ts` | `recommendationController.ts` | STABLE |

---

## Capability Catalogue

### Customer Management

**Handler:** `pages/api/customers.ts`  
**Scope:** GET, POST, PATCH on `/api/customers`. All customer CRUD and search operations. Does not handle subscription or order operations.

**Controller functions:**

| Function | Operation |
|----------|-----------|
| `getCustomer(customerId, token)` | Fetch single customer by ID |
| `getAllCustomers(resellerId, offset, limit, token)` | Paginated customer list under a reseller |
| `findCustomerByName(resellerId, name, token, offset, limit)` | Wildcard name search under a reseller |
| `createCustomer(body, token)` | Create new customer |
| `updateCustomer(customerId, body, token)` | PATCH customer (benefits, company profile) |

**Domain:** `models/CustomerDetails.ts` — `CustomerDetailsSchema`, `CreateCustomerSchema`, `UpdateCustomerSchema`; `models/Customer.ts` — `CustomerMinimalSchema`

**Validation:** Zod `.parse()` on request body before API call for POST and PATCH; called inside controller.

**Key Decisions:**
- `getAllCustomers` and `findCustomerByName` return a `hasMore` boolean computed as `offset + limit < totalCount`.
- Invalid customers in list responses are skipped with a warning log rather than failing the entire request.
- **Enforcement:** `CustomerMinimalSchema.parse()` per item with `.filter(Boolean)` — bad items silently dropped.

**Call graph (createCustomer):**
1. Route handler calls `handlePrerequisites(req)` → `getAccessToken()` → IMS token
2. Route handler calls `createCustomer(body, token)`
3. Controller calls `CreateCustomerSchema.parse(body)` — throws `ApiError(400)` on failure
4. Controller calls `fetch(POST /v3/customers)` with bearer token
5. Controller calls `CustomerDetailsSchema.parse(data)` on response
6. Route handler calls `forwardRequestIdHeader(res, requestId)`

---

### Reseller Management

**Handler:** `pages/api/resellers.ts`  
**Scope:** GET and POST on `/api/resellers`. Reseller listing, detail fetch, name search, and creation. Does not handle customers or orders.

**Controller functions:**

| Function | Operation |
|----------|-----------|
| `getAllResellers(offset, limit, token)` | Paginated reseller list |
| `getResellerDetails(id, token)` | Single reseller by ID |
| `findResellerByName(companyName, token, offset, limit)` | Wildcard name search |
| `createReseller(body, token)` | Create new reseller |

**Domain:** `models/Reseller.ts` — `ResellerMinimalSchema`, `ResellerSchema`; `models/ResellerDetails.ts` — `ResellerDetailsSchema`

**Validation:** `ResellerSchema.parse(body)` in `createReseller` before API call.

**Key Decisions:**
- `getAllResellers` skips invalid resellers (no `.filter()` null guard, but `ResellerMinimalSchema.parse()` is called without try/catch — a parse failure here propagates as an unhandled error).
- **Enforcement:** Not enforced — a single malformed reseller in a list response will throw and fail the whole request. Known limitation.

---

### Search

**Handler:** `pages/api/search.ts`  
**Scope:** GET on `/api/search`. Federated search over resellers and customers. Uses existing reseller and customer controller functions — no dedicated controller file.

**Logic:**
- Runs wildcard name search AND (if query is 3+ digits) a numeric ID lookup in parallel via `Promise.allSettled`
- Deduplicates results by `resellerId` / `customerId`

**Key Decisions:**
- Partial failures are tolerated: if the numeric lookup fails but name search succeeds, the name search results are returned.
- **Enforcement:** `Promise.allSettled` — failures are checked per-result rather than throwing.

---

### Order Operations

**Handler:** `pages/api/orders.ts`  
**Scope:** GET and POST on `/api/orders`. All order types: NEW, RETURN, PREVIEW, PREVIEW_RENEWAL, RENEWAL, SWITCH, PREVIEW_SWITCH. Does not handle subscription creation.

**Controller functions:**

| Function | File | Operation |
|----------|------|-----------|
| `getOrders(customerId, orderId, token)` | `orderController.ts` | Fetch single order |
| `getOrdersHistoryForCustomer(customerId, offset, limit, token)` | `orderController.ts` | Paginated order history |
| `createNewOrder(customerId, extRefId, currency, lineItems, token)` | `orderController.ts` | New purchase |
| `createReturnOrder(customerId, refOrderId, extRefId, currency, lineItems, token)` | `orderController.ts` | Return / cancel |
| `createPreviewOrder(customerId, extRefId, currency, lineItems, token, queryParams)` | `orderController.ts` | Preview pricing |
| `createPreviewRenewalOrder(customerId, currency, token, lineItems, queryParams)` | `orderController.ts` | Preview renewal pricing |
| `createRenewalOrder(customerId, extRefId, currency, lineItems, token)` | `orderController.ts` | Submit renewal |
| `createPreviewSwitchOrder(customerId, currency, lineItems, cancellingItems, token, queryParams, extRefId)` | `switchOrderController.ts` | Preview switch/upgrade |
| `createSwitchOrder(customerId, currency, lineItems, cancellingItems, token, queryParams, extRefId)` | `switchOrderController.ts` | Submit switch/upgrade |

**Domain:** `models/Order.ts` — `NewOrderSchema`, `ReturnOrderSchema`, `RenewalOrderSchema`, `PreviewOrderSchema`, `PreviewRenewalOrderSchema`, `OrderSchema`, `LineItemSchema`; `models/SwitchOrder.ts` — `SwitchOrderSchema`, `PreviewSwitchOrderSchema`

**Validation:** Zod `.parse()` inside each controller function before API call; throws `ApiError(400)` on failure.

**Flex discount codes:** `LineItemSchema` includes `flexDiscountCodes?: string[]`. Passed to the Partner API in line items; result returned in `flexDiscounts[].result: "SUCCESS" | "FAILURE"`.

**Key Decisions:**
- All order creation functions route through `createOrder()` internal helper in `orderController.ts`
- Backend error responses are attempted as JSON parse; if JSON, passed through directly to caller; if not, wrapped in `{ error: message }`
- `createPreviewRenewalOrder` body includes `lineItems` only when provided (optional)
- **Enforcement:** Per-function Zod schema parse before any network call.

**Call graph (createNewOrder):**
1. Route handler: `handlePrerequisites` → IMS token
2. Route handler: `createNewOrder(customerId, extRefId, currency, lineItems, token)`
3. Controller: `NewOrderSchema.parse(body)` — throws `ApiError(400)` on failure
4. Controller: delegates to `createOrder(body, token)`
5. `createOrder`: `fetch(POST {base}/v3/customers/{customerId}/orders)` with bearer token
6. `createOrder`: `OrderSchema.parse(data)` on response
7. Route handler: `forwardRequestIdHeader(res, requestId)` — returns HTTP 201

---

### Subscription Management

**Handler:** `pages/api/subscriptions.ts`  
**Scope:** GET, POST, PATCH on `/api/subscriptions`. Subscription creation, detail fetch, customer subscription list, and auto-renewal updates including flex discount codes.

**Controller functions:**

| Function | Operation |
|----------|-----------|
| `createSubscription(customerId, body, token)` | Create subscription with auto-renewal and optional flex discount codes |
| `getSubscriptionDetails(customerId, subscriptionId, token)` | Single subscription detail |
| `getCustomerSubscriptions(customerId, token)` | All subscriptions for a customer |
| `updateSubscription(customerId, subscriptionId, autoRenewalBody, token, resetFlexDiscount?)` | Update auto-renewal settings; pass `resetFlexDiscount=true` to append `?reset-flex-discount-codes=true` |

**Domain:** `models/Subscription.ts` — `SubscriptionSchema`, `CreateSubscriptionRequestSchema`, `UpdateSubscriptionRequestSchema`

**Validation:** `CreateSubscriptionRequestSchema.parse(body)` and `UpdateSubscriptionRequestSchema.parse(body)` in respective controller functions.

**Key Decisions:**
- `updateSubscription` wraps the body in `{ autoRenewal: body }` before sending — the route handler passes the `autoRenewal` property from the request body directly
- `resetFlexDiscount` flag appends `?reset-flex-discount-codes=true` to the PATCH URL — used when clearing discount codes from a subscription
- **Enforcement:** `UpdateSubscriptionRequestSchema` is loose — `autoRenewal`, `renewalQuantity`, and `flexDiscountCodes` are all optional; no required field validation

---

### Price List

**Handler:** `pages/api/pricelist.ts`  
**Scope:** POST on `/api/pricelist` only.

**Controller function:** `getPriceList(body, token)` in `priceController.ts`

**Domain:** `models/PriceList.ts` — `PriceListRequestSchema`, `PriceListResponseSchema`

**Validation:** `PriceListRequestSchema.parse(body)` + explicit empty-currency guard before API call.

**Key Decisions:**
- Pagination params (`offset`, `limit`) are extracted from body and only appended as URL query params when explicitly provided in the original request
- `hasMore` is computed as `items.length === limit` (not `offset + limit < totalCount`)
- Empty `currency` field is rejected before hitting the upstream API to avoid confusing error responses
- **Enforcement:** `PriceListRequestSchema.parse(body)` + `if (!parsedRequest.currency)` guard

---

### Offer Switch Paths

**Handler:** `pages/api/offer-switch-paths.ts`  
**Scope:** GET on `/api/offer-switch-paths`.

**Controller function:** `getOfferSwitchPaths(token, queryParams)` in `switchOrderController.ts`

**Domain:** `models/SwitchOrder.ts` — `OfferSwitchPathsResponseSchema`

**Query params forwarded to Partner API:** `market-segment`, `country`, `language`, `offer-id`, `subscription-id`, `customer-id`, `limit`, `offset` — all optional

---

### Partner Details

**Handler:** `pages/api/partnerDetails.ts`  
**Scope:** GET on `/api/partnerDetails`. Reads entirely from environment variables — makes NO outbound API call.

**Controller functions:** `getPartnerDetails()`, `getPartnerDetailsMarketSegments()`, `getPartnerDetailsCurrencies()` in `partnerDetailsController.ts`

**Domain:** `models/PartnerDetails.ts` — `PartnerDetailsSchema`

**Key Decisions:**
- Partner identity is baked into deployment env vars, not fetched from the Partner API
- `MARKET_SEGMENTS` and `CURRENCIES` must be valid JSON arrays (e.g. `["COM","EDU"]`) — malformed values throw `ApiError(500)` with a clear message
- **Enforcement:** `parseEnvArray()` wrapper calls `JSON.parse()` and validates `Array.isArray()` at runtime

---

### Recommendations

**Handler:** `pages/api/recommendation.ts`  
**Scope:** GET on `/api/recommendation`.

**Controller function:** `getRecommendations(params, token)` in `recommendationController.ts`

**Domain:** `models/recommendation.ts` — `RecommendationRequestSchema`, `RecommendationResponseSchema`

**Key Decisions:**
- `customerId` is coerced from string to number before Zod validation (Partner API expects numeric customer ID)
- If response fails schema validation, raw data is returned rather than throwing — graceful degradation
- **Enforcement:** `RecommendationRequestSchema.parse(validatedParams)` before API call; response validation failure returns raw data

---

## Cross-Reference: Connector → Capabilities

| Connector | Capabilities that use it |
|-----------|--------------------------|
| IMSToken (`imsTokenService.ts`) | All capabilities (called via `handlePrerequisites`) |
| PartnerAPI — `/v3/customers` | Customer Management, Search |
| PartnerAPI — `/v3/resellers` | Reseller Management, Search |
| PartnerAPI — `/v3/customers/{id}/orders` | Order Operations |
| PartnerAPI — `/v3/customers/{id}/subscriptions` | Subscription Management |
| PartnerAPI — `/v3/pricelist` | Price List |
| PartnerAPI — `/v3/offer-switch-paths` | Offer Switch Paths |
| PartnerAPI — `/v3/recommendations` | Recommendations |

---

## Type Inventory

| Type | Kind | Used by |
|------|------|---------|
| `CustomerDetails` | Response | Customer Management |
| `CustomerMinimal` | Response (list item) | Customer Management, Search |
| `CreateCustomer` | Request | Customer Management |
| `UpdateCustomer` | Request | Customer Management |
| `Order` | Response | Order Operations |
| `LineItem` | Request + Response sub-type | Order Operations |
| `Subscription` | Response | Subscription Management |
| `ResellerMinimal` | Response (list item) | Reseller Management, Search |
| `ResellerDetails` | Response | Reseller Management |
| `PartnerDetails` | Response | Partner Details |
| `BackendResult<T>` | Wrapper | All controllers — wraps `{ data, requestId? }` |
| `PaginatedCustomersResponse` | Response | Customer Management |
| `PaginatedResellersResponse` | Response (from `utils/AccountApis.ts`) | Client-side fetch helpers |
| `SwitchOrderResponse` | Response | Offer Switch Paths, Order Operations |
| `OfferSwitchPathsResponse` | Response | Offer Switch Paths |
