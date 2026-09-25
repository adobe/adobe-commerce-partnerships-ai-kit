# CONNECTORS.md — adobe-commerce-partnerships-ref-app

adobe-commerce-partnerships-ref-app has two outbound dependencies. Neither uses a dedicated HTTP client library or wrapper
class — every call is native `fetch()`, inlined directly inside the controller function that
needs it. There is no connector abstraction layer (no `*Client.ts` / `*Connector.ts` file).

---

### 1. Adobe Commerce Partner API — external partner platform (customers, resellers, orders, subscriptions, price lists, recommendations)

| Field | Value |
|---|---|
| **SDK / library** | None — native `fetch()` |
| **Base URL source** | `process.env.PARTNER_API_BASE_URL` (e.g. `https://partnersandbox-stage.adobe.io`), read once per controller module at import time |
| **Auth mechanism** | `Authorization: Bearer <access_token>` (token obtained from Connector 2) + `x-api-key: <PARTNER_CLIENT_ID>` header on every request |
| **Connect timeout** | none configured |
| **Read/write timeout** | none configured (no `AbortController`/`signal` on any server-side outbound call) |
| **Retry config** | none configured — single attempt per call |
| **Error handling** | Any non-`response.ok` result is immediately thrown as `ApiError(message, response.status, requestId)`. A non-JSON response body is caught and thrown as `ApiError('Non-JSON response from Adobe API'/'from API', status, requestId)`. Zod validation failures on the parsed response are thrown as `ApiError('<Resource> data validation failed', 500, requestId)`. |
| **Unavailability posture** | Fail-fast — no fallback value, no circuit breaker, no degraded mode. Every controller function propagates the `ApiError` straight to its calling `pages/api/*.ts` route, which maps it to an HTTP response. |
| **Correlation** | Every request sends fresh `X-Correlation-Id` / `X-Request-Id` headers (`uuidv4()`), generated per call — not propagated from the inbound request. The upstream's own `X-Request-Id` response header is read back via `extractRequestIdFromResponse()` (`utils/commonUtils.ts:158`) and forwarded to the adobe-commerce-partnerships-ref-app API response via `forwardRequestIdHeader()`. |
| **Used by** | `controllers/customerController.ts`, `controllers/orderController.ts`, `controllers/resellerController.ts`, `controllers/subscriptionController.ts`, `controllers/priceController.ts`, `controllers/recommendationController.ts` |
| **Path construction** | Each controller builds its own URL template from `baseUrl`, e.g. `${baseUrl}/v3/customers/${id}`, `${baseUrl}/v3/pricelist`, `${baseUrl}/v3/resellers/${id}/customers`. No shared path-builder utility. |

> **Inconsistency noted as-is:** `subscriptionController.ts::getSubscriptionDetails` builds its URL as `` `${baseUrl}/customers/...` `` (missing the `/v3` prefix that every other subscription call uses) — documented here as observed in code, not corrected.

---

### 2. Adobe IMS OAuth Token Service — client-credentials token issuance

| Field | Value |
|---|---|
| **SDK / library** | None — native `fetch()`, in `utils/imsTokenService.ts::getAccessToken` |
| **Base URL source** | `process.env.IMS_TOKEN_URL` + `/ims/token/v2` |
| **Auth mechanism** | OAuth2 `client_credentials` grant — `client_id`/`client_secret` from `PARTNER_CLIENT_ID`/`PARTNER_CLIENT_SECRET`, `scope` from `IMS_SCOPES` (defaults to `openid,AdobeID,read_organizations`), sent as URL-encoded POST body |
| **Connect timeout** | none configured |
| **Read/write timeout** | none configured |
| **Retry config** | none configured |
| **Error handling** | Missing `IMS_TOKEN_URL`, `PARTNER_CLIENT_ID`, or `PARTNER_CLIENT_SECRET` throws `ApiError(..., 500)` before any network call. A non-OK token response throws `ApiError('Failed to fetch IMS access token: <body>', response.status)`. |
| **Unavailability posture** | Fail-fast — no fallback token, no retry |
| **Caching** | Module-level in-memory variables `cachedToken` / `tokenExpiry` (process lifetime, not shared across instances/replicas). Token is reused across requests until `Date.now() >= tokenExpiry`. Expiry is computed as `now + expires_in - 5 minutes` (5-minute safety buffer before actual expiry). |
| **Called from** | `utils/commonUtils.ts::handlePrerequisites` — invoked at the top of every `pages/api/*.ts` handler except `partnerDetails.ts` (env-only, no upstream call) and `env.ts` (static config) |

> **Note:** the code comment at `imsTokenService.ts:40` says "`expires_in` is in ms" but IMS token endpoints conventionally return `expires_in` in seconds; documented as written in code, not corrected.

---

## Cross-Reference: Connector → Consumers

| Connector | Consumed by (controller) |
|---|---|
| Adobe Commerce Partner API | customerController, orderController, resellerController, subscriptionController, priceController, recommendationController |
| Adobe IMS OAuth Token Service | `handlePrerequisites` (called by every `pages/api/*.ts` route except `partnerDetails.ts`, `env.ts`) |