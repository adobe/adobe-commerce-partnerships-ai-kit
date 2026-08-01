# Connectors — Bridge

Bridge makes outbound calls to two external services: Adobe IMS (for token issuance) and the VIP Marketplace Partner API (for all business operations).

---

### 1. IMSToken — OAuth Token Endpoint

**File:** `utils/imsTokenService.ts`

| Field | Value |
|-------|-------|
| Purpose | Obtain an OAuth 2.0 bearer token using client credentials |
| Auth mechanism | OAuth 2.0 `client_credentials` grant — no auth header on the token request itself |
| Token URL | `process.env.IMS_TOKEN_URL` \|\| `https://ims-na1-stg1.adobelogin.com/ims/token/v2` |
| Client ID | `process.env.PARTNER_CLIENT_ID` |
| Client secret | `process.env.PARTNER_CLIENT_SECRET` |
| Scopes | `process.env.IMS_SCOPES` \|\| `openid,AdobeID,read_organizations` |
| HTTP client | Native `fetch` (Node.js built-in) |
| Connect timeout | none configured |
| Read/write timeout | none configured |
| Retry config | none configured |
| Token caching | Module-level variables `cachedToken` and `tokenExpiry`; refreshed when `Date.now() >= tokenExpiry`. Buffer: `expires_in - 5 min`. Process restart clears cache. |
| Error handling | Non-OK response → `ApiError(message, response.status)` |
| Unavailability posture | Fail-fast — throws `ApiError(500)` which surfaces to the caller as a 500 response |

**Token request shape:**
```
POST {IMS_TOKEN_URL}?grant_type=client_credentials&client_id={id}&client_secret={secret}&scope={scopes}
```

**How it is invoked:**  
`handlePrerequisites(req)` in `utils/commonUtils.ts` calls `getAccessToken()` at the start of every API route handler. The returned token is passed explicitly to every controller function.

---

### 2. PartnerAPI — VIP Marketplace Partner API

**Files:** `controllers/customerController.ts`, `controllers/orderController.ts`, `controllers/subscriptionController.ts`, `controllers/resellerController.ts`, `controllers/priceController.ts`, `controllers/switchOrderController.ts`, `controllers/recommendationController.ts`

| Field | Value |
|-------|-------|
| Purpose | All VIP Marketplace business operations: customers, orders, subscriptions, resellers, pricing, recommendations |
| Auth mechanism | Bearer token — obtained from IMS connector above |
| Base URL | `process.env.PARTNER_API_BASE_URL` |
| HTTP client | Native `fetch` (Node.js built-in) |
| Connect timeout | none configured |
| Read/write timeout | none configured |
| Retry config | none configured |
| Error handling | Non-OK response → `ApiError(message, response.status, requestId)` |
| Unavailability posture | Fail-fast — ApiError propagates to the API route handler which returns the status to the client |

**Standard request headers (all controllers):**
```
Authorization: Bearer {access_token}
Content-Type: application/json
x-api-key: {PARTNER_CLIENT_ID}
X-Correlation-Id: {uuidv4()}
X-Request-Id: {uuidv4()}     (some controllers use X-Request-Id, some only X-Correlation-Id)
```

**Request ID propagation:**  
`extractRequestIdFromResponse(result)` in `commonUtils.ts` reads the `X-Request-Id` header (case-insensitive) from the Partner API response. The ID is attached to `ApiError.requestId` and forwarded to the API route caller via `forwardRequestIdHeader(res, requestId)`.

**Per-resource URL patterns:**

| Resource | URL Pattern |
|----------|-------------|
| Customers | `{base}/v3/customers` / `{base}/v3/customers/{id}` |
| Resellers | `{base}/v3/resellers` / `{base}/v3/resellers/{id}` |
| Reseller customers | `{base}/v3/resellers/{resellerId}/customers` |
| Orders | `{base}/v3/customers/{customerId}/orders` / `…/{orderId}` |
| Subscriptions | `{base}/v3/customers/{customerId}/subscriptions` / `…/{subscriptionId}` |
| Pricelist | `{base}/v3/pricelist` |
| Switch paths | `{base}/v3/offer-switch-paths` |
| Recommendations | `{base}/v3/recommendations` |

**Flex discount codes on orders:**  
`lineItems[].flexDiscountCodes: string[]` — passed through as-is from the request body. The Partner API validates and returns per-line-item results in `lineItems[].flexDiscounts[].result: "SUCCESS" | "FAILURE"`.

**Flex discount codes on subscriptions:**  
`autoRenewal.flexDiscountCodes: string[]` — set on `PATCH /v3/customers/{id}/subscriptions/{subId}`. Cleared by appending `?reset-flex-discount-codes=true` to the URL.
