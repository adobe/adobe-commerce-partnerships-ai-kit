# LLD — Anytime Upgrade (Backend)
---

## §1 — Summary

Anytime Upgrade lets a partner move a customer's active subscription to a higher-tier product before its
anniversary date. The backend work is an extension of this service's existing **Orders** capability plus one
new **discovery** capability — there is no new persistent state, since the service is stateless and VIP MP
remains the system of record.

Two new/modified endpoints:

- **`GET /api/offer-switch-paths`** (new route) — proxies `GET /v3/offer-switch-paths`, keyed by
  `customerId` + `subscriptionId`, to discover which upgrade targets are available for a subscription.
- **`POST /api/orders`** (extended) — two new `type` values, `PreviewSwitch` and `Switch`, mapping to the
  API spec's `PREVIEW_SWITCH` and `SWITCH` order types, following the exact same shared `createOrder` sink
  already used by `NEW`/`RETURN`/`PREVIEW`/`PREVIEW_RENEWAL`/`RENEWAL`.

End-to-end flow: the UI calls the discovery endpoint to decide whether to show an "Upgrade" action, then
drives a preview → place-order sequence against `/api/orders` using the same request/response envelope as
every other order type in this codebase. `PREVIEW_REVERT_SWITCH`/`REVERT_SWITCH` from the API spec are
explicitly out of scope — see §7.

---

## §2 — Data Flow

### Operation: Discover upgrade paths for a subscription

```
Operation: UI renders a subscription card and needs to know if an upgrade is available

  → GET /api/offer-switch-paths?customerId=&subscriptionId=(&offset=&limit=)
  → pages/api/offer-switch-paths.ts (handler)
  → controllers/offerSwitchPathController.ts · getOfferSwitchPaths(customerId, subscriptionId, accessToken, offset, limit)
  → GET ${PARTNER_API_BASE_URL}/v3/offer-switch-paths?customer-id=&subscription-id=&offset=&limit=
  ← response: { totalCount, count, offset, limit, productUpgrades: [{ sourceBaseOfferId, targetList: [{ targetBaseOfferId, sequence, switchType }] }] }
  ← handler maps to: OfferSwitchPathsResponseSchema (models/OfferSwitchPath.ts)
  ← route returns: HTTP 200 + OfferSwitchPathsResponse body
```

Purely read-through — no transformation beyond Zod validation; `hasMore` is not computed here because the
upstream response's own `totalCount`/`count`/`offset`/`limit` are already sufficient for the UI's needs (the
UI only ever checks `productUpgrades.length > 0`, per the experience card).

### Operation: Preview a switch (upgrade) order

```
Operation: UI loads the Upgrade Order Summary screen (auto-fires on page load) or user changes quantity/discount code

  → POST /api/orders?type=PreviewSwitch&fetch-price=true
  → pages/api/orders.ts (handler)
  → controllers/orderController.ts · createPreviewSwitchOrder(customerId, currencyCode, lineItems, cancellingItems, accessToken, externalReferenceId, queryParams)
  → controllers/orderController.ts · createOrder(body, accessToken, queryParams)  [shared sink, unchanged]
  → POST ${PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders?fetch-price=true
  ← response: { orderType: 'PREVIEW_SWITCH', lineItems: [{ offerId, quantity, proratedDays, pricing }], cancellingItems: [{ offerId, quantity, subscriptionId, pricing, referenceLineItemNumber }], pricingSummary: [{ totalLineItemPartnerPrice, currencyCode }] }
  ← handler maps to: OrderSchema (models/Order.ts, extended — see §3)
  ← route returns: HTTP 201 + Order body (no order is persisted upstream for a preview, per API spec)
```

### Operation: Place a switch (upgrade) order

```
Operation: user presses "Place order" on the Upgrade Order Summary screen

  → POST /api/orders?type=Switch
  → pages/api/orders.ts (handler)
  → controllers/orderController.ts · createSwitchOrder(customerId, currencyCode, lineItems, cancellingItems, accessToken, externalReferenceId, queryParams)
  → controllers/orderController.ts · createOrder(body, accessToken, queryParams)  [shared sink, unchanged]
  → POST ${PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders
  ← response: { orderType: 'SWITCH', orderId, status, lineItems, cancellingItems }
  ← handler maps to: OrderSchema (models/Order.ts, extended — see §3)
  ← route returns: HTTP 201 + Order body
```

`queryParams` on the SWITCH call is the same generic `restQuery` mechanism `PREVIEW`/`PREVIEW_RENEWAL` already
use — it transparently supports the API spec's optional `reassign-users=true` param with zero new code (see
§5, Decision 6).

### Operation: Verify a switch order (no change)

`GET /api/orders?type=getOrders` and `GET /api/orders?type=getOrdersHistory` already fetch any order by ID or
list a customer's history regardless of `orderType` (`getOrders`/`getOrdersHistoryForCustomer` in
`controllers/orderController.ts` do not filter or validate `orderType` against the strict enum on the read
path — only `createOrder`'s response validation does). **No backend change is required** for verifying a
placed switch order; the UI can already call `GET /api/orders?type=getOrders&customerId=&orderId=` for a
switch order's `orderId` today.

---

## §3 — Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `models/OfferSwitchPath.ts` | CREATE | Model | Zod schemas for the new `/v3/offer-switch-paths` request/response shape |
| `controllers/offerSwitchPathController.ts` | CREATE | Controller | New connector + business logic for the discovery capability |
| `pages/api/offer-switch-paths.ts` | CREATE | Route | New inbound route exposing discovery to the UI |
| `models/Order.ts` | MODIFY | Model | Extend `OrderTypeEnum`/`OrderSchema`; add `CancellingItemSchema`, `PreviewSwitchOrderSchema`, `SwitchOrderSchema` |
| `controllers/orderController.ts` | MODIFY | Controller | Add `createPreviewSwitchOrder` / `createSwitchOrder`, delegating to the existing `createOrder` sink |
| `pages/api/orders.ts` | MODIFY | Route | Dispatch the two new `type` values to the new controller functions |
| `utils/constants.ts` | MODIFY | Constants | Add `PREVIEW_SWITCH` / `SWITCH` entries to `ORDER_API_TYPE` |

---

### `models/OfferSwitchPath.ts` — CREATE

New file, following the `models/<Resource>.ts` convention (Zod schema + colocated `z.infer` type, per
`CODE_PATTERNS.md §1/§2`).

```ts
import { z } from 'zod';

export const SwitchTypeEnum = z.enum(['PARTIAL_ALLOWED', 'FULL_ONLY']);

export const TargetOfferSchema = z
  .object({
    targetBaseOfferId: z.string(),
    sequence: z.number(),
    switchType: SwitchTypeEnum,
  })
  .passthrough();

export const ProductUpgradeSchema = z
  .object({
    sourceBaseOfferId: z.string(),
    targetList: z.array(TargetOfferSchema),
  })
  .passthrough();

export const OfferSwitchPathsResponseSchema = z
  .object({
    totalCount: z.number(),
    count: z.number(),
    offset: z.number(),
    limit: z.number(),
    productUpgrades: z.array(ProductUpgradeSchema).optional().default([]),
  })
  .passthrough();

export type TargetOffer = z.infer<typeof TargetOfferSchema>;
export type ProductUpgrade = z.infer<typeof ProductUpgradeSchema>;
export type OfferSwitchPathsResponse = z.infer<typeof OfferSwitchPathsResponseSchema>;
```

- `.passthrough()` on all three object schemas, matching every other response schema in `models/Order.ts` /
  `models/CustomerDetails.ts` — tolerates additional upstream fields without failing validation.
- `productUpgrades` defaults to `[]` rather than being required, so a customer/subscription with zero
  upgrade paths still validates cleanly (API spec shows the field as optional/absent in that case).
- No request schema is added here — the route does lightweight presence checks on `customerId`/
  `subscriptionId` directly (matching `pages/api/customers.ts`'s convention of route-level presence checks
  for simple query-param-driven GETs), not a Zod schema, since there is no request *body* to validate.

---

### `controllers/offerSwitchPathController.ts` — CREATE

New file, one function, following the exact GET-with-query-params-and-pagination shape of
`customerController.getAllCustomers`.

```ts
import { OfferSwitchPathsResponseSchema, OfferSwitchPathsResponse } from '../models/OfferSwitchPath';
import { ApiError } from '../utils/apiError';
import { v4 as uuidv4 } from 'uuid';
import { createControllerLogger, logRequest, logResponse, logErrorResponse } from '../utils/logger';
import { extractRequestIdFromResponse } from '../utils/commonUtils';
import type { BackendResult } from '../models/Result';
import { HTTP_METHOD } from '../utils/constants';

const baseUrl = `${process.env.PARTNER_API_BASE_URL}/v3/offer-switch-paths`;

export async function getOfferSwitchPaths(
  customerId: string,
  subscriptionId: string,
  accessToken: string,
  offset: number = 0,
  limit: number = 20
): Promise<BackendResult<OfferSwitchPathsResponse>> {
  const logger = createControllerLogger('offerSwitchPathController', 'getOfferSwitchPaths');
  const apiKey = process.env.PARTNER_CLIENT_ID;

  logRequest(logger, { customerId, subscriptionId, offset, limit });

  if (!accessToken) throw new ApiError('Missing ACCESS_TOKEN', 500);
  if (!apiKey) throw new ApiError('Missing ADOBE_API_KEY', 500);

  const url =
    `${baseUrl}?customer-id=${encodeURIComponent(customerId)}` +
    `&subscription-id=${encodeURIComponent(subscriptionId)}` +
    `&offset=${offset}&limit=${limit}`;

  const result = await fetch(url, {
    method: HTTP_METHOD.GET,
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      'X-Correlation-Id': uuidv4(),
      'X-Request-Id': uuidv4(),
    },
  });

  const text = await result.text();
  const requestId = extractRequestIdFromResponse(result);

  let data;
  try {
    data = JSON.parse(text);
  } catch (e) {
    logErrorResponse(logger, { error: 'Parse failed', text }, requestId, { status: result.status });
    throw new ApiError('Non-JSON response from Adobe API', result.status, requestId);
  }

  if (!result.ok) {
    logErrorResponse(logger, data, requestId, { status: result.status, customerId, subscriptionId });
    throw new ApiError(data.message || data.error || 'Failed to fetch offer switch paths', result.status, requestId);
  }

  logResponse(logger, data, requestId, { customerId, subscriptionId });

  try {
    const validatedData = OfferSwitchPathsResponseSchema.parse(data);
    return { data: validatedData, requestId };
  } catch (err: any) {
    logErrorResponse(logger, data, requestId, { status: result.status, validationError: String(err) });
    throw new ApiError('Offer switch paths data validation failed', 500, requestId);
  }
}
```

- Upstream query params use the API spec's hyphenated names (`customer-id`, `subscription-id`); the
  function's own parameter names stay camelCase, matching this codebase's internal-vs-upstream naming split
  already used in `customerController.getAllCustomers` (`resellerId` param → `/v3/resellers/{resellerId}/...`
  path segment).
- Fail-closed on transport/HTTP/schema errors — no per-item skip-and-log tolerance like
  `getAllCustomers`/`findCustomerByName`, because `productUpgrades`/`targetList` is not a paginated list of
  independently-fallible records the way a customer list is; a malformed response here means the whole
  discovery call is untrustworthy (see §5, Decision 7).
- `market-segment`/`country`/`language`/`offer-id` query params from the API spec are **not** implemented —
  see §7 (Out of Scope).

---

### `pages/api/offer-switch-paths.ts` — CREATE

New route, following the single-purpose GET-only pattern of `pages/api/recommendation.ts` (no `type` query
param dispatch needed, since this route has exactly one operation).

| Field | Value |
|-------|-------|
| Method | GET only (405 + `Allow: GET` otherwise, matching `pages/api/env.ts`'s convention) |
| Auth | Server-side IMS Bearer via `handlePrerequisites` (same as every Partner-API-backed route except `/api/partnerDetails`/`/api/env`) |

**Query parameters:**

| Param | Required | Type | Notes |
|-------|----------|------|-------|
| `customerId` | Yes | string | 400 if missing or an array |
| `subscriptionId` | Yes | string | 400 if missing or an array |
| `offset` | No | string → parsed int | defaults to `0` |
| `limit` | No | string → parsed int | defaults to `20` (matches the API spec's own documented default) |

**Response 200:** `OfferSwitchPathsResponse` shape (`models/OfferSwitchPath.ts`), unwrapped from the
`{ data, requestId }` envelope — matching the unwrapping convention every other route in this codebase uses
(`responseData = result.data`).

**Error conditions → status:**

| Condition | Status |
|-----------|--------|
| Missing/invalid `customerId` or `subscriptionId` | 400 |
| Upstream 4xx/5xx | forwarded as-is via `ApiError` |
| Non-JSON upstream response, or response schema validation failure | 500 |
| Method other than GET | 405 |

```ts
import type { NextApiRequest, NextApiResponse } from 'next';
import { getOfferSwitchPaths } from '../../controllers/offerSwitchPathController';
import { ApiError } from '../../utils/apiError';
import { forwardRequestIdHeader, handlePrerequisites } from '../../utils/commonUtils';
import { HTTP_METHOD } from '../../utils/constants';
import { createAPILogger } from '../../utils/logger';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const logger = createAPILogger(req);

  if (req.method !== HTTP_METHOD.GET) {
    res.setHeader('Allow', 'GET');
    return res.status(405).end(`Method ${req.method} Not Allowed`);
  }

  try {
    const { accessToken } = await handlePrerequisites(req);
    const { customerId, subscriptionId, offset, limit } = req.query;

    if (!customerId || Array.isArray(customerId)) {
      throw new ApiError('customerId is required', 400);
    }
    if (!subscriptionId || Array.isArray(subscriptionId)) {
      throw new ApiError('subscriptionId is required', 400);
    }

    const offsetNum = offset ? parseInt(Array.isArray(offset) ? offset[0] : offset, 10) : 0;
    const limitNum = limit ? parseInt(Array.isArray(limit) ? limit[0] : limit, 10) : 20;

    const result = await getOfferSwitchPaths(customerId, subscriptionId, accessToken, offsetNum, limitNum);

    forwardRequestIdHeader(res, result.requestId);
    return res.status(200).json(result.data);
  } catch (error: any) {
    logger.error({ error: error.message, stack: error.stack }, 'Offer switch paths API error');
    if (error instanceof ApiError) {
      if (error.requestId) forwardRequestIdHeader(res, error.requestId);
      return res.status(error.status).json({ error: error.message });
    }
    return res.status(500).json({ error: 'Internal Server Error' });
  }
}
```

---

### `models/Order.ts` — MODIFY

Current state (read in full — see `models/Order.ts:1-207`): `OrderTypeEnum` is
`z.enum(['NEW', 'RETURN', 'PREVIEW', 'PREVIEW_RENEWAL', 'RENEWAL'])`; `OrderSchema` has no `cancellingItems`
field; there is no `CancellingItemSchema`, `PreviewSwitchOrderSchema`, or `SwitchOrderSchema`.

Required changes:

1. **Extend `OrderTypeEnum`** (line 60) to include `'SWITCH'` and `'PREVIEW_SWITCH'`:
   ```ts
   const OrderTypeEnum = z.enum(['NEW', 'RETURN', 'PREVIEW', 'PREVIEW_RENEWAL', 'RENEWAL', 'PREVIEW_SWITCH', 'SWITCH']);
   ```
   This is mandatory, not cosmetic — `createOrder` (unchanged, shared by every order type) calls
   `OrderSchema.parse(data)` on every response, and `OrderSchema.orderType` is typed against this enum.
   Without this change, every switch/preview-switch order would fail response validation with a spurious
   `ApiError('Order data validation failed', 500)` even though the upstream call succeeded.

2. **Add `CancellingItemSchema`**, reusing `LineItemSchema`'s existing dual-purpose pattern (required fields
   common to request+response; response-only fields optional):
   ```ts
   export const CancellingItemSchema = z
     .object({
       extLineItemNumber: z.number(),
       referenceLineItemNumber: z.number(),
       subscriptionId: z.string(),
       quantity: z.number(),
       offerId: z.string().optional(),       // response-only
       pricing: PricingSchema.optional(),     // response-only
     })
     .passthrough();
   ```

3. **Add `PreviewSwitchOrderSchema`** and **`SwitchOrderSchema`**, matching the existing per-order-type
   schema convention (`z.literal` discriminator, not a `z.discriminatedUnion` — see `CODE_PATTERNS.md §4`):
   ```ts
   export const PreviewSwitchOrderSchema = z.object({
     customerId: z.string(),
     orderType: z.literal('PREVIEW_SWITCH'),
     externalReferenceId: z.string().optional(),
     currencyCode: z.string(),
     lineItems: z.array(LineItemSchema),
     cancellingItems: z.array(CancellingItemSchema),
   });

   export const SwitchOrderSchema = z.object({
     customerId: z.string(),
     orderType: z.literal('SWITCH'),
     externalReferenceId: z.string().optional(),
     currencyCode: z.string(),
     lineItems: z.array(LineItemSchema),
     cancellingItems: z.array(CancellingItemSchema),
   });
   ```
   `externalReferenceId` is optional on both, matching `NewOrderSchema`'s existing looseness (only
   `ReturnOrderSchema`/`RenewalOrderSchema` require it) — the API spec itself marks it "Not Required".

4. **Add `cancellingItems` to `OrderSchema`** (the shared response schema), immediately after the existing
   `lineItems` field:
   ```ts
   cancellingItems: z.array(CancellingItemSchema).optional(),
   ```
   Optional because non-switch order types never populate it.

5. **Export new types**: `export type CancellingItem = z.infer<typeof CancellingItemSchema>;` alongside the
   existing `Order`/`LineItem`/`Pricing` exports at the bottom of the file.

---

### `controllers/orderController.ts` — MODIFY

Current state (read in full — see `controllers/orderController.ts:1-443`): five `create*Order` wrapper
functions (`createNewOrder`, `createReturnOrder`, `createPreviewOrder`, `createPreviewRenewalOrder`,
`createRenewalOrder`) all follow the same shape — build a typed body with a literal `orderType`, validate it
with the matching Zod schema (throwing `ApiError(400)` on failure), then delegate to the shared `createOrder`.
`createOrder` itself is untouched by this change.

Add two new functions, inserted after `createPreviewOrder` (to sit next to its closest sibling in behavior):

```ts
// 6. Preview a Switch (Upgrade) Order
export async function createPreviewSwitchOrder(
  customerId: string,
  currencyCode: string,
  lineItems: any[],
  cancellingItems: any[],
  accessToken: string,
  externalReferenceId?: string,
  queryParams?: any
) {
  const logger = createControllerLogger('orderController', 'createPreviewSwitchOrder');
  logger.info('Request Received for orderController - createPreviewSwitchOrder');

  const body: any = { customerId, orderType: 'PREVIEW_SWITCH', currencyCode, lineItems, cancellingItems };
  if (externalReferenceId) body.externalReferenceId = externalReferenceId;

  try {
    PreviewSwitchOrderSchema.parse(body);
  } catch (err) {
    logger.error({ err }, 'Input validation failed');
    throw new ApiError('Input validation failed', 400);
  }
  return createOrder(body, accessToken, queryParams);
}

// 7. Place a Switch (Upgrade) Order
export async function createSwitchOrder(
  customerId: string,
  currencyCode: string,
  lineItems: any[],
  cancellingItems: any[],
  accessToken: string,
  externalReferenceId?: string,
  queryParams?: any
) {
  const logger = createControllerLogger('orderController', 'createSwitchOrder');
  logger.info('Request Received for orderController - createSwitchOrder');

  const body: any = { customerId, orderType: 'SWITCH', currencyCode, lineItems, cancellingItems };
  if (externalReferenceId) body.externalReferenceId = externalReferenceId;

  try {
    SwitchOrderSchema.parse(body);
  } catch (err) {
    logger.error({ err }, 'Input validation failed');
    throw new ApiError('Input validation failed', 400);
  }
  return createOrder(body, accessToken, queryParams);
}
```

Also update the top-of-file import block to bring in the two new schemas:

```ts
import {
  NewOrderSchema,
  ReturnOrderSchema,
  RenewalOrderSchema,
  PreviewOrderSchema,
  PreviewRenewalOrderSchema,
  PreviewSwitchOrderSchema,
  SwitchOrderSchema,
} from '../models/Order';
```

`createOrder` (unchanged) already accepts an optional `queryParams` and appends it as a URL-encoded query
string to the outbound POST — both `fetch-price=true` (preview) and the API spec's optional `reassign-users`
(final switch) flow through this existing mechanism with no new code (see §5, Decision 6).

---

### `pages/api/orders.ts` — MODIFY

Current state (read in full — see `pages/api/orders.ts:1-153`): the `POST` branch is an `if`/`else if` chain
on `req.query.type`, each branch destructuring the relevant fields off `req.body` and calling the matching
controller function; `PREVIEW`/`PREVIEW_RENEWAL` additionally strip `type` off `req.query` into `restQuery`
and forward it as `queryParams`.

1. **Update the import** from `controllers/orderController` to add `createPreviewSwitchOrder`,
   `createSwitchOrder`.

2. **Add two new `else if` branches**, inserted after the existing `PREVIEW` branch (to mirror its
   `restQuery`-forwarding shape) and before the final `else`:

```ts
} else if (type === ORDER_API_TYPE.PREVIEW_SWITCH) {
  const { customerId, externalReferenceId, currencyCode, lineItems, cancellingItems } = orderData;
  const { type: _type, ...restQuery } = req.query;
  result = await createPreviewSwitchOrder(
    customerId,
    currencyCode,
    lineItems,
    cancellingItems,
    accessToken,
    externalReferenceId,
    restQuery
  );
  statusCode = 201;
  responseData = result.data;
} else if (type === ORDER_API_TYPE.SWITCH) {
  const { customerId, externalReferenceId, currencyCode, lineItems, cancellingItems } = orderData;
  const { type: _type, ...restQuery } = req.query;
  result = await createSwitchOrder(
    customerId,
    currencyCode,
    lineItems,
    cancellingItems,
    accessToken,
    externalReferenceId,
    restQuery
  );
  statusCode = 201;
  responseData = result.data;
}
```

`statusCode = 201` for `PREVIEW_SWITCH` matches this route's existing (if debatable) convention of returning
201 for `PREVIEW`/`PREVIEW_RENEWAL` even though no order is actually created — kept for consistency rather
than introducing a one-off 200 for just this order type (see §5, Decision 8).

No changes to the existing `catch` block — `orderController.createOrder`'s error shape
(`JSON.stringify(upstreamErrorBody)` re-parsed by this route) is already generic to all order types,
including SWITCH.

---

### `utils/constants.ts` — MODIFY

Current state (read in full — see `utils/constants.ts:62-70`): `ORDER_API_TYPE` has `GET_ORDERS`,
`GET_ORDERS_HISTORY`, `NEW`, `RETURN`, `PREVIEW`, `PREVIEW_RENEWAL`, `RENEWAL_ORDER`.

Add two entries, following the existing PascalCase-word string-value convention (not `SCREAMING_SNAKE_CASE` —
that convention is reserved for env var keys per `CODE_PATTERNS.md §2`):

```ts
export const ORDER_API_TYPE = {
  GET_ORDERS: 'getOrders',
  GET_ORDERS_HISTORY: 'getOrdersHistory',
  NEW: 'NEW',
  RETURN: 'Return',
  PREVIEW: 'Preview',
  PREVIEW_RENEWAL: 'PreviewRenewal',
  RENEWAL_ORDER: 'RenewalOrder',
  PREVIEW_SWITCH: 'PreviewSwitch',
  SWITCH: 'Switch',
} as const;
```

---

## §4 — DB Changes

No DB changes — service is stateless (`docs/ai-kit/service-cards/backend/DB_SCHEMA.md`: "this service owns no
data store"). VIP MP remains the system of record for switch orders, exactly as for every other order type.

---

## §5 — Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|--------------|
| Reuse the existing `createOrder` sink for `PREVIEW_SWITCH`/`SWITCH` rather than a new HTTP-calling function | Matches the established one-sink-per-resource pattern every other order type already follows (`createNewOrder`/`createPreviewOrder`/etc. all delegate to it) | None — this is the path of least deviation from existing convention | `controllers/orderController.ts` existing structure |
| New standalone `offerSwitchPathController.ts` + `pages/api/offer-switch-paths.ts`, not folded into `orderController`/`orders.ts` | `/v3/offer-switch-paths` is a distinct upstream resource, not an order operation — matches the one-controller-per-resource naming convention (`CODE_PATTERNS.md §1`, `<resource>Controller.ts`) | One more file pair vs. topical proximity to Orders | `CODE_PATTERNS.md` package-layout table |
| Extend `OrderTypeEnum` to include `SWITCH`/`PREVIEW_SWITCH` | `createOrder`'s shared response validation (`OrderSchema.parse`) would otherwise throw a spurious 500 on every successful switch order, since `orderType` is checked against this enum | None — required for correctness, not optional | `models/Order.ts` `OrderTypeEnum` |
| Add a typed `cancellingItems` field to `OrderSchema` rather than relying on `.passthrough()` alone | Matches this schema's existing posture of typing every field the API spec documents on the response (see `lineItems`, `pricingSummary`, `eligibleOffers`); `.passthrough()` alone would still accept the payload but leave `cancellingItems` untyped for any downstream/UI consumer | Slightly larger schema | `models/Order.ts` `OrderSchema` |
| Single `CancellingItemSchema` reused for both outbound request and inbound response, with response-only fields (`offerId`, `pricing`) marked optional | Mirrors `LineItemSchema`'s existing dual-purpose precedent exactly — avoids introducing a second, inconsistent modeling convention for line items | Response-only fields are typed optional even though the API always returns them on a switch-order response | `models/Order.ts` `LineItemSchema` precedent |
| `reassign-users` (API spec's optional query param on the final `SWITCH` call) is supported only via the existing generic `queryParams` passthrough on `createOrder`/`createSwitchOrder` — no dedicated param, constant, or validation | The experience card confirms no UI surface in this app sends it today; the generic passthrough mechanism (already used by `PREVIEW`/`PREVIEW_RENEWAL` for `fetch-price=true`) covers it for free if a future caller needs it, without speculative dedicated code | No type-level documentation or validation of this specific param — a caller could pass any string | `controllers/orderController.ts createOrder`'s existing `queryParams` mechanism — not enforced beyond that |
| `offerSwitchPathController` is fail-closed with no per-item skip tolerance (unlike `getAllCustomers`'s per-item skip-and-log) | `productUpgrades`/`targetList` is a small, tightly-coupled structure describing valid upgrade paths — a malformed entry here likely signals a real upstream/version-drift problem, not a single bad customer record in a large paginated list | A single malformed target in the response fails the whole discovery call rather than degrading gracefully | `models/OfferSwitchPath.ts` `OfferSwitchPathsResponseSchema.parse` (throwing, no try/catch per item) |
| `PREVIEW_SWITCH` returns HTTP 201, matching `PREVIEW`/`PREVIEW_RENEWAL`'s existing (already debatable) convention, even though nothing is persisted upstream | Consistency with sibling preview order types in the same route outweighs correcting a pre-existing REST-semantics inconsistency that is out of scope for this feature | Continues an existing minor inconsistency rather than fixing it | `pages/api/orders.ts` existing `PREVIEW`/`PREVIEW_RENEWAL` branches |
| New `offerSwitchPathController`/`SWITCH` order calls inherit the service-wide "Outbound HTTP resilience" fragile area — no timeout, no retry, no circuit breaker | This feature reuses the exact same `fetch()`-with-no-resilience-config posture every other Partner API connector already has (`SERVICE_CARD.md §4 Fragile Areas`); introducing resilience config for only this feature's calls would be inconsistent with the rest of the codebase | A slow/unavailable `/v3/offer-switch-paths` or switch-order call blocks the request indefinitely, same as every other connector today | Not enforced — inherits `CONNECTORS.md`'s documented "none configured" posture |
| `createOrder`'s existing error-message-shape inconsistency (`JSON.stringify(upstreamErrorBody)` re-parsed by the route) applies unchanged to `SWITCH`/`PREVIEW_SWITCH` | `createOrder` is reused unmodified (Decision 1); fixing this pre-existing inconsistency service-wide is out of scope for this feature | Consumers of a failed switch-order response must handle the same parsed-vs-raw-string ambiguity documented in `SERVICE_CARD.md §4 Fragile Areas` ("Error message shape inconsistency") | `pages/api/orders.ts` existing `catch` block, unchanged |

---

## §6 — Acceptance Criteria Coverage

Source: `feature-specs/anytimeUpgrade/experience-card/anytime-upgrade.md` §Goals (this feature spec has no
separate "Acceptance Criteria" section; Goals is the closest equivalent, as with the sibling early-renewal
card).

| # | Acceptance criterion (from feature spec) | LLD section | Status |
|---|---|---|---|
| 1 | Partners can tell at a glance whether a subscription has an available upgrade, without leaving the Customer Details page. | §3 `pages/api/offer-switch-paths.ts`, `controllers/offerSwitchPathController.ts` | ✅ Covered |
| 2 | Partners cannot open the upgrade flow for a subscription whose currency isn't one the partner is contracted in, or that's missing an offer ID. | — | ⚠ Not covered — this is a UI-only gate (the experience card's `regionCurrencies`/`offerId` check in `ActiveProducts.tsx`); this service has no concept of "partner-contracted currencies" and no backend enforcement point exists or is needed for it |
| 3 | Partners see an accurate price preview — including prorated amount — before committing to the switch order. | §3 `controllers/orderController.ts createPreviewSwitchOrder`, `models/Order.ts` (`proratedDays`/`pricing` already present on `LineItemSchema`) | ✅ Covered |
| 4 | Partners get clear feedback after submitting an upgrade order, **and cannot submit without write permission on the org**. | §3 `pages/api/orders.ts` (error responses); write-permission clause — not covered | ⚠ Partially covered — order submission and its success/error responses are covered; "cannot submit without write permission on the org" is enforced today only in the UI (`hasWritePermissionInOrg`/`canWrite`), consistent with this service's existing fragile area ("No per-request end-user auth" — `SERVICE_CARD.md §4`): no capability in this codebase performs per-caller authorization, and adding one is out of scope for this feature alone |

---

## §7 — Out of Scope

- **`PREVIEW_REVERT_SWITCH` / `REVERT_SWITCH` order types.** Fully documented in the API spec ("Revert Switch
  Order", 14-day window) but the experience card's own "Known Limitations / Not Implemented" section confirms
  no UI surface in this app calls them today. Add them later by following the exact pattern established here
  for `PREVIEW_SWITCH`/`SWITCH` (new schemas in `models/Order.ts`, new controller functions delegating to
  `createOrder`, new `ORDER_API_TYPE` entries, new route branches) once a revert UI surface exists.
- **Offer-switch-path discovery by `market-segment`/`country`/`language`/`offer-id`** (the API spec's
  alternative to `customer-id`+`subscription-id`). The experience card's only real usage is the
  subscription-keyed lookup; `getOfferSwitchPaths` only implements that path. Extending the route to accept
  the broader query combination is straightforward if a future caller (e.g. a catalog-driven upgrade browser
  not tied to an existing subscription) needs it.