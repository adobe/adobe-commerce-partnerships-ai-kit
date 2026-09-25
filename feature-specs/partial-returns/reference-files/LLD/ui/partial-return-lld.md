


# LLD — Partial Return (UI)

**Experience card:** `feature-specs/partial-returns/experience-card/partial-return.md`
**Backend LLD:** `feature-specs/partial-returns/reference-files/LLD/backend/partial-returns-apispec-lld.md`
**Target repo:** `adobe-commerce-partnerships-ref-app`
**UI service cards read:** `UI_SERVICE_CARD.md`, `UI_MODULE_INDEX.md`, `ROUTES.md`, `DATA_LAYER.md`, `STATE_MANAGEMENT.md`, `UI_CODE_PATTERNS.md`, `UI_PLATFORM.md`
**Source verified directly:** `components/customerdetails/ReturnOrderDialog.tsx`, `components/customerdetails/PurchaseHistoryPanel.tsx`, `utils/returnOrderUtils.ts`, `hooks/useOrderOperations.ts`, `models/Order.ts`, `types/order.ts`, `utils/customerDetailsUtils.ts`, `styles/customerdetails/ReturnOrderDialog.module.css`

---

## §1 Summary

**No Figma URL — LLD derived from experience card description only** (the card's `figma-url:` field is blank). All layout specs below are derived from the card's text description and the existing `ReturnOrderDialog.tsx` implementation; they are approximate, not pixel-exact.

Partial Return upgrades the existing **Return Order Dialog** (`components/customerdetails/ReturnOrderDialog.tsx`, opened from `components/customerdetails/PurchaseHistoryPanel.tsx`) from a full-quantity-only return to a per-line-item, partial-quantity return. This app already has a working, shipped "Return an Order" capability (`UI_MODULE_INDEX.md → View Purchase History & Return an Order`) — this feature is a **modification of that existing dialog**, not a new capability:

- **Surface 1 (Return entry point)** — `PurchaseHistoryPanel.tsx`'s "Return items" button visibility logic (status `1000`, order type not `RETURN`/`SWITCH`/`REVERT_SWITCH`, within the 14-day return window via `isOrderWithinReturnWindow`) already matches the experience card's gating rules **exactly, verbatim**. **No change needed to this file.**
- **Surface 2 (Return Order Dialog)** — needs real changes: it currently shows every status-`1000` line item with its full quantity as a read-only number, and always submits the full `item.quantity` on return. It must instead: filter out lines with zero remaining returnable quantity, show "`{remaining} of {original}` licenses returnable" per line, add a **Quantity to return** input (bounded `1..remaining`) that only appears when the line's **Return** toggle is on, submit each line's *entered* quantity (not the full quantity), and show "No line items available for return" when every line is already fully returned.
- **Error handling** — the dialog already has a code → partner-facing-message lookup (`utils/returnOrderUtils.ts::buildReturnErrorMessage`, mirroring the order-preview error pattern) covering `SWITCH_ORDER_CANCELLATION_NOT_ALLOWED` and `THREE_YEAR_COMMIT`, but is **missing** the third code the card requires: `PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE`.

No new route, no new data fetch unit, no new state store. The mutation (`useReturnOrder`, unit #14 in `DATA_LAYER.md`) already sends whatever `quantity` value is in the payload and already parses `code`/`additionalDetails` off an error response — it needs no change. The one dependency outside this LLD's control: `models/Order.ts`'s `LineItemSchema` must gain an optional `remainingQuantity: z.number().optional()` field for `item.remainingQuantity` to type-check in the dialog — this is already specified in the **backend LLD** (§3, `models/Order.ts` — MODIFY) and is called out here only as a cross-reference, not re-specified.

---

## §2 Data Flow

### Flow A — Return entry point visibility (unchanged, existing behavior)

```
User action: partner opens the Customer Details page → Purchase History tab
  → PurchaseHistoryPanel renders each order row
  → per row: order.status === '1000' && order.orderType not in ['RETURN','SWITCH','REVERT_SWITCH']
             && isOrderWithinReturnWindow(order.creationDate)  (RETURN_WINDOW_DAYS = 14)
  ← component renders: "Return items" button shown only when all three hold — no disabled/tooltip
    state, matches the experience card's Surface 1 rules exactly, verified against current source.
    NO CODE CHANGE — already correct.
```

### Flow B — Opening the Return Order Dialog and computing eligible line items

```
User action: partner clicks "Return items" on an eligible order row
  → PurchaseHistoryPanel sets selectedReturnOrder = order, isReturnDialogOpen = true
  → ReturnOrderDialog mounts/re-opens with orderData = order
  → on-open effect resets: selectedItems = {}, returnQuantities = {}, returnOrder.reset(),
    clearError(), clearSuccess()
  → eligibleLineItems = orderData.lineItems.filter(item =>
        item.status === '1000' && getRemainingReturnableQuantity(item) > 0)
      where getRemainingReturnableQuantity(item) = item.remainingQuantity ?? item.quantity
  → subscriptionContexts built from eligibleLineItems only (offerId + currencyCode + region)
  → useProductNamesFromPricelist(subscriptionContexts) fires (existing fetch unit #11,
    fetchProductPricing, unchanged) to resolve product names/icons
  ← component renders:
      - eligibleLineItems.length === 0 → "No line items available for return" (no list)
      - else → one Card per eligible line item, each showing:
          product icon + name, "{remaining} of {original} licenses returnable",
          Return toggle, offer ID, and (conditionally) the quantity control — see Flow C
```

### Flow C — Toggling Return on/off and entering a quantity

```
User action: partner flips a line item's "Return" toggle on
  → toggleItem(extLineItemNumber) adds it to selectedItems
  → returnQuantities[extLineItemNumber] initialized to 1 (the minimum)
  ← component renders: a NumberField "Quantity to return (max {remaining})" for that line,
    minValue=1, maxValue=remaining, value=returnQuantities[extLineItemNumber]

User action: partner adjusts the Quantity to return field
  → NumberField's own minValue/maxValue clamp the value (Spectrum's built-in behavior —
    same pattern as EditRenewalDialog/CheckoutProductCard/LateRenewalOrderDialog)
  → onChange updates returnQuantities[extLineItemNumber]

User action: partner flips the same toggle off
  → toggleItem removes it from selectedItems and deletes its entry from returnQuantities
  ← component renders: the line item's full original quantity as plain text (not editable),
    replacing the quantity field — the offer ID remains visible either way
```

### Flow D — Submitting the return

```
User action: partner clicks "Return" (enabled only when: not submitting, ≥1 item toggled on,
             and the mutation hasn't already succeeded)
  → ReturnOrderDialog::handleReturn builds lineItems from eligibleLineItems filtered to
    selectedItems.has(extLineItemNumber), mapping each to:
      { extLineItemNumber, offerId, quantity: returnQuantities[extLineItemNumber] ?? 1,
        ...(item.currencyCode ? { currencyCode: item.currencyCode } : {}) }
  → useReturnOrder().mutate({ customerId, referenceOrderId: orderData.orderId ||
      orderData.referenceOrderId, externalReferenceId: generateExternalReferenceId(),
      currencyCode: orderData.currencyCode, lineItems })
  → POST /api/orders?type=Return   (unchanged — see backend LLD §2 Operation A)
  ← response (success, HTTP 201): Order body
  ← response (failure, HTTP 422): { code, message, additionalDetails }
  ← onSuccess: showSuccess('Return submitted'); button reads "Returned!"; after 1500ms:
      onClose() + queryClient.invalidateQueries({ queryKey: ['ordersHistory', customerId] })
      → PurchaseHistoryPanel refetches, showing the new RETURN order and the originating
        order's updated remainingQuantity once the backend reflects it
  ← onError: showError(buildReturnErrorMessage(error))
      → scans error.additionalDetails for a "Reason: <CODE>" entry matching one of
        PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE / SWITCH_ORDER_CANCELLATION_NOT_ALLOWED
        / THREE_YEAR_COMMIT → partner-facing message; else falls back to error.message
        (+ additionalDetails appended), shown verbatim
  ← component renders: ErrorToast with the resolved message; original order/line items unchanged
    (remaining-quantity figures only update after Flow B re-runs on the next successful refetch)
```

---

## §3 Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `components/customerdetails/ReturnOrderDialog.tsx` | MODIFY | Component | Add remaining-returnable-quantity display, per-line quantity-to-return input, empty state for fully-returned orders, and submit the entered quantity instead of the full quantity |
| `utils/returnOrderUtils.ts` | MODIFY | Util | Add the missing `PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE` error code mapping; add a `getRemainingReturnableQuantity` helper |
| `styles/customerdetails/ReturnOrderDialog.module.css` | MODIFY | Style | Add classes for the "remaining of original" label line and the quantity-field wrapper |

No new route, data fetch unit, or state store. `components/customerdetails/PurchaseHistoryPanel.tsx` needs **no change** — verified against the card's Surface 1 rules and left out of the table per the "don't list unchanged files" rule.

### `components/customerdetails/ReturnOrderDialog.tsx` — MODIFY

**Current implementation (read in full before this change):** a functional component that tracks only `selectedItems: Set<number>` (keyed by `extLineItemNumber`); renders every `status === '1000'` line item with its full `item.quantity` as plain text and a `Return` `Switch`; on submit, maps every selected line to `{ extLineItemNumber, offerId, quantity: item.quantity, currencyCode? }` — i.e. it can only ever return a line's **full** quantity today.

**State variables:**

| Name | Type | Default | Triggers re-fetch? |
|---|---|---|---|
| `selectedItems` | `Set<number>` (extLineItemNumber) | `new Set()` | No — local only |
| `returnQuantities` | `Record<number, number>` (extLineItemNumber → quantity) — **NEW** | `{}` | No — local only |

Both are reset (along with `returnOrder.reset()`, `clearError()`, `clearSuccess()`) in the existing on-open `useEffect` keyed on `[isOpen, orderData, clearError, clearSuccess]` — add `setReturnQuantities({})` alongside the existing `setSelectedItems(new Set())`.

**Derived values:**

| Name | Formula |
|---|---|
| `eligibleLineItems` | `orderData?.lineItems.filter(item => item.status === '1000' && getRemainingReturnableQuantity(item) > 0) ?? []` (uses the new util from `returnOrderUtils.ts`) |
| `subscriptionContexts` | rebuilt from `eligibleLineItems` instead of the current unfiltered `orderData.lineItems` — avoids fetching product names/pricing for line items that will never render |
| per-line `remaining` | `getRemainingReturnableQuantity(item)` |
| per-line `isSelected` | `selectedItems.has(item.extLineItemNumber)` |

**Interaction logic:**

- `toggleItem(extLineItemNumber, remaining)`:
  - toggles membership in `selectedItems` (unchanged logic)
  - if turning **on**: `setReturnQuantities(prev => ({ ...prev, [extLineItemNumber]: 1 }))`
  - if turning **off**: `setReturnQuantities(prev => { const next = { ...prev }; delete next[extLineItemNumber]; return next; })`
- Quantity field `onChange`: `value => setReturnQuantities(prev => ({ ...prev, [item.extLineItemNumber]: value ?? 1 }))` — no manual clamp function; rely on the `NumberField`'s own `minValue`/`maxValue` enforcement, matching `EditRenewalDialog`/`CheckoutProductCard`/`LateRenewalOrderDialog`'s existing convention (`UI_CODE_PATTERNS.md` has no separate quantity-clamp utility — every existing quantity input trusts Spectrum's own bounds).
- `handleReturn`: change the `lineItems` map from `quantity: item.quantity` to `quantity: returnQuantities[item.extLineItemNumber] ?? 1`; keep the existing `.filter(item => selectedItems.has(item.extLineItemNumber) && item.status === '1000')` filter, but source it from `eligibleLineItems` instead of `orderData.lineItems` (equivalent for status, additionally excludes zero-remaining lines defensively).

**Visual spec (approximate — no Figma reference):**

- Per-line `Card` (`UNSAFE_className={styles.productCard}`, unchanged container):
  - **Row 1** (`cardHeader`, unchanged): product icon (`getIconWithFallback`) + name on the left; `Switch` "Return" on the right, `isDisabled={isSubmitting}` (unchanged)
  - **Row 2 — NEW**: `<div className={styles.remainingLine}>{remaining} of {item.quantity} licenses returnable</div>`
  - **Row 3** (`productInfo` / `leftCol`, modified):
    - if `isSelected`: `NumberField` — `id={`return-quantity-${item.extLineItemNumber}`}`, `label={`Quantity to return (max ${remaining})`}`, `value={returnQuantities[item.extLineItemNumber] ?? 1}`, `minValue={1}`, `maxValue={remaining}`, `isDisabled={isSubmitting}`, `size="S"` — matches the `NumberField` prop shape already used in `EditRenewalDialog.tsx:366-374`
    - else: `<span className={styles.quantity}>{item.quantity} licenses</span>` (existing plain-text display, unchanged)
    - `<span className={styles.offerId}>{item.offerId}</span>` directly below either the field or the plain text (unchanged element, same position)
- Empty state — **NEW branch**, sibling to the existing `orderData ? (...) : <"No order data available">` check:
  ```tsx
  {orderData ? (
    eligibleLineItems.length === 0 ? (
      <div className={styles.noDataContainer}>
        <p>No line items available for return</p>
      </div>
    ) : (
      <div className={styles.productList}>
        {eligibleLineItems.map((item, index) => { /* per-line card, as above */ })}
      </div>
    )
  ) : (
    <div className={styles.noDataContainer}>
      <p>No order data available</p>
    </div>
  )}
  ```

**Loading / error / empty states:**

- Loading: unchanged — `Switch`/`NumberField`/`Button` all gate on the existing `isSubmitting` (`returnOrder.isPending`).
- Error: unchanged mechanism (`ErrorToast` via `useToastState`), but now resolves through `buildReturnErrorMessage` after its `utils/returnOrderUtils.ts` update (see below) — no call-site change needed in this file.
- Empty: **NEW** — "No line items available for return" when `eligibleLineItems.length === 0` (see above). Distinct from the pre-existing "No order data available" (`orderData` itself is `null`).

**Accessibility:** unchanged — dialog `role="dialog"`/`aria-modal="true"`/`aria-labelledby` and the `Escape`/click-outside dismiss handlers are untouched by this change. Add nothing new here; `NumberField`'s own label prop (`Quantity to return (max N)`) provides its accessible name, consistent with every other `NumberField` usage in this codebase (none of which add a separate `aria-label` when a visible `label` prop is present, e.g. `CartDetails.tsx:214`).

---

### `utils/returnOrderUtils.ts` — MODIFY

**Current implementation (read in full before this change):**

```ts
export const RETURN_ERROR_MESSAGES: Record<string, string> = {
  SWITCH_ORDER_CANCELLATION_NOT_ALLOWED:
    'The referenced order is a SWITCH order — REVERT_SWITCH must be used instead.',
  THREE_YEAR_COMMIT:
    "The return would reduce a Three-Year Commit product family's quantity below its minimum commit quantity.",
};
```
`buildReturnErrorMessage` (unchanged logic — scans `additionalDetails` for a substring match against `RETURN_ERROR_MESSAGES`' keys, falls back to `message` + joined `additionalDetails`).

**Change 1 — add the missing third error code**, per the experience card's Error Handling table:

```ts
export const RETURN_ERROR_MESSAGES: Record<string, string> = {
  PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE:
    'The offer is flagged as a Minimum Order Quantity (MOQ) offer.',
  SWITCH_ORDER_CANCELLATION_NOT_ALLOWED:
    'The referenced order is a SWITCH order — REVERT_SWITCH must be used instead.',
  THREE_YEAR_COMMIT:
    "The return would reduce a Three-Year Commit product family's quantity below its minimum commit quantity.",
};
```
`buildReturnErrorMessage` itself needs no logic change — it already iterates `Object.keys(RETURN_ERROR_MESSAGES)` generically.

**Change 2 — add the remaining-returnable-quantity helper**, used by `ReturnOrderDialog.tsx` (Flow B/C above):

```ts
import type { LineItem } from '../models/Order';

/**
 * A line item's remaining returnable quantity: `remainingQuantity` if the upstream has set it
 * (i.e. at least one return has already been processed against this line), otherwise the line's
 * full `quantity` (never returned yet — fully returnable).
 */
export function getRemainingReturnableQuantity(item: LineItem): number {
  return item.remainingQuantity ?? item.quantity;
}
```

This depends on `LineItem` (`z.infer<typeof LineItemSchema>`) having an optional `remainingQuantity: number` field — see the `models/Order.ts` cross-reference below.

---

### `styles/customerdetails/ReturnOrderDialog.module.css` — MODIFY

**Current implementation:** has `.productCard`, `.cardHeader`, `.productHeader`, `.productIcon`, `.productName`, `.productInfo`, `.leftCol`, `.quantity`, `.offerId`, `.noDataContainer` (see file for exact values — unchanged, reused as-is).

**New classes needed** — no Figma reference, so no pixel/color values are specified here; match the sizing and color tokens already used by the sibling `.quantity`/`.offerId` classes in this same file:

- `.remainingLine` — the "`{remaining} of {original}` licenses returnable" label line
- `.quantityFieldWrapper` — layout wrapper around the "Quantity to return" `NumberField`

No existing class is removed or renamed; `.quantity` and `.offerId` are reused unchanged for the toggle-off plain-text state.

---

## §4 Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| Keep `returnQuantities` as local `useState` in `ReturnOrderDialog`, not lifted to `CartContext`/a new store | The value is scoped entirely to one dialog's open lifetime and is reset every time the dialog opens (`STATE_MANAGEMENT.md`'s only two stores are `CartContext`/`PartnerContext`, neither of which models per-return-line quantities) | None identified — matches `UI_MODULE_INDEX.md`'s "Do not add a global state library" rule | Not enforced by tooling — same convention as `selectedItems`, which is already local state in this component |
| Trust `NumberField`'s built-in `minValue`/`maxValue` clamping instead of writing a manual clamp function | Every other quantity input in this codebase (`EditRenewalDialog`, `CheckoutProductCard`, `LateRenewalOrderDialog`, `CartDetails`) does the same — introducing a manual clamp here would be a second, inconsistent pattern | A malformed/rapid input could theoretically flash an out-of-range value for one render before Spectrum corrects it — accepted, since no existing quantity input in this app guards against that either | Not enforced — matches existing convention exactly |


---

## §5 Acceptance Criteria Coverage

| Goal (from experience card) | Covered by |
|-----------------------------|-----------|
| "Partners can tell at a glance which past orders are still eligible for a return, without leaving the Customer Details page." | `components/customerdetails/PurchaseHistoryPanel.tsx` — **pre-existing, verified unchanged**; not in §3 because no code change is needed |
| "Partners cannot attempt a return on an order that is outside the return window, already fully processed as a return/switch, or not in a returnable status." | `components/customerdetails/PurchaseHistoryPanel.tsx` — **pre-existing, verified unchanged** (same as above) |
| "Partners can return **fewer** licenses than a line item's full quantity, and see exactly how many licenses remain returnable before doing so." | `components/customerdetails/ReturnOrderDialog.tsx` (§3) — quantity field bounded `1..remaining`; "`{remaining} of {original}`" label line |
| "Partners cannot enter a return quantity that exceeds what is actually still returnable on a line item." | `components/customerdetails/ReturnOrderDialog.tsx` (§3) — `NumberField maxValue={remaining}` |
| "Partners get clear feedback — success or failure — after submitting, and the Purchase History reflects the new return order once it's created." | `components/customerdetails/ReturnOrderDialog.tsx` (§3, Flow D) — unchanged toast/refetch mechanism, now driven by per-line entered quantities |
| Dialog loading: reset selections/quantities, filter to status `1000` + remaining > 0, "No line items available for return" when none remain | `components/customerdetails/ReturnOrderDialog.tsx` (§3, Flow B) |

All acceptance criteria from the experience card are covered — none flagged.

---