# Contracts — Bridge

All API routes require a valid IMS bearer token (fetched server-side via `imsTokenService.ts`). The `handlePrerequisites(req)` utility runs at the top of every handler and throws `ApiError(500)` if the token cannot be obtained.

Request IDs from the upstream Partner API are forwarded to the caller via the `X-Request-Id` response header when available.

---

## HTTP Routes Exposed

### GET /api/customers — Get Customer / Get All Customers

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `getCustomer` or `getAllCustomers` |
| `customerId` | When `type=getCustomer` | Customer ID |
| `resellerId` | When `type=getAllCustomers` | Reseller ID to list customers under |
| `offset` | No | Pagination offset (default: 0) |
| `limit` | No | Page size (default: 50) |

**Response (`type=getCustomer`):** `CustomerDetails` object (see `models/CustomerDetails.ts`)

**Response (`type=getAllCustomers`):**
```json
{
  "customers": [{ "customerId": "...", "companyProfile": { "companyName": "..." } }],
  "totalCount": 100,
  "count": 50,
  "offset": 0,
  "limit": 50,
  "hasMore": true
}
```

---

### POST /api/customers — Create Customer

**Body:** `CreateCustomer` (see `models/CustomerDetails.ts`)
```json
{
  "resellerId": "...",
  "externalReferenceId": "...",
  "companyProfile": { "companyName": "...", "preferredLanguage": "...", "address": {...}, "contacts": [...] }
}
```
**Response:** `CustomerDetails` — HTTP 201

---

### PATCH /api/customers — Update Customer

**Query params:** `type=updateCustomer`, `customerId`

**Body:** `UpdateCustomer`
```json
{
  "benefits": [{ "type": "THREE_YEAR_COMMIT", "commitmentRequest": { "minimumQuantities": [...] } }],
  "companyProfile": { ... }
}
```
**Response:** `CustomerDetails` — HTTP 200

---

### GET /api/orders — Get Order / Get Orders History

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `getOrders` or `getOrdersHistory` |
| `customerId` | Yes | Customer ID |
| `orderId` | When `type=getOrders` | Order ID |
| `offset` | When `type=getOrdersHistory` | Default: 0 |
| `limit` | When `type=getOrdersHistory` | Default: 25 |

**Response (`type=getOrders`):** Raw paginated order response from Partner API

**Response (`type=getOrdersHistory`):**
```json
{
  "totalCount": 50,
  "count": 25,
  "offset": 0,
  "limit": 25,
  "items": [ { "orderId": "...", "orderType": "NEW", "status": "1000", "lineItems": [...] } ],
  "hasMore": true
}
```

---

### POST /api/orders — Create Order

**Query param:** `type` — one of `NEW`, `Return`, `Preview`, `PreviewRenewal`, `RenewalOrder`, `PREVIEW_SWITCH`, `SWITCH`

**Body (type=NEW):**
```json
{
  "customerId": "...",
  "externalReferenceId": "...",
  "currencyCode": "USD",
  "lineItems": [{ "extLineItemNumber": 1, "offerId": "...", "quantity": 10, "flexDiscountCodes": ["CODE"] }]
}
```

**Body (type=PreviewRenewal):**
```json
{
  "customerId": "...",
  "currencyCode": "USD",
  "lineItems": [{ "extLineItemNumber": 1, "offerId": "...", "quantity": 10, "flexDiscountCodes": ["CODE"] }]
}
```

**Body (type=SWITCH / PREVIEW_SWITCH):**
```json
{
  "customerId": "...",
  "currencyCode": "USD",
  "lineItems": [...],
  "cancellingItems": [...],
  "externalReferenceId": "..."
}
```

**Response:** `Order` object — HTTP 201

`Order.lineItems[].flexDiscounts` shows per-line-item discount validation result:
```json
{ "id": "...", "code": "DISCOUNT_CODE", "result": "SUCCESS" }
```

---

### GET /api/subscriptions — Get Subscriptions

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `getCustomerSubscriptions` or `getSubscriptionDetails` |
| `customerId` | Yes | Customer ID |
| `subscriptionId` | When `type=getSubscriptionDetails` | Subscription ID |

**Response (`type=getCustomerSubscriptions`):** Array of `Subscription` objects

**Response (`type=getSubscriptionDetails`):** Single `Subscription` object

`Subscription` shape:
```json
{
  "subscriptionId": "...",
  "offerId": "...",
  "status": "1000",
  "autoRenewal": { "enabled": true, "renewalQuantity": 10, "flexDiscountCodes": ["CODE"] },
  "renewalDate": "...",
  "currencyCode": "USD"
}
```

---

### POST /api/subscriptions — Create Subscription

**Body:**
```json
{
  "customerId": "...",
  "offerId": "...",
  "autoRenewal": { "enabled": true, "renewalQuantity": 10, "flexDiscountCodes": ["CODE"] }
}
```
**Response:** Created subscription object — HTTP 201

---

### PATCH /api/subscriptions — Update Subscription

**Body:**
```json
{
  "customerId": "...",
  "subscriptionId": "...",
  "autoRenewal": { "autoRenewal": true, "renewalQuantity": 10, "flexDiscountCodes": ["CODE"] }
}
```

**Query param:** `reset-flex-discount-codes=true` — appended by the controller when clearing flex discount codes from a subscription.

**Response:** Updated subscription object — HTTP 200

---

### GET /api/resellers — Get Resellers

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `getAllResellers` or `getResellerDetails` |
| `id` | When `type=getResellerDetails` | Reseller ID |
| `offset` | No | Default: 0 |
| `limit` | No | Default: 50 |

**Response (`type=getAllResellers`):**
```json
{ "resellers": [...], "totalCount": 100, "count": 50, "offset": 0, "limit": 50, "hasMore": true }
```

---

### POST /api/resellers — Create Reseller

**Body:** Reseller object (validated against `ResellerSchema`)
**Response:** Created reseller — HTTP 201

---

### POST /api/pricelist — Get Price List

**Body:**
```json
{
  "currency": "USD",
  "country": "US",
  "marketSegment": "COM",
  "offset": 0,
  "limit": 100
}
```
**Response:** Paginated pricelist with `hasMore` flag

---

### GET /api/offer-switch-paths — Get Switch Paths

**Query params:** `market-segment`, `country`, `language`, `offer-id`, `subscription-id`, `customer-id`, `limit`, `offset`

**Response:** Paginated offer switch paths from Partner API

---

### GET /api/partnerDetails — Get Partner Details

**Query params:** `type` — `marketSegments` | `currencies` | (omit for full details)

**Response (default):**
```json
{
  "data": {
    "partnerName": "...",
    "marketSegments": [{ "programType": "VIPMP", "marketSegment": "COM" }],
    "currencies": [{ "priceRegion": "NA", "currency": "USD" }]
  }
}
```

---

### GET /api/search — Search Resellers or Customers

**Query params:**

| Param | Required | Description |
|-------|----------|-------------|
| `type` | Yes | `reseller` or `customer` |
| `query` | Yes | Search string — wildcarded as `*query*` on the API call |
| `resellerId` | When `type=customer` | Reseller scope for customer search |
| `offset` | No | Default: 0 |
| `limit` | No | Default: 50 |

**Behaviour:** Runs name search + numeric ID lookup in parallel via `Promise.allSettled`; deduplicates by ID before returning.

---

### GET /api/recommendation — Get Recommendations

**Query params:** `customerId`, `recommendationContext`, `country`, `language`

**Response:** Recommendation response from Partner API

---

### GET /api/env — Get Public Env Vars

**Response:** JSON object of public runtime environment variables (currently empty object). Cached in memory for 60 seconds.
