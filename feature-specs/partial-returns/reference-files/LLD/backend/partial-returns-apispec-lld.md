# LLD — Partial Return (Backend)

**Feature spec:** `feature-specs/partial-returns/experience-card/partial-return.md`
**API spec:** `feature-specs/partial-returns/APISpec/partial-returns-apispec.md`

---

## §1 Summary

Partial Return lets a partner return **part of** the quantity on a past order's line item — not
just the full line — and submit multiple partial returns against the same line item over time, as
long as the order is within its 14-day return window. This capability introduces **no new API
endpoint or route**. Both operations it depends on already exist in this service exactly as
needed:

- **Submitting the return** reuses the existing `POST /api/orders?type=Return` capability
  (`orderController::createReturnOrder`) unchanged — "partial" vs. "full" is only a difference in
  the `quantity` value sent per line item, and that field has never been constrained to the line
  item's original quantity.
- **Seeing the remaining returnable quantity** reuses the existing `GET
  /api/orders?type=getOrdersHistory` capability unchanged in behavior, but its response schema needs the upstream's new `remainingQuantity`
  per-line-item field added explicitly.

The only backend file this LLD touches is `models/Order.ts` (add `remainingQuantity` to
`LineItemSchema`). Every other piece of this feature's backend behavior — accepting a partial
`quantity`, relaying the upstream's new HTTP 422 status, and relaying the return-specific
error codes already works correctly today,

---

## §2 Data Flow

### Operation A — Submit a return (full or partial)

```
Operation: partner submits the Return Order Dialog with one or more line items marked
           "Return" and a quantity for each (1 ≤ quantity ≤ that line's remaining returnable
           quantity)

  → POST /api/orders?type=Return
  → pages/api/orders.ts handler → orderController::createReturnOrder
       validates body against ReturnOrderSchema (unchanged — quantity per line item is an
       unconstrained z.number(), already accepts any value up to the caller's own line-item max)
  → orderController::createOrder → POST {PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders
       body: { customerId, orderType: "RETURN", referenceOrderId, externalReferenceId,
               currencyCode, lineItems: [{ extLineItemNumber, offerId, quantity, currencyCode? }] }
  ← response (success): order object matching OrderSchema — HTTP 201
  ← response (rejected): HTTP 422, { code, message, additionalDetails } — per the experience
       card's Error Handling section, the top-level code/message are generic (e.g. code: "2120",
       message: "Line item quantity out of range" — this is the shape a plain quantity-exceeds-
       remaining rejection takes, since the API spec no longer lists a dedicated code for it); the
       specific reason, when one of the three enumerated scenarios applies, is carried as a
       "Reason: <CODE>" entry inside additionalDetails, where <CODE> is one of:
         PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE (MOQ offer)
         SWITCH_ORDER_CANCELLATION_NOT_ALLOWED (referenced order is a SWITCH order)
         THREE_YEAR_COMMIT (would breach a 3YC product family's minimum commit quantity)
  ← handler maps to:
       success → OrderSchema.parse(data) → BackendResult<Order>
       failure → ApiError(JSON.stringify(data), result.status, requestId)  (orderController.ts,
                 unchanged — see CONNECTORS.md #1 "Error handling")
  ← route returns:
       success → HTTP 201 + Order body
       failure → HTTP <upstream status, e.g. 422> + the upstream's exact
                 { code, message, additionalDetails } body, via pages/api/orders.ts's existing
                 JSON.parse(error.message) re-forward
                 (CONTRACTS.md → "Errors (all /api/orders operations)")
```

No code change is required to make this operation correct for partial quantities or for the new
422/error-code contract — it is purely read-through/pass-through at every layer already.

### Operation B — Load Purchase History (remaining returnable quantity)

```
Operation: Customer Details page → Purchase History tab renders, or the Return Order Dialog's
           parent list is refetched after a return succeeds

  → GET /api/orders?type=getOrdersHistory&customerId=...&offset=...&limit=...
  → pages/api/orders.ts handler → orderController::getOrdersHistoryForCustomer
  → GET {PARTNER_API_BASE_URL}/v3/customers/{customerId}/orders?offset&limit&fetch-price=true
  ← response: { totalCount, count, offset, limit, items: [{ ..., lineItems: [{ extLineItemNumber,
       offerId, quantity, remainingQuantity?, status, ... }] }] }
  ← handler maps to: OrdersHistoryResponseSchema.parse(data) + computed hasMore
       — after this LLD's change, remainingQuantity is a typed optional field on every line item,
         not just an unvalidated passthrough value
  ← route returns: HTTP 200 + OrdersHistoryResponseWithPagination body
```

Purely read-through — no transformation of `remainingQuantity` happens in this service; it is
validated (now explicitly typed) and passed straight to the client, which computes "remaining of
original" display text and the return-quantity input's max bound itself.
---

## §3 Change Summary

Both API-spec operations reuse existing, unchanged routes (`CONTRACTS.md`'s `POST /api/orders —
Return/cancel an order — type=Return` and `GET /api/orders — Fetch order history for a customer —
type=getOrdersHistory` entries need no update — their documented shapes were already generic
enough to cover this). One shared model file needs a field added.

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `models/Order.ts` | MODIFY | Model | Add `remainingQuantity` to the shared `LineItemSchema` so the upstream's per-line-item remaining-returnable-quantity value is explicitly typed (not just passthrough-preserved) everywhere line items appear in a response |

### `models/Order.ts` — MODIFY

Current implementation (read in full before this change): `LineItemSchema` (lines 26–49) is a
`.passthrough()` Zod object used by every order request schema (`NewOrderSchema`,
`ReturnOrderSchema`, `RenewalOrderSchema`, `PreviewOrderSchema`, `PreviewRenewalOrderSchema`) and
every order response schema (`OrderSchema`, `OrdersHistoryOrderSchema`) via `lineItems:
z.array(LineItemSchema)`.

- Add one new optional field, `remainingQuantity: z.number().optional()`, placed immediately after
  `quantity` for readability (it's semantically "what's left of `quantity` after prior returns").
- **Why optional, not required:** per the feature/API spec, this field is only present on a line
  item that has had at least one return (partial or full) processed against it — a never-returned
  line item omits it entirely, and the caller must fall back to the line item's own `quantity` as
  the fully-returnable amount (this fallback logic itself is UI-side, out of scope here — see §7).
- **Why one edit propagates everywhere:** `LineItemSchema` is the single shared definition for
  every order request/response schema in this file (`CODE_PATTERNS.md §1`: "Both request and
  response shapes for a resource live in the same file"). No other schema, controller, or route
  needs a matching change.
- **No request-side constraint added:** `quantity` on a `ReturnOrderSchema` line item remains
  `z.number()` with no min/max tied to the original or remaining quantity — this was already true
  before this feature and already permits partial-quantity return requests to pass validation.
  Do not add an upper-bound check here (see §5, Design Decision row 2).

```ts
// models/Order.ts — LineItemSchema (excerpt, new line marked)
export const LineItemSchema = z
  .object({
    extLineItemNumber: z.number(),
    offerId: z.string(),
    quantity: z.number(),
    remainingQuantity: z.number().optional(), // NEW — remaining returnable quantity after prior return(s); present only once a return has been processed against this line item
    subscriptionId: z.string().optional(),
    status: z.string().optional(),
    currencyCode: z.string().optional(),
    deploymentId: z.string().optional(),
    discountCode: z.string().optional(),
    proratedDays: z.number().optional(),
    pricing: PricingSchema.optional(),
    flexDiscountCodes: z.array(z.string()).optional(),
    flexDiscounts: z
      .array(
        z.object({
          id: z.string(),
          code: z.string(),
          result: z.string(),
        })
      )
      .optional(),
  })
  .passthrough();
```

**Downstream type impact (no further file changes needed):** `LineItem` (`z.infer<typeof
LineItemSchema>`) gains `remainingQuantity?: number`, which flows into `Order` and
`OrdersHistoryOrder` (both embed `LineItem[]`) automatically via existing type inference —
`MODULE_INDEX.md`'s Type Inventory entry for `LineItem` already documents it as "embedded in all
Order request/response schemas," which remains accurate after this change.

**Controllers, routes, and connectors — explicitly unchanged and why:**

| Component | Why no change is needed |
|---|---|
| `orderController::createReturnOrder` / `ReturnOrderSchema` | `quantity` per line item was already an unconstrained `z.number()` — partial-quantity requests already validate and forward correctly |
| `orderController::createOrder` (shared POST) | Already stringifies and forwards any non-OK upstream response verbatim as `ApiError(JSON.stringify(data), result.status, requestId)` (`orderController.ts:150`) — the new HTTP 422 status and the `{code, message, additionalDetails}` error bodies (including the `"Reason: <CODE>"` entries) pass through with no code change |
| `pages/api/orders.ts` (both GET/POST handlers) | Already `JSON.parse(error.message)`s and forwards the backend error body verbatim at the upstream's own status code (`CONTRACTS.md → "Errors (all /api/orders operations)"`) |
| `orderController::getOrdersHistoryForCustomer` / `OrdersHistoryResponseSchema` | Already validates and returns the full `lineItems` array per order — the new field rides along once `LineItemSchema` is updated |
| Adobe Commerce Partner API connector (`CONNECTORS.md #1`) | No new auth, transport, retry, or timeout requirement — same base URL, same Bearer + `x-api-key` pattern, same fail-fast posture |

---

## §4 DB Changes

No DB changes — service is stateless (`DB_SCHEMA.md`, `PLATFORM.md §5`). All order/return state is
owned by the upstream Adobe Commerce Partner API.

---

## §5 Design Decisions

| Decision | Why | Trade-off | Enforcement |
|---|---|---|---|
| Add `remainingQuantity` to `LineItemSchema` explicitly rather than relying on `.passthrough()` alone | The field is now load-bearing for return-eligibility logic (not an incidental unknown upstream field), so implicit passthrough typing understates its importance and leaves it undiscoverable to anyone reading the schema (`CODE_PATTERNS.md §2`: passthrough is for fields the app "doesn't fully control," but every field the app actually *reads* is typed elsewhere in this same schema) | One more field to keep in sync with the upstream contract if it's ever renamed | Matches existing convention of typing every field the app reads; not enforced by tooling — same review discipline as every other schema field |
| No backend-side pre-validation that `quantity ≤ remainingQuantity` before forwarding to upstream | The upstream already enforces this and rejects with HTTP 422 (a generic `{code: "2120", message: "Line item quantity out of range"}`-shaped body per the experience card — the API spec no longer lists a dedicated "exceeds remaining" code, only the three MOQ/Switch/3YC scenarios) that this service's existing fail-closed, verbatim-forwarding error path already surfaces correctly (`CONNECTORS.md #1` "Error handling"; `MODULE_INDEX.md → Create New/Return/Preview/Preview-Renewal/Renewal Order → Key Decisions`) | An invalid request makes a wasted round-trip to the upstream instead of failing fast at the edge — consistent with every other order-write capability in this service, which validate shape only, never business rules (`CODE_PATTERNS.md §4`) | Not enforced — relies on the existing "shape-only validation, defer business rules to upstream" convention |
| Reuse the existing `POST /api/orders?type=Return` operation unchanged rather than adding a `type=PartialReturn` operation | The API spec confirms partial and full returns share the identical request shape — quantity is just a value, not a distinct operation type | None identified — this is a pure simplification versus adding a redundant, functionally-identical route | `CONTRACTS.md`'s existing `type=Return` entry already documents this generically; no `CONTRACTS.md` update needed |

---

## §6 Acceptance Criteria Coverage

| # | Acceptance criterion (from feature spec) | LLD section | Status |
|---|---|---|---|
| 1 | "Partners can tell at a glance which past orders are still eligible for a return, without leaving the Customer Details page." | — | ⚠ Not covered — UI-only (Purchase History table gating). No backend change needed: `orderType`, `status`, and `creationDate` are already returned unmodified by the existing `getOrdersHistory` capability. |
| 2 | "Partners cannot attempt a return on an order that is outside the return window, already fully processed as a return/switch, or not in a returnable status." | — | ⚠ Not covered — UI-only gating (button visibility). The backend does not need to replicate this check; if bypassed, the upstream itself would reject an ineligible order via its own validation. |
| 3 | "Partners can return **fewer** licenses than a line item's full quantity, and see exactly how many licenses remain returnable before doing so." | §3 `models/Order.ts` | ✅ Covered — "return fewer" required no change (quantity was already unconstrained); "see exactly how many remain" is covered by the new typed `remainingQuantity` field. |
| 4 | "Partners cannot enter a return quantity that exceeds what is actually still returnable on a line item." | §2 Operation A; §5 row 2 | ✅ Covered (backend safety-net half only) — upstream enforces via a generic HTTP 422 rejection (no dedicated code for this scenario in the current API spec), relayed verbatim by this service's existing error-forwarding. The "cannot enter" UI input constraint itself is out of scope for this backend LLD. |
| 5 | "Partners get clear feedback — success or failure — after submitting, and the Purchase History reflects the new return order once it's created." | §2 Operations A & B | ✅ Covered (backend half) — success/error response bodies pass through unchanged; Purchase History refetch already returns the new `RETURN` order and updated `remainingQuantity` once §3's schema change ships. Toast construction itself is UI, out of scope. |
| 6 | "A known return error code gets a partner-facing message; anything else falls back to the backend's generic message." | §2 Operation A | ⚠ Not covered — entirely a UI-side code→message lookup table (over `PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE` / `SWITCH_ORDER_CANCELLATION_NOT_ALLOWED` / `THREE_YEAR_COMMIT`). The backend's only responsibility is relaying the upstream `{code, message, additionalDetails}` body unchanged, which it already does. |

---