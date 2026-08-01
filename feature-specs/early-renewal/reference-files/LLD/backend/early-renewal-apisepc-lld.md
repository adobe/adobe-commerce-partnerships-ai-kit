# LLD — Early Renewal — Backend

## §1 — Summary

Early Renewal lets a partner manually renew a customer's subscriptions before the Anniversary Date
(AD) — either renewing/extending products the customer already owns (Customer Details entry point)
or adding a brand-new product the customer doesn't yet own (Catalog entry point). Three backend
operations are involved, all already implemented as generic renewal capabilities (early and late
renewal share the same order types — there is no separate "early renewal" order type at the API
level):

1. **Preview renewal** — `POST /api/orders?type=PreviewRenewal` → Adobe Partner API `POST /v3/customers/{customerId}/orders` (`orderType: PREVIEW_RENEWAL`), returns per-line pricing/eligibility without committing.
2. **Create renewal order** — `POST /api/orders?type=RenewalOrder` → same upstream endpoint (`orderType: RENEWAL`), places the order and returns `orderId`/`status`.
3. **Get customer subscriptions** — `GET /api/subscriptions?type=getCustomerSubscriptions` → Adobe Partner API `GET /v3/customers/{customerId}/subscriptions`, returns each subscription's `currentQuantity`, `renewedQuantity`, `status`, and `allowedActions`. Required by both entry points to compute eligibility and drive the Existing/Additional segmentation (Customer Details) or the new-products-only / already-in-progress guards (Catalog) described in the feature spec.

All three routes, controllers, and schemas exist today (`pages/api/orders.ts`,
`controllers/orderController.ts`, `models/Order.ts`; `pages/api/subscriptions.ts`,
`controllers/subscriptionController.ts`). The reference app remains stateless — no DB changes. The only
required change is adding `renewedQuantity` to `SubscriptionSchema` (§3) so the field the UI depends
on is explicitly typed rather than relying on `.passthrough()` to leak it through untyped — this one
change covers both entry points, since both read subscriptions through the same route.

---

## §2 — Data Flow

### Operation 1 — Preview renewal (dialog open / "Update price")

```
Trigger: partner opens the Early Renewal Order Dialog, or presses "Update price" inside it

  → POST /api/orders?type=PreviewRenewal&fetch-price=true
      Body: { customerId, currencyCode, lineItems: [{extLineItemNumber, offerId, subscriptionId,
              quantity, currencyCode, flexDiscountCodes?}] }
      (pages/api/orders.ts · handler, ORDER_API_TYPE.PREVIEW_RENEWAL branch — the `type` query param
      is stripped and every other query param, including `fetch-price`, is forwarded as-is via
      `restQuery` into `queryParams` below)
  → controllers/orderController.ts · createPreviewRenewalOrder(customerId, currencyCode,
      accessToken, lineItems, queryParams)
  → body assembled with orderType: 'PREVIEW_RENEWAL', validated via PreviewRenewalOrderSchema.parse
      (ApiError 400 on failure)
  → createOrder(body, accessToken, queryParams)
  → POST {PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders?fetch-price=true
      (queryParams appended verbatim to the upstream URL by createOrder)
      Headers: Authorization: Bearer <token>, Content-Type: application/json,
               X-Correlation-Id (fresh uuid), X-Request-Id (fresh uuid), x-api-key
  ← response: { lineItems[].pricing{partnerPrice, discountedPartnerPrice, netPartnerPrice,
                lineItemPartnerPrice}, lineItems[].proratedDays, pricingSummary[], eligibleOffers[] }
  ← validated via OrderSchema.parse (ApiError 500 on shape mismatch)
  ← route returns: HTTP 201 + Order body (raw, no further transform)

  Note — `fetch-price=true` is load-bearing, not optional: per the API spec (Order-scenarios.md),
  `pricing` and `proratedDays` are only present on the response "when the fetch-price parameter is
  set to true in the request" — omitting it returns eligibility/offer data with no pricing at all.
  Both UI surfaces' reused client helper (`fetchRenewalOrderPreview` in `utils/renewalOrderUtils.ts`)
  always appends this param; this route must keep forwarding whatever query params it's given
  (currently unconditional) rather than only conditionally passing it through.

  Error path: upstream non-2xx → ApiError(JSON.stringify(upstreamBody), status, requestId) →
              route catch re-parses error.message and returns HTTP <upstream status> + the
              upstream error body verbatim (see §5 — no per-code translation).
```

### Operation 2 — Create renewal order (dialog "Renew now")

```
Trigger: partner presses "Renew now" in the Early Renewal Order Dialog

  → POST /api/orders?type=RenewalOrder
      Body: { customerId, externalReferenceId, currencyCode, lineItems: [{extLineItemNumber,
              offerId, subscriptionId, quantity, currencyCode, flexDiscountCodes?}] }
      (pages/api/orders.ts · handler, ORDER_API_TYPE.RENEWAL_ORDER branch)
  → controllers/orderController.ts · createRenewalOrder(customerId, externalReferenceId,
      currencyCode, lineItems, accessToken)
  → body assembled with orderType: 'RENEWAL', validated via RenewalOrderSchema.parse
      (ApiError 400 on failure)
  → createOrder(body, accessToken)
  → POST {PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders
      (same headers as Operation 1)
  ← response: { orderId, status, lineItems[], links }
  ← validated via OrderSchema.parse (ApiError 500 on shape mismatch)
  ← route returns: HTTP 201 + Order body

  Error path: same pass-through pattern as Operation 1 — covers error codes 2164, 3115, 3121,
              3131, 3132, 3133 from the API spec (see §5).
```

### Operation 3 — Get customer subscriptions (dialog open, and Customer Details page render)

```
Trigger: Customer Details page loads (to compute AD-window eligibility and the "renewed" badge
         data), and again when the Early Renewal Order Dialog opens (to filter/segment subscriptions)

  → GET /api/subscriptions?type=getCustomerSubscriptions&customerId=<id>
      (pages/api/subscriptions.ts · handler, SUBSCRIPTION_API_TYPE.GET_CUSTOMER_SUBSCRIPTIONS branch)
  → controllers/subscriptionController.ts · getCustomerSubscriptions(customerId, accessToken)
  → GET {PARTNER_API_BASE_URL}/v3/customers/{customerId}/subscriptions
      Headers: Authorization: Bearer <token>, Accept: application/json, Content-Type: application/json,
               X-Correlation-Id (fresh uuid), X-Request-Id (fresh uuid), x-api-key
  ← response: { items: [{subscriptionId, offerId, currentQuantity, renewedQuantity, status,
                autoRenewal{...}, allowedActions[], renewalDate, currencyCode}] }
  ← each item validated via SubscriptionSchema.parse (ApiError 500 on shape mismatch)
  ← route returns: HTTP 200 + Subscription[] body

  This is a pure read-through — no data transformation beyond Zod validation. `renewedQuantity`
  currently passes through only because of `.passthrough()` on SubscriptionSchema — see §3 for the
  fix that makes it an explicit, typed field.
```

The Anniversary Date (AD) itself is not a subscription field — it is `CustomerDetails.cotermDate`
(`models/CustomerDetails.ts:92`), already required and already returned by the existing
`GET /api/customers?type=getCustomer` route. No change needed there.

### Operation 4 — Add-new-product early renewal (Catalog) reuses Operations 1–2 verbatim

```
Trigger: partner adds a product the customer doesn't already own to the cart, selects that
         customer in the Catalog's "Add customer details" step, chooses "Now (Early Renewal)",
         and completes the dedicated early-renewal checkout

  → same Operation 1 (preview, including the `fetch-price=true` param — see Operation 1's note)
    and Operation 2 (create) calls as above, with one difference: every lineItem's `subscriptionId`
    is omitted (the product is not yet a subscription the customer holds) — no other request field,
    header, or response-handling difference
  → no new backend route, controller function, or schema is introduced for this journey — this is
    made possible entirely by `LineItemSchema.subscriptionId` (`models/Order.ts`) already being
    `z.string().optional()`, and by neither `PreviewRenewalOrderSchema` nor `RenewalOrderSchema`
    requiring it on any line (see §5)
```


---

## §3 — Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `models/Subscription.ts` | MODIFY | Model | `renewedQuantity` is returned by the upstream Partner API (confirmed in the API spec's sample response) and is required by both early-renewal UI surfaces (window-rollover check, Existing/Additional segmentation, quantity clamping, "renewed" badge, and the Catalog's new-products-only / already-in-progress guards), but is not declared on `SubscriptionSchema` — it only reaches consumers today via `.passthrough()`, untyped. |

No other file requires a change: `pages/api/orders.ts`, `controllers/orderController.ts`,
`models/Order.ts` (PREVIEW_RENEWAL/RENEWAL schemas, including `LineItemSchema.subscriptionId`),
`pages/api/subscriptions.ts`, and `controllers/subscriptionController.ts` already implement
everything the feature spec and API spec require for all four operations in §2 — Operations 1–3 as
used from Customer Details, and Operation 4's reuse from the Catalog. Operation 4 in particular
needs no schema change because `LineItemSchema.subscriptionId` (`models/Order.ts`) is already
`z.string().optional()`, and neither `PreviewRenewalOrderSchema` nor `RenewalOrderSchema` requires
it on any line — a `lineItems` array where every entry omits `subscriptionId` (the Catalog's
new-product case) validates and forwards exactly as an existing-subscription line does, with zero
code change. See §5 for this as a documented design decision.

### `models/Subscription.ts` — MODIFY

Current schema (relevant excerpt):

```typescript
export const SubscriptionSchema = z
  .object({
    subscriptionId: z.string().optional(),
    currentQuantity: z.number().optional(),
    usedQuantity: z.number().optional(),
    renewalDate: z.string().optional(),
    creationDate: z.string().optional(),
    status: z.string().optional(),
    allowedActions: z.array(z.string()).optional(),
    links: z.object({ self: z.object({ uri: z.string(), method: z.string(), headers: z.array(z.unknown()) }) }).optional(),
    offerId: z.string().optional(),
    autoRenewal: z.object({
      enabled: z.boolean(),
      renewalQuantity: z.number(),
      renewalCode: z.string().optional(),
      flexDiscountCodes: z.array(z.string()).optional(),
    }),
    currencyCode: z.string().optional(),
  })
  .passthrough();
```

Change: add one field, immediately after `currentQuantity`:

```typescript
    currentQuantity: z.number().optional(),
    renewedQuantity: z.number().optional(),
```

- Type: `z.number().optional()` — matches the existing style for other server-populated,
  not-always-present numeric fields (`currentQuantity`, `usedQuantity`).
- Keep `.passthrough()` on the object as-is (existing convention per `CODE_PATTERNS.md` — "most
  response schemas use `.passthrough()`"); this change is additive and does not remove that
  behavior for any other unmodeled field the upstream API might send.
- No change needed to `CreateSubscriptionRequestSchema` or `UpdateSubscriptionRequestSchema` —
  `renewedQuantity` is a server-computed, response-only field; it is never sent in a create/update
  request body.
- After this change, the exported `Subscription` type (`z.infer<typeof SubscriptionSchema>`)
  includes `renewedQuantity?: number`, so `hooks/` and `components/` consuming this type get it
  without a cast.

---

## §4 — DB Changes

No DB changes — service is stateless (per `DB_SCHEMA.md`: the reference app owns no data store).

---

## §5 — Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|--------------|
| Add `renewedQuantity` as an explicit optional field rather than leaving it to `.passthrough()` | The early-renewal UI's core eligibility/segmentation logic reads this field directly; relying on passthrough means it flows through at runtime but has no compile-time type, so a rename/removal upstream would fail silently instead of at the type level | Slightly wider schema surface vs. staying minimal; gains type safety and self-documentation | `models/Subscription.ts` — Zod schema is the single source of truth for the `Subscription` type |
| No per-error-code translation added for 2164 / 3115 / 3121 / 3131 / 3132 / 3133 | These are business-rule rejections owned by the upstream Partner API (3YC final-term check, EOL/EOS SKU eligibility, in-progress-early-renewal check, etc.) — the reference app is a stateless proxy with no local copy of commitment/SKU state to re-validate against, per `SERVICE_CARD.md` ("owns no data of its own") | Partner-facing error messages are whatever the upstream API sends verbatim, with no app-side friendliness layer — matches existing behavior for every other order type today | `controllers/orderController.ts · createOrder` — `throw new ApiError(JSON.stringify(data), result.status, requestId)`; `pages/api/orders.ts` catch block re-parses and forwards the upstream body as-is |

## §6 — Acceptance Criteria Coverage

| # | Acceptance criterion (from feature spec) | LLD section | Status |
|---|----------------------------------------|-------------|--------|
| 1 | Partners can tell at a glance whether a customer is currently eligible for early renewal, without navigating away from the Customer Details page | §2 Op. 3 (subscriptions), §5 (AD from `cotermDate`) | ✅ Covered — data already available; UI computation is out of scope for backend (§7) |
| 2 | Partners cannot accidentally trigger an early renewal outside the eligible window | §7 | ⚠ Not covered — window enforcement is entirely client-side; Adobe's API additionally rejects out-of-window/ineligible submissions via error codes (§5), but the reference app does not pre-validate |
| 3 | Partners can distinguish, within the dialog, between renewing a customer's existing licenses and adding additional seats beyond current quantity | §3 (`renewedQuantity` + `currentQuantity` on `SubscriptionSchema`) | ✅ Covered |
| 4 | Partners can add a new product to a customer's early renewal from the Catalog, without needing to already know new products aren't supported from the Customer Details dialog | §2 Op. 4 (reuses Op. 1–2 verbatim, `subscriptionId` omitted) | ✅ Covered — no backend change needed; `LineItemSchema.subscriptionId` already optional (§5) |
| 5 | Partners cannot create an ambiguous order that mixes new products with products the customer already owns | §7 | ⚠ Not covered — this guard (comparing cart offer IDs against the customer's existing subscriptions) is entirely client-side; the upstream API's `subscriptionId`/`offerId` collision rejection (error `3121`) is the actual backstop if a mixed order is somehow submitted (§5) |
| 6 | Partners see accurate, up-to-date pricing (per-license price, discount level, prorated days) before confirming the order, on either path | §2 Op. 1 (`PreviewRenewalOrderSchema` / `LineItemSchema.pricing` / `proratedDays` / `fetch-price=true`) | ✅ Covered |
| 7 | Partners get clear feedback — success or failure — after submitting, with the customer record reflecting the renewed quantity immediately after | §2 Op. 2 (order placement) | ⚠ Partially covered — order placement and its response are fully implemented; there is no backend push/invalidation mechanism, so "immediately after" depends entirely on the UI re-fetching subscriptions (flagged as a gap in the feature spec itself) |
| 8 | Error codes 2164 / 3115 / 3121 / 3131 / 3132 / 3133 surfaced to the partner | §2 (error paths), §5 | ✅ Covered — generic pass-through already forwards the upstream error body verbatim for both operations, on both entry points |

---