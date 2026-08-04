# LLD — Anytime Upgrade (UI)

**No Figma URL** — the experience card's `figma-url:` field is empty. This LLD is derived entirely from the
experience card description; all visual specs (spacing, colors, exact layout) are approximate/TBD and should
be refined against Bridge's existing Spectrum-based screens (`ActiveProducts.tsx`, `checkout.tsx`) during
implementation.

---

## §1 — Summary

Anytime Upgrade lets a partner move a customer's active subscription to a higher-tier product before its
renewal date. Three UI surfaces are affected, all reached from the existing Customer Details page:

1. **Upgrade Entry Point** — an "Upgrade" action added to each subscription card in `ActiveProducts.tsx`
   (Active Products list), shown only when a per-subscription discovery call finds upgrade targets and the
   subscription has an `offerId`.
2. **Upgrade Paths Dialog** (new) — a radio list of target offers, reusing the already-fetched discovery data
   from Surface 1; resolves target product names via a live pricelist lookup.
3. **Upgrade Order Summary** (new page) — a checkout-style screen driven entirely by URL query params
   (no cart, no app context) that auto-fires a price preview, allows quantity edit / promo code entry
   depending on switch type, and places the switch order. Per the experience card's Surface 3 screen layout
   (now grounded in an actual product screenshot), this screen is visually and structurally a direct sibling of
   `pages/checkout.tsx` — same breadcrumb, same left "review details" card / right "order summary" card split,
   same terms-line and Estimated Total copy — with upgrade-specific content (source→target transition row,
   per-license proration caption, Offer ID) substituted in. See §4 for the resulting component-split decision.

This feature depends on the backend LLD's new `GET /api/offer-switch-paths` route and the `PreviewSwitch` /
`Switch` extensions to `POST /api/orders` — those backend changes (including `models/Order.ts` and
`utils/constants.ts` updates) are **not** re-specified here; see backend LLD §3.

Two acceptance criteria have no existing implementation to hook into anywhere in this codebase and are
flagged rather than fabricated — see §4 and §5.

---

## §2 — Data Flow

### Flow A: Partner views the Customer Details page (Surface 1 eligibility)

```
User action: Partner opens the Customer Details page
  → CustomerProductsOverviewPanel renders → ActiveProducts renders with `products` + `customerId`
  → ActiveProducts calls useOfferSwitchPaths(customerId, products) — one query per subscription
  → GET /api/offer-switch-paths?customerId=&subscriptionId=&offset=0&limit=20  (per subscription)
  ← response: { totalCount, count, offset, limit, productUpgrades: [{ sourceBaseOfferId, targetList }] }
  ← hook provides: { [subscriptionId]: { targets: TargetOffer[], isLoading } } — targets=[] on any error (silent)
  ← ActiveProducts renders: "Upgrade" button enabled only if offerId is set AND targets.length > 0
```

### Flow B: Partner selects an upgrade target (Surface 2)

```
User action: Partner presses "Upgrade" on a subscription card
  → ActiveProducts opens UpgradePathsDialog with the card's already-fetched `targets` (no new discovery call)
  → UpgradePathsDialog calls useProductNamesFromPricelist(targets.map(t => ({offerId: t.targetBaseOfferId, currencyCode, region})))
  → POST /api/pricelist (one call, batched via useQueries) — per target offer, live lookup
  ← response: pricelist offers; name resolved per target, or raw offer ID string if lookup fails/misses
  ← dialog renders: radio list (icon, name/fallback, offer ID) + "Upgrade to: {name}" once selected
User action: Partner selects a target, presses "Proceed"
  → onProceed({ targetOfferId, switchType, targetProductName }) fires
  → ActiveProducts calls router.push('/upgradeOrderSummary', { query: {...} }) — all data via URL query params
```

### Flow C: Price preview auto-fires on the Upgrade Order Summary page (Surface 3)

```
User action: Partner lands on /upgradeOrderSummary (navigation from Flow B)
  → page mounts → one-shot effect (guarded by a ref, router.isReady) fires runPreview()
  → POST /api/orders?type=PreviewSwitch&fetch-price=true
  → body: { customerId, orderType: 'PREVIEW_SWITCH', currencyCode, lineItems: [{extLineItemNumber:1, offerId: targetOfferId, quantity, flexDiscountCodes?}], cancellingItems: [{extLineItemNumber:1, referenceLineItemNumber:1, subscriptionId: sourceSubscriptionId, quantity}] }
  — note: the cancelling item's `quantity` is always the SAME value as the line item's `quantity` (the
    partner is swapping N licenses from source to target, not necessarily the source's full original
    quantity) — it is NOT the `sourceQuantity` query param, which is only the upper bound cap
  ← response (per backend LLD §2): { orderType:'PREVIEW_SWITCH', lineItems:[{pricing, proratedDays}], cancellingItems, pricingSummary:[{totalLineItemPartnerPrice, currencyCode}] }
  ← page provides: per-license price, proratedDays, line total, estimated total (+ its own currency), isPreviewing
  ← page passes state down to UpgradeOrderSummaryCard → price fields + "(N days proration)" caption populate;
    Place order enabled once resolved; inline error (backend message when available, else generic fallback
    text, see §3 `utils/orderPreviewErrorMessages.ts`) if the call fails
```

### Flow D: Partner edits quantity or applies a promo code (Surface 3)

```
User action: Partner edits the (editable) quantity field, or applies/removes a promo code
  → quantity change: setState + needsRecalculation=true (Update price becomes enabled; no auto-fire)
  → promo code apply/remove: runs the same POST /api/orders?type=PreviewSwitch immediately (independent of
    the Update price button), with flexDiscountCodes set/cleared on the target line item
  ← same response shape as Flow C; price fields + estimated total refresh
```

### Flow E: Partner places the order (Surface 3)

```
User action: Partner presses "Place order" (enabled once a preview has resolved, nothing pending)
  → POST /api/orders?type=Switch
  → body: same shape as the preview call, orderType: 'SWITCH', independent of the last preview response's own values
  ← response (per backend LLD §2): { orderType:'SWITCH', orderId, status, lineItems, cancellingItems }
  ← success: router.push('/orderConfirmation', { query: { customerId, currencyCode, totalAmount: <local
    estimatedTotal>, customerName, resellerId, products: JSON.stringify([{productFamily: targetProductName,
    quantity}]) } }) — existing route, unmodified, values from local state not the SWITCH response
  ← failure: fixed inline message "Failed to place order. Please try again." — backend detail intentionally
    not surfaced (see §4); partner stays on the page and must press Place order again manually
```

---

## §3 — Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `hooks/useOfferSwitchPaths.ts` | CREATE | Data Fetch | Per-subscription discovery query backing Surface 1's "Upgrade" button eligibility |
| `types/upgrade.ts` | CREATE | Util | Client-side types mirroring the backend's `OfferSwitchPath` response + upgrade line/cancelling item shapes |
| `utils/upgradeOrderUtils.ts` | CREATE | Util | `PreviewSwitch`/`Switch` request builders + fetch functions for Surface 3 |
| `utils/orderPreviewErrorMessages.ts` | CREATE | Util | Reason-code → partner-friendly text mapping for order-preview failures (new; not reused elsewhere in this LLD) |
| `components/upgrade/UpgradePathsDialog.tsx` | CREATE | Component | Surface 2 — target-offer selection dialog |
| `styles/upgrade/UpgradePathsDialog.module.css` | CREATE | Style | Styles for the above dialog |
| `pages/upgradeOrderSummary.tsx` | CREATE | Route | Surface 3 — preview + place-order page; orchestration only (state, data fetch, handlers), renders the two panel components below |
| `components/upgrade/UpgradeReviewDetails.tsx` | CREATE | Component | Surface 3 left panel — "Review upgrade details" checklist card, modeled on `CheckoutReviewDetails.tsx` |
| `components/upgrade/UpgradeOrderSummaryCard.tsx` | CREATE | Component | Surface 3 right panel — "Order summary" card (product, quantity, pricing, Update price, Estimated Total, Place order), modeled on `CheckoutSummary.tsx` + `CheckoutProductCard.tsx` |
| `styles/upgrade/UpgradeOrderSummary.module.css` | CREATE | Style | Styles for the page + both panel components above |
| `components/customerdetails/ActiveProducts.tsx` | MODIFY | Component | Add "Upgrade" action, discovery hook call, dialog wiring, navigation to Surface 3 |

Not listed (owned by the backend LLD, already specified there — see `docs/ai-kit/LLD/backend/anytime-upgrade-apispec-lld.md` §3): `models/Order.ts` (`OrderTypeEnum`, `CancellingItemSchema`, `PreviewSwitchOrderSchema`, `SwitchOrderSchema`), `utils/constants.ts` (`ORDER_API_TYPE.PREVIEW_SWITCH`/`SWITCH`), `pages/api/offer-switch-paths.ts`, `pages/api/orders.ts`, `controllers/*`, `models/OfferSwitchPath.ts`. The files below assume those changes have landed.

---

### `hooks/useOfferSwitchPaths.ts` — CREATE

New file, following `hooks/useProductPricing.ts`'s exact shape: a co-located `fetch*` function + a
`useQueries`-based hook for N parallel per-item GET queries (not `utils/AccountApis.ts`, matching the
convention that per-item pricelist-style lookups live inside their hook file).

- **Fetch function** `fetchOfferSwitchPaths(customerId: string, subscriptionId: string): Promise<OfferSwitchPathsResponse>` — `GET /api/offer-switch-paths?customerId=&subscriptionId=&offset=0&limit=20`. Parses JSON; on non-OK response or parse failure, throws (caller treats as empty per the fail-silent posture below — this function itself does not swallow errors, matching `fetchProductPricing`'s throw-on-failure convention).
- **Cache key**: `['offerSwitchPaths', customerId, subscriptionId]` — new domain, does not collide with anything in `DATA_LAYER.md`.
- **Freshness**: `staleTime: 5 * 60 * 1000`, `gcTime: 10 * 60 * 1000`, `retry: 1` — matches the "Pricelist" cache tier in `STATE_MANAGEMENT.md §Cache Strategy` (this is a similarly-volatile discovery lookup, not a long-lived config).
- **Fetch guard**: `enabled: !!customerId && !!sub.subscriptionId && !!sub.offerId` per query.
- **Hook**:
  ```ts
  export interface OfferSwitchPathsResult {
    targets: TargetOffer[];
    isLoading: boolean;
  }

  export function useOfferSwitchPaths(
    customerId: string | undefined,
    subscriptions: { subscriptionId: string; offerId?: string }[]
  ): Record<string, OfferSwitchPathsResult> {
    const queries = useQueries({
      queries: subscriptions.map(sub => ({
        queryKey: ['offerSwitchPaths', customerId, sub.subscriptionId],
        queryFn: () => fetchOfferSwitchPaths(customerId!, sub.subscriptionId),
        enabled: !!customerId && !!sub.subscriptionId && !!sub.offerId,
        staleTime: 5 * 60 * 1000,
        gcTime: 10 * 60 * 1000,
        retry: 1,
      })),
    });

    const result: Record<string, OfferSwitchPathsResult> = {};
    subscriptions.forEach((sub, i) => {
      const query = queries[i];
      // Per the experience card: eligibility-call failure is silent — resolves as empty, no error surfaced.
      const targets = query.error ? [] : (query.data?.productUpgrades?.[0]?.targetList ?? []);
      result[sub.subscriptionId] = { targets, isLoading: query.isLoading };
    });
    return result;
  }
  ```
- **Graceful degradation**: per-subscription errors resolve to `targets: []` — this is the experience card's
  explicit "silent failure" requirement (§Surface 1), not an oversight. No `ErrorToast`, no retry affordance.

---

### `types/upgrade.ts` — CREATE

Plain TypeScript types (no Zod) — matches the existing convention for order-flow client types
(`types/order.ts`, `types/lateRenewal.ts` are un-validated interfaces; server-side validation already
happened via the backend's `models/OfferSwitchPath.ts` / `models/Order.ts` before the browser ever sees the
payload).

```ts
export type SwitchType = 'PARTIAL_ALLOWED' | 'FULL_ONLY';

export interface TargetOffer {
  targetBaseOfferId: string;
  sequence: number;
  switchType: SwitchType;
}

export interface ProductUpgrade {
  sourceBaseOfferId: string;
  targetList: TargetOffer[];
}

export interface OfferSwitchPathsResponse {
  totalCount: number;
  count: number;
  offset: number;
  limit: number;
  productUpgrades: ProductUpgrade[];
}

export interface UpgradeCancellingItem {
  extLineItemNumber: number;
  referenceLineItemNumber: number;
  subscriptionId: string;
  quantity: number;
}
```

`UpgradeCancellingItem`'s shape mirrors the backend LLD's `CancellingItemSchema` request-side fields exactly
(§3 of the backend LLD) — no `offerId`/`pricing` here since those are response-only.

---

### `utils/upgradeOrderUtils.ts` — CREATE

New file, following `utils/renewalOrderUtils.ts`'s exact shape (line-item builders + two `fetch*` functions
that hit `/api/orders` with a `type` query param and parse the same error shape every other order-flow util
already uses).

```ts
import { LineItem, Order } from '../models/Order';
import { UpgradeCancellingItem } from '../types/upgrade';
import { ORDER_API_TYPE, HTTP_METHOD } from './constants'; // PREVIEW_SWITCH / SWITCH — added by backend LLD

export const buildUpgradeLineItem = (
  targetOfferId: string,
  quantity: number,
  discountCode?: string
): LineItem => ({
  extLineItemNumber: 1,
  offerId: targetOfferId,
  quantity,
  ...(discountCode && { flexDiscountCodes: [discountCode] }),
});

// `quantity` must always mirror the target line item's quantity — a switch order cancels exactly
// as many licenses of the source subscription as are being upgraded, not the source subscription's
// full original quantity. Callers must pass the current (possibly edited) quantity here, never the
// quantity cap (`sourceQuantity`/`maxQuantity`).
export const buildUpgradeCancellingItem = (
  sourceSubscriptionId: string,
  quantity: number
): UpgradeCancellingItem => ({
  extLineItemNumber: 1,
  referenceLineItemNumber: 1,
  subscriptionId: sourceSubscriptionId,
  quantity,
});

export const fetchUpgradeOrderPreview = async (
  customerId: string,
  lineItems: LineItem[],
  cancellingItems: UpgradeCancellingItem[],
  currencyCode: string
): Promise<Order> => {
  const url = `/api/orders?type=${ORDER_API_TYPE.PREVIEW_SWITCH}&fetch-price=true`;
  const response = await fetch(url, {
    method: HTTP_METHOD.POST,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      customerId,
      orderType: 'PREVIEW_SWITCH',
      currencyCode,
      lineItems,
      cancellingItems,
    }),
  });
  if (!response.ok) {
    const errorData = await response.json().catch(() => ({}));
    throw new UpgradePreviewError(errorData);   // see utils/orderPreviewErrorMessages.ts
  }
  const data = await response.json();
  if (!data.lineItems || data.lineItems.length === 0) {
    throw new Error('No line items returned from upgrade order preview API');
  }
  return data;
};

export const placeUpgradeOrder = async (
  customerId: string,
  lineItems: LineItem[],
  cancellingItems: UpgradeCancellingItem[],
  currencyCode: string
): Promise<Order> => {
  const url = `/api/orders?type=${ORDER_API_TYPE.SWITCH}`;
  const response = await fetch(url, {
    method: HTTP_METHOD.POST,
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ customerId, orderType: 'SWITCH', currencyCode, lineItems, cancellingItems }),
  });
  if (!response.ok) {
    // Deliberately generic — the experience card requires the backend failure reason NOT be surfaced here
    // (unlike every other order-placement util in this codebase). See §4 Design Decisions.
    throw new Error('Failed to place order. Please try again.');
  }
  return response.json();
};
```

- **Cache/mutation**: neither function is a TanStack Query hook — both are called directly from
  `pages/upgradeOrderSummary.tsx`'s own `useState`/`useCallback` handlers, matching `renewalOrderUtils.ts`'s
  pattern (that file's functions are also called directly, not wrapped in `useMutation`).
- **Graceful degradation**: preview failure surfaces a message (via `UpgradePreviewError` /
  `orderPreviewErrorMessages.ts`); place-order failure is intentionally flattened to one fixed string.

---

### `utils/orderPreviewErrorMessages.ts` — CREATE

New shared util implementing the two-tier fallback Surface 3's "Pricing preview" section specifies verbatim:
"an inline message shows the backend's error message when available, falling back to 'Failed to load pricing.
Click 'Update price' to retry.' if not."

```ts
export class UpgradePreviewError extends Error {
  constructor(public readonly errorData: { message?: string; additionalDetails?: string[] }) {
    super(errorData.message || 'Upgrade preview failed');
  }
}

export const getPreviewErrorMessage = (err: unknown): string => {
  if (err instanceof UpgradePreviewError && err.errorData.message) {
    return err.errorData.message;
  }
  if (err instanceof Error && err.message) return err.message;
  return "Failed to load pricing. Click 'Update price' to retry.";
};
```

- **Consumer**: `pages/upgradeOrderSummary.tsx`'s preview `catch` block calls `getPreviewErrorMessage(err)`.

---

### `components/upgrade/UpgradePathsDialog.tsx` — CREATE

New file. Custom backdrop+`<div>` modal, matching every existing "dialog" in this codebase
(`EditRenewalDialog.tsx`, `LateRenewalOrderDialog.tsx`) — this app does not use Spectrum's native
`Dialog`/`DialogTrigger` anywhere, so introducing one here would be inconsistent. Header reuses
`components/renewal/RenewalDialogHeader.tsx` (title + Cancel/Proceed buttons) rather than a new header
component, since its props already cover this dialog's needs (title, onCancel, onSave, isSaveDisabled,
saveButtonLabel).

**Props:**

| Prop | Type | Required | Notes |
|---|---|---|---|
| `isOpen` | `boolean` | Yes | |
| `onClose` | `() => void` | Yes | Cancel — no data captured |
| `onProceed` | `(target: { targetOfferId: string; switchType: SwitchType; targetProductName: string }) => void` | Yes | Fired on Proceed; caller (`ActiveProducts`) handles navigation |
| `sourceProductName` | `string` | Yes | For the "Current product: {name}" header line |
| `sourceOfferId` | `string` | Yes | Used to derive market segment / discount-level context for the pricelist lookup, matching `getMarketSegmentFromOfferId` usage elsewhere |
| `currencyCode` | `string` | Yes | Passed to the pricelist lookup for every target |
| `targets` | `TargetOffer[]` | Yes | Already fetched by `ActiveProducts` via `useOfferSwitchPaths` — **no new discovery call is made here**, per the experience card |

**Visual spec (approximate — no Figma reference):**

- Backdrop `<div>` (click-to-close) + dialog `<div>`, styled per `styles/upgrade/UpgradePathsDialog.module.css`.
- Header (via `RenewalDialogHeader`): title `"Available upgrades"`, Cancel button (`onCancel={onClose}`), Proceed button (`onSave={handleProceed}`, `saveButtonLabel="Proceed"`, `isSaveDisabled={!selectedTargetOfferId}`).
- Below header: `"Current product: <strong>{sourceProductName}</strong>"`.
- While `useProductNamesFromPricelist(...).isLoading`: render `"Loading upgrade options..."` in place of the list (loading state — no skeleton, matching the experience card's literal copy).
- Once loaded: `RadioGroup` (Spectrum `RadioGroup`/`Radio`, matching `ExistingCustomer.tsx`'s pattern) — one `Radio` per target, rendered in `targets` array order (the `sequence` field is **not** used to sort, per the experience card). Each option shows: product icon (`getIconWithFallback(name, '32x32')`), product name or the raw `targetBaseOfferId` fallback if the lookup missed/failed, and the offer ID string.
- Once a target is selected: summary line `"Upgrade to: <strong>{selected product name}</strong>"` appears below the radio list.
- No explicit empty-state — per the experience card, this dialog is only ever opened when `targets.length > 0` (enforced by `ActiveProducts`'s button-disable condition), so an empty list is not a state this component needs to render.

**Interaction logic:**

- Internal state: `selectedTargetOfferId: string | null` (default `null`).
- `handleProceed`: looks up the matching `TargetOffer` by `selectedTargetOfferId`, calls
  `onProceed({ targetOfferId, switchType, targetProductName: getProductName(targetOfferId) ?? targetOfferId })`.
- Cancel (`onClose`) does not reset `selectedTargetOfferId` before the next open — the parent unmounts/remounts
  the dialog by keying it on the source subscription, so stale selection state does not leak between
  different subscriptions' dialogs (`ActiveProducts` renders one dialog instance, reset via a `useEffect` on
  the `sourceOfferId` prop changing, mirroring `LateRenewalOrderDialog`'s open-transition reset pattern).

**Accessibility:** `RadioGroup` has an explicit `aria-label="Available upgrade targets"` (Spectrum provides
roving-focus + ARIA roles internally, per `UI_CODE_PATTERNS.md §B.5`); the backdrop `<div>` gets
`role="presentation"` (not itself focusable/interactive — closing is a click affordance, not a required a11y
path, consistent with `EditRenewalDialog.tsx`'s existing backdrop).

---

### `styles/upgrade/UpgradePathsDialog.module.css` — CREATE

CSS Modules, per `UI_CODE_PATTERNS.md`. Mirrors `styles/renewal/Renewal.module.css`'s backdrop/dialog/content
shell classes (fixed-position backdrop, centered dialog panel, scrollable content area) since this dialog is
structurally identical to the renewal dialogs it's modeled on:

```css
.backdrop { position: fixed; inset: 0; background: rgba(0,0,0,0.4); z-index: 100; }
.dialog { position: fixed; top: 50%; left: 50%; transform: translate(-50%,-50%); z-index: 101;
  background: white; border-radius: 8px; width: 480px; max-height: 80vh; overflow-y: auto; }
.content { padding: 24px; display: flex; flex-direction: column; gap: 16px; }
.currentProductLine { font-size: 14px; color: var(--spectrum-gray-700, #444); }
.loadingText { font-size: 14px; color: var(--spectrum-gray-600, #666); padding: 16px 0; }
.targetOption { display: flex; align-items: center; gap: 12px; }
.targetIcon { width: 32px; height: 32px; }
.targetOfferId { font-size: 12px; color: var(--spectrum-gray-600, #666); }
.upgradeToSummary { font-weight: 600; font-size: 14px; }
```

---

### `pages/upgradeOrderSummary.tsx` — CREATE

New page (Pages Router file = route `/upgradeOrderSummary`, matching `ROUTES.md`'s conventions — does not
collide with any existing entry). **Entry point:** reachable only via `router.push` from
`UpgradePathsDialog`'s Proceed action inside `ActiveProducts.tsx`; there is no other way to reach this URL.
**No auth guard** — matches every other page in this app (auth is entirely server-side per `ROUTES.md`).
Now an **orchestration-only** page — mirroring `pages/checkout.tsx`'s own division of responsibility, all
query-param parsing, state, derived values, and data fetch/mutation calls live here; all rendering below the
breadcrumb is delegated to `UpgradeReviewDetails` (left panel) and `UpgradeOrderSummaryCard` (right panel).

**Data-passing strategy:** all inputs arrive via URL query params (no `CartContext`, no router state) —
matching the experience card's explicit requirement ("nothing is passed through app context or session
storage") and `ROUTES.md`'s documented "State-passing strategy: Query string parameters" convention used by
`orderConfirmation.tsx` today.

**Query params consumed:**

| Param | Required | Notes |
|---|---|---|
| `customerId` | Yes | |
| `customerName` | Yes | Display only |
| `resellerId` | Yes | Used to fetch reseller name via existing `getResellerDetails` |
| `sourceOfferId` | Yes | Display only (icon in the transition row) |
| `sourceSubscriptionId` | Yes | Becomes the cancelling item's `subscriptionId` |
| `sourceCurrencyCode` | Yes | Default currency until the preview response's own currency resolves |
| `sourceQuantity` | Yes | The subscription's quantity **at the time the upgrade was started** — becomes the quantity field's cap (`maxQuantity`) only. It is **not** what's sent as the cancelling item's quantity — that always mirrors the current (possibly edited) `quantity` state, see below. |
| `sourceProductName` | Yes | Display only |
| `targetOfferId` | Yes | The line item's `offerId`; also displayed as "Offer ID: {targetOfferId}" |
| `targetProductName` | Yes | Display only — resolved once in `UpgradePathsDialog`, not re-fetched here |
| `switchType` | Yes | `'PARTIAL_ALLOWED' \| 'FULL_ONLY'` — decides quantity editability |

**State variables:**

| Variable | Type | Default | Triggers re-fetch? |
|---|---|---|---|
| `quantity` | `number` | `sourceQuantity` (parsed int) | No — only flips `needsRecalculation`; a fresh preview requires pressing Update price |
| `discountCode` | `string \| undefined` | `undefined` | Yes — apply/remove immediately re-runs the preview |
| `needsRecalculation` | `boolean` | `false` | N/A (gates the Update price button, mirrors `CheckoutSummary.tsx`'s `!needsRecalculation \|\| isRecalculating` disable expression) |
| `isPreviewing` | `boolean` | `false` | N/A |
| `isPlacingOrder` | `boolean` | `false` | N/A |
| `previewError` | `string \| null` | `null` | Local only |
| `placeOrderError` | `string \| null` | `null` | Local only |
| `pricePerLicense` / `lineTotal` / `estimatedTotal` | `number \| null` | `null` | Local only — set from preview response |
| `proratedDays` | `number \| null` | `null` | Local only — set from the preview response's line item `proratedDays` field (per backend LLD `LineItemSchema`); rendered as the "(N days proration)" caption — read directly off the response, no client-side proration math |
| `previewCurrency` | `string` | `sourceCurrencyCode` | Local only — overwritten once the preview response returns its own `currencyCode` (may differ from the source currency, per the experience card) |

**Derived values:**

- `maxQuantity = parseInt(sourceQuantity, 10)`.
- `isQuantityEditable = switchType === 'PARTIAL_ALLOWED'` — when `false` (`FULL_ONLY`), the quantity `NumberField` is rendered `isReadOnly` and fixed at `maxQuantity`.
- `discountLevelText` — `getDiscountLevelsFromOfferIdsAndQuantities([{ offerId: targetOfferId, quantity }]).LICENSE?.discountLevel`, the same existing `utils/commonUtils.ts` helper `CheckoutSummary.tsx`/`LateRenewalOrderDialog.tsx` already call, applied to the single target line item. Feeds the "{quantity} Licenses ({discountLevelText})" subheading — see `UpgradeOrderSummaryCard` below.
- `canPlaceOrder = lineTotal != null && estimatedTotal != null && !needsRecalculation && !isPreviewing && !isPlacingOrder` — **see §4**: a write-permission clause belongs in this expression per the experience card but has no implementation source anywhere in this codebase; left as an explicit extension point (see below), not fabricated.

**Data fetch / mutation calls:**

- `useQuery({ queryKey: ['reseller', resellerId], queryFn: () => getResellerDetails(resellerId), enabled: !!resellerId, staleTime: 10*60*1000, gcTime: 10*60*1000 })` — reuses the existing `getResellerDetails` fetch unit (already in `DATA_LAYER.md`) purely for `resellerName` display; no new fetch code. Same query shape `pages/checkout.tsx` already uses for its own `resellerData`.
- `runPreview()` — calls `fetchUpgradeOrderPreview` (see `utils/upgradeOrderUtils.ts`) with `buildUpgradeLineItem(targetOfferId, quantity, discountCode)` + `buildUpgradeCancellingItem(sourceSubscriptionId, quantity)` — **both built from the same `quantity` value**, so a quantity edit that's later re-previewed cancels exactly as many source licenses as are being upgraded, not the full original `maxQuantity`. Fires once automatically on mount (guarded by a `useRef` flag + `router.isReady`, matching `LateRenewalOrderDialog`'s `hasInitialPreview` guard). On success: populates `pricePerLicense`/`lineTotal`/`estimatedTotal`/`proratedDays`/`previewCurrency`, clears `needsRecalculation`. On failure: `setPreviewError(getPreviewErrorMessage(err))`. The same pairing (`buildUpgradeLineItem(targetOfferId, quantity, discountCode)` + `buildUpgradeCancellingItem(sourceSubscriptionId, quantity)`) is used again, unchanged, in `handlePlaceOrder()`.
- `handlePlaceOrder()` — calls `placeUpgradeOrder`; on success navigates to `/orderConfirmation` (existing route, unmodified) with `totalAmount` = local `estimatedTotal`, `products = [{ productFamily: targetProductName, quantity }]` — explicitly **not** the SWITCH response's own values, per the experience card. On failure: `setPlaceOrderError('Failed to place order. Please try again.')` — fixed text, no backend detail (see §4).

**Screen layout (mirrors `pages/checkout.tsx`'s page shell exactly — see §4):**

1. `<Layout activePage="checkout">` (closest existing `Sidebar` nav item; there is no dedicated "Upgrade" entry — see `ROUTES.md`/`Sidebar.tsx`).
2. `<NavigationPanel items={[{label:'Resellers', href:'/resellers'}, {label:resellerName||'Reseller', href:'/customers?resellerId='+resellerId}, {label:customerName||'Customer', href:'/customerdetails?resellerId='+resellerId+'&customerId='+customerId}, {label:'Review upgrade', href:undefined}]} />` — identical breadcrumb-construction pattern to `pages/checkout.tsx`'s `navigationItems`, with the trailing crumb changed to "Review upgrade" (matches the experience card's screenshot exactly, including its literal "Reseller"/"Customer" fallback labels when those names haven't resolved yet).
3. Two-column container (`styles.upgradeContainer` → `.leftColumn` / `.rightColumn`, same flex ratios as `Checkout.module.css`'s `.checkoutContainer`):
   - **Left column:** `<UpgradeReviewDetails customerName resellerName sourceProductName targetProductName />`
   - **Right column:** `<UpgradeOrderSummaryCard ... />` (full prop list — see its own subsection below) passing every piece of state/derived-value computed above, plus `onQuantityChange`, `onPromoCodeApply`/`onPromoCodeRemove`, `onUpdatePrice={() => runPreview()}`, `onPlaceOrder={handlePlaceOrder}`.
4. If any required query param is missing (`!hasRequiredParams`, checked once `router.isReady`): render a single inline `"Missing required upgrade details."` message inside `<Layout>` instead of the two-column layout (defensive; the experience card does not specify this case since the page is only ever reached via Flow B's navigation).

**Loading / error / empty states:** delegated to `UpgradeOrderSummaryCard` (previewError / placeOrderError / initial-load spinner) — see that component's subsection. The page owns the state, the card owns the rendering.

**Accessibility:** `router.isReady`-gated render avoids a flash of the "Missing required upgrade details." message before query params resolve; the page itself renders no heading (the `<h1>` lives in `UpgradeReviewDetails` — see below, matching `UI_CODE_PATTERNS.md`'s "`<h1>` on page titles" rule).

---

### `components/upgrade/UpgradeReviewDetails.tsx` — CREATE

New file, modeled directly on `components/checkout/CheckoutReviewDetails.tsx` (read in full) — same
`Preview` icon + heading + `<ul>` of `CheckmarkCircle`-prefixed `<li>` items, same `styles.reviewCard` /
`.reviewList` / `.reviewListItem` / `.reviewIcon` shell classes (added to the upgrade CSS module below with
equivalent values) — with upgrade-specific heading text, an extra non-checkmarked transition row, and
upgrade-specific checklist copy taken verbatim from the experience card's Surface 3 screen layout.

**Props:**

| Prop | Type | Required | Notes |
|---|---|---|---|
| `customerName` | `string` | Yes | |
| `resellerName` | `string` | Yes | |
| `sourceProductName` | `string` | Yes | Icon + label in the transition row; also interpolated into the pro-ration checklist line |
| `targetProductName` | `string` | Yes | Icon + label in the transition row; also interpolated into the pro-ration checklist line |

**Visual spec (grounded in the experience card's screenshot + `CheckoutReviewDetails.tsx`'s exact markup):**

- `<section className={styles.reviewCard}>` (same card shell as Checkout's — white background, rounded, padded, drop shadow).
- Header row (`styles.reviewHeaderRowJustified`, matching `CheckoutReviewDetails`'s row exactly): `Preview` icon (`@react-spectrum/s2/icons/Preview`) + `<h1 className={styles.reviewTitle}>Review upgrade details</h1>` — deliberately `<h1>` here, not the `<h2>` `CheckoutReviewDetails.tsx` literally uses, to satisfy `UI_CODE_PATTERNS.md`'s "`<h1>` on page titles" rule without changing the visual weight (same CSS class, `styles.reviewTitle`, controls appearance either way).
- `<ul className={styles.reviewList}>`, five items in this exact order (matches the experience card's Surface 3 "Left panel" bullets):
  1. `<li className={styles.reviewListItem}><CheckmarkCircle/> This purchase is for <strong>{customerName}</strong> on behalf of <strong>{resellerName}</strong>.</li>`
  2. `<li className={styles.reviewListItem}><CheckmarkCircle/> Upgraded licenses will be immediately available and automatically assigned.</li>`
  3. **No checkmark** — transition row: `<li className={styles.transitionRow}>` containing `getIconWithFallback(sourceProductName, '48x48')` + `"Licenses"`, a `→` separator (`styles.transitionArrow`), then `getIconWithFallback(targetProductName, '48x48')` + `"Licenses"` — matches the screenshot's icon-only row exactly (no checklist bullet on this row).
  4. `<li className={styles.reviewListItem}><CheckmarkCircle/> Adobe is not responsible for billing these licenses to the customer or collecting payments.</li>`
  5. `<li className={styles.reviewListItem}><CheckmarkCircle/> The amount is the partner price difference between <strong>{sourceProductName}</strong> and <strong>{targetProductName}</strong> and pro-rated to the next anniversary date.</li>`

**Interaction logic:** none — purely presentational, no local state, no handlers.

**Loading / error / empty states:** none applicable — all props are already-resolved strings by the time the parent page renders this component (see the page's `hasRequiredParams` gate).

**Accessibility:** `<h1>` per above; `CheckmarkCircle` icons are decorative (the adjacent `<Text>` already conveys the full meaning) so no additional `aria-label` is added, matching `CheckoutReviewDetails.tsx`'s existing posture.

---

### `components/upgrade/UpgradeOrderSummaryCard.tsx` — CREATE

New file, modeled on `components/checkout/CheckoutSummary.tsx` + `components/checkout/CheckoutProductCard.tsx`
(both read in full) collapsed into one component, since this screen only ever has exactly one line item (the
target offer) — unlike Checkout's summary, which maps over N cart items via a separate `CheckoutProductCard`
per item. Reuses `CheckoutSummary.tsx`'s exact "Estimated Total" row markup/classes and terms-line copy, and
`CheckoutProductCard.tsx`'s exact product-card markup/classes (icon, `Heading`, "Offer ID:" line, `NumberField`
+ price layout) — see §4 for why `EstimatedTotal.tsx` (the shared common component) is *not* reused here.

**Props:**

| Prop | Type | Required | Notes |
|---|---|---|---|
| `targetProductName` | `string` | Yes | |
| `targetOfferId` | `string` | Yes | Rendered as "Offer ID: {targetOfferId}" |
| `quantity` | `number` | Yes | Current (possibly edited) quantity |
| `maxQuantity` | `number` | Yes | `NumberField` `maxValue` |
| `isQuantityEditable` | `boolean` | Yes | `NumberField` `isReadOnly={!isQuantityEditable}` |
| `onQuantityChange` | `(value: number) => void` | Yes | Sets `quantity` + `needsRecalculation=true` in the parent, only wired when editable |
| `discountLevelText` | `string \| undefined` | No | Feeds the "{quantity} Licenses ({discountLevelText})" subheading; omitted (falls back to "{quantity} Licenses") if not yet resolved |
| `pricePerLicense` | `number \| null` | Yes | |
| `proratedDays` | `number \| null` | Yes | Renders "(N days proration)" under the per-license price when non-null |
| `lineTotal` | `number \| null` | Yes | Shown beside the quantity stepper |
| `estimatedTotal` | `number \| null` | Yes | |
| `currency` | `string` | Yes | |
| `discountCode` | `string \| undefined` | No | |
| `onPromoCodeApply` | `(offerId: string, code: string) => void` | Yes | |
| `onPromoCodeRemove` | `(offerId: string) => void` | Yes | |
| `needsRecalculation` | `boolean` | Yes | Gates Update price |
| `isPreviewing` | `boolean` | Yes | Gates Update price + shows the initial-load spinner |
| `onUpdatePrice` | `() => void` | Yes | |
| `canPlaceOrder` | `boolean` | Yes | |
| `isPlacingOrder` | `boolean` | Yes | |
| `onPlaceOrder` | `() => void` | Yes | |
| `previewError` | `string \| null` | Yes | |
| `placeOrderError` | `string \| null` | Yes | |

**Visual spec (grounded in the experience card's screenshot + the two Checkout components it's modeled on):**

- `<aside className={styles.summaryCard}>` (same card shell as `CheckoutSummary.tsx`'s `.summaryCard`).
- `<Heading level={2}>Order summary</Heading>` (sentence case, matching the screenshot exactly — not Checkout's "Order Summary").
- Subheading block (`styles.partnerPricingText`, reusing Checkout's class): `"{quantity} Licenses ({discountLevelText})"` (discount-level suffix omitted if unresolved) then, on its own line, `"Reflecting partner pricing for customer."` — note the trailing "for customer." the screenshot adds, vs. Checkout's plain "Reflecting partner pricing."
- Product card (`styles.productCard`, reusing `CheckoutProductCard.tsx`'s exact shell/column classes):
  - Icon column: `getIconWithFallback(targetProductName, '48x48')`.
  - Info column: `<Heading level={3}>{targetProductName}</Heading>`, `<Text>Offer ID: {targetOfferId}</Text>`.
  - Bottom row (`styles.productCardBottomRow`): `NumberField` (`label="Licenses"`, `value={quantity}`, `minValue={1}`, `maxValue={maxQuantity}`, `isReadOnly={!isQuantityEditable}`, `hideStepper={false}`, `onChange={onQuantityChange}`) on the left; on the right, `formatPrice(lineTotal, currency)` (the line total shown beside the stepper, matching the screenshot's "$39,568.55" placement).
  - Below the bottom row: `"{formatPrice(pricePerLicense, currency)} per license"` and, on the next line, `"({proratedDays} days proration)"` when `proratedDays != null` — new caption, not present in Checkout, sourced from the backend's `proratedDays` field per the experience card's screenshot.
  - `PromoCodeButton` (reused from `components/common/PromoCodeButton.tsx`, unmodified) right-aligned in the same row group — its unapplied-state label is literally `"Apply discount code"`, matching the screenshot's link text with no copy changes needed.
- `previewError`, if non-null: inline error block (`styles.errorMessage`) wrapped in `aria-live="polite"`, above the Update price button; Place order stays disabled until a successful preview clears it.
- `UpdatePriceButton` (reused from `components/common/UpdatePriceButton.tsx`, unmodified), `isDisabled={!needsRecalculation || isPreviewing}`.
- Estimated Total row — **bespoke markup mirroring `CheckoutSummary.tsx`'s `.summaryTotalRow`/`.priceContainer`/`.taxText` classes exactly** (not the shared `EstimatedTotal.tsx` component — see §4): `<Text>Estimated Total</Text>` on the left, `formatPrice(estimatedTotal, currency)` + `<span className={styles.taxText}>+local taxes apply</span>` stacked on the right.
- Terms text (`styles.termsText`, copy identical to `CheckoutSummary.tsx`'s, already matching the screenshot verbatim): `By clicking "Place order", you will agree to pay Adobe for these licenses pursuant to Adobe's APC Agreement with you.`
- `Place order` button — Spectrum `Button` styled full-width/blue via `UNSAFE_className` mirroring `CheckoutSummary.tsx`'s `.placeOrderBtn`/`.enabled`/`.disabled` classes (not default Spectrum sizing — see §4), `isDisabled={!canPlaceOrder}` (see write-permission caveat, §4 of this LLD / §4 of the page's own note above).
- `placeOrderError`, if non-null: inline error block (`styles.errorMessage`) wrapped in `aria-live="polite"`, below the Place order button.

**Interaction logic:** every interaction is a callback prop invocation — this component holds no local state
of its own (`quantity`/`discountCode`/preview state all live in the parent page, per the orchestration split
above).

**Loading / error / empty states:**
- While `isPreviewing` is `true` **and** `lineTotal == null` (the initial load, not a subsequent Update price press): render a `ProgressCircle` (`aria-label="Loading pricing"`, `isIndeterminate`) in place of the per-license-price/proration text, matching `CheckoutSummary.tsx`'s `ProgressCircle` usage pattern on its own Update-price button.
- `previewError` / `placeOrderError` inline blocks as specified above.
- No empty state — this component is never rendered with zero line items (the page's `hasRequiredParams` gate ensures a `targetOfferId` always exists before this component mounts).

**Accessibility:** `aria-label="Upgrade quantity"` on the `NumberField`; `aria-live="polite"` regions around `previewError` and `placeOrderError` so screen readers announce failures without requiring focus to move.

---

### `styles/upgrade/UpgradeOrderSummary.module.css` — CREATE

CSS Modules. Now styles three files (the page shell + both panel components) rather than one, mirroring
`components/checkout/Checkout.module.css`'s actual class set (that file is shared across `checkout.tsx` +
`CheckoutReviewDetails.tsx` + `CheckoutSummary.tsx` + `CheckoutProductCard.tsx` today, which is the precedent
for one CSS module backing several related files):

```css
/* Page shell — same two-column ratios as Checkout.module.css's .checkoutContainer */
.upgradeContainer { display: flex; max-width: 1400px; margin: 0 0 0 32px; gap: 32px; padding: 32px 0; }
.leftColumn { flex: 2.5; display: flex; flex-direction: column; gap: 24px; }
.rightColumn { flex: 1.5; }
.missingParamsMessage { padding: 32px; color: #444; }

/* UpgradeReviewDetails — mirrors Checkout.module.css's reviewCard/reviewList/reviewIcon classes */
.reviewCard { background: #fff; border-radius: 14px; box-shadow: 0 1px 4px rgba(0,0,0,0.06); padding: 32px; margin-bottom: 24px; text-align: left; }
.reviewHeaderRowJustified { display: flex; align-items: center; justify-content: space-between; margin-bottom: 16px; gap: 10px; }
.reviewTitle { font-family: inherit; font-size: 2rem; font-weight: 700; color: #111827; margin-bottom: 14px; }
.reviewList { font-family: inherit; color: #222; margin-bottom: 14px; list-style: none; padding: 0; font-size: 1.125rem; }
.reviewListItem { font-size: 1.125rem; color: #222; margin-bottom: 12px; display: flex; align-items: flex-start; gap: 10px; }
.reviewIcon { display: flex; align-items: center; justify-content: center; width: 24px; height: 24px; margin-right: 6px; flex-shrink: 0; color: #1e824c; }
.transitionRow { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; }
.transitionArrow { font-size: 20px; color: #6b7280; }

/* UpgradeOrderSummaryCard — mirrors Checkout.module.css's summaryCard/productCard/summaryTotalRow classes */
.summaryCard { background: #fff; border-radius: 16px; box-shadow: 0 1px 4px rgba(0,0,0,0.06); padding: 32px; min-width: 480px; }
.partnerPricingText { font-size: 14px; font-weight: 400; color: #666; }
.productCard { display: flex; align-items: flex-start; background: #f9fafb; border-radius: 12px; padding: 14px 16px; margin: 16px 0; gap: 12px; }
.productCardIconCol { display: flex; flex-direction: column; align-items: center; min-width: 48px; }
.productCardInfoCol { display: flex; flex-direction: column; flex: 1; min-width: 0; gap: 4px; }
.productCardBottomRow { display: flex; align-items: flex-start; justify-content: space-between; gap: 48px; margin-top: 6px; }
.proratedDaysText { font-size: 13px; color: #6b7280; }
.errorMessage { color: #c9252d; font-size: 14px; margin: 8px 0; }
.summaryTotalRow { display: flex; justify-content: space-between; font-weight: 700; font-size: 22px; margin: 16px 0; color: #111827; }
.priceContainer { display: flex; flex-direction: column; align-items: flex-end; gap: 2px; }
.taxText { font-size: 12px; font-weight: 400; color: #6d6d6d; white-space: nowrap; }
.termsText { color: #444; font-size: 14px; margin: 8px 0; }
.placeOrderBtn { width: 100%; height: 40px; font-size: 16px; border: none; display: flex; align-items: center; justify-content: center; gap: 8px; }
.placeOrderBtn.enabled { background-color: #1473e6; color: #fff; cursor: pointer; }
.placeOrderBtn.disabled { background-color: #eaeaea; color: #888; cursor: not-allowed; }
```

---

### `components/customerdetails/ActiveProducts.tsx` — MODIFY

Current state (read in full — see `components/customerdetails/ActiveProducts.tsx:1-159`): a presentational
component rendering one `Card` per `ProductToDisplay`, with a single "Add licenses" `Button` per card gated
by a local `isDisabled(product)` check (`offerId` present + `currencyCode` in `regionCurrencies`); the "Add
licenses" action calls `useCart()` directly (`addToCart`/`setShowCartModal`) — there is no dialog state in
this component today.

**Required changes:**

1. **New imports:** `useRouter` from `next/router`; `useOfferSwitchPaths` from `../../hooks/useOfferSwitchPaths`; `UpgradePathsDialog` from `../upgrade/UpgradePathsDialog`.
2. **New hook call**, alongside the existing `usePartnerDetails`/`useCart` calls:
   ```ts
   const upgradePaths = useOfferSwitchPaths(
     customerId,
     products.map(p => ({ subscriptionId: p.id, offerId: p.offerId }))
   );
   ```
3. **New local state:** `upgradeDialogProduct: ProductToDisplay | null` (default `null`) — which card's dialog is open, `null` = closed.
4. **New derived helper**, alongside `isDisabled`:
   ```ts
   function canUpgrade(product: ProductToDisplay): boolean {
     return !!product.offerId && (upgradePaths[product.id]?.targets.length ?? 0) > 0;
   }
   ```
5. **New button** in `.buttonRow`, next to "Add licenses":
   ```tsx
   <Button
     variant="primary"
     fillStyle={'outline'}
     onPress={() => setUpgradeDialogProduct(product)}
     isDisabled={!canUpgrade(product)}
   >
     Upgrade
   </Button>
   ```
6. **New handler**, `handleProceedToUpgrade`:
   ```ts
   const handleProceedToUpgrade = (target: {
     targetOfferId: string;
     switchType: SwitchType;
     targetProductName: string;
   }) => {
     if (!upgradeDialogProduct || !customerId || !resellerId) return;
     router.push({
       pathname: '/upgradeOrderSummary',
       query: {
         customerId,
         customerName: customerName || '',
         resellerId,
         sourceOfferId: upgradeDialogProduct.offerId || '',
         sourceSubscriptionId: upgradeDialogProduct.id,
         sourceCurrencyCode: upgradeDialogProduct.currencyCode,
         sourceQuantity: String(upgradeDialogProduct.currentQuantity || 0),
         sourceProductName: upgradeDialogProduct.name,
         targetOfferId: target.targetOfferId,
         targetProductName: target.targetProductName,
         switchType: target.switchType,
       },
     });
     setUpgradeDialogProduct(null);
   };
   ```
7. **Render** (once, outside the `products.map`, at the bottom of the component alongside where a dialog would sit):
   ```tsx
   <UpgradePathsDialog
     isOpen={!!upgradeDialogProduct}
     onClose={() => setUpgradeDialogProduct(null)}
     onProceed={handleProceedToUpgrade}
     sourceProductName={upgradeDialogProduct?.name || ''}
     sourceOfferId={upgradeDialogProduct?.offerId || ''}
     currencyCode={upgradeDialogProduct?.currencyCode || ''}
     targets={upgradeDialogProduct ? upgradePaths[upgradeDialogProduct.id]?.targets ?? [] : []}
   />
   ```

**Why this component and not `CustomerProductsOverviewPanel.tsx`:** the Upgrade flow is scoped to a single
subscription card, the same scope as the existing "Add licenses" action (which is also self-contained in
`ActiveProducts.tsx`, calling `useCart()` directly rather than lifting to the parent). `ActiveProducts`
already receives every prop this feature needs (`customerId`, `customerName`, `resellerId`, and each
product's `id`/`offerId`/`currencyCode`/`currentQuantity`/`name`) — no new prop drilling through
`CustomerProductsOverviewPanel` is required, unlike the Late Renewal / Edit Renewal dialogs which operate
across *all* subscriptions at once and are correctly owned by the parent.

**Loading/error state:** no visible loading state for the discovery call — per the experience card, the
"Upgrade" button simply stays disabled (`canUpgrade` returns `false` while `isLoading` is `true`, since
`targets` defaults to `[]` before the query resolves) — indistinguishable from "no upgrade paths," which is
the exact silent-failure behavior specified.

---

## §4 — Acceptance Criteria Coverage

| Goal (from experience card) | Covered by |
|---|---|
| Partners can tell at a glance whether a subscription has an available upgrade, without leaving the Customer Details page. | §3 `hooks/useOfferSwitchPaths.ts`, `components/customerdetails/ActiveProducts.tsx` (MODIFY) |
| Partners see an accurate price preview — including prorated amount — before committing to the switch order. | §3 `utils/upgradeOrderUtils.ts` (`fetchUpgradeOrderPreview`), `pages/upgradeOrderSummary.tsx` (preview state), `components/upgrade/UpgradeOrderSummaryCard.tsx` (per-license price + "(N days proration)" caption + estimated total display) — both the price and the `proratedDays` count come directly from the backend's `pricing`/`proratedDays` fields per the backend LLD; this LLD only displays what the response returns, no client-side proration math |
| Partners get clear feedback after submitting an upgrade order, **and cannot submit without write permission on the org**. | feedback (success navigation, fixed error message) is covered by `pages/upgradeOrderSummary.tsx` + `components/upgrade/UpgradeOrderSummaryCard.tsx` §3.

---

## §5 — Out of Scope

- **`PREVIEW_REVERT_SWITCH` / `REVERT_SWITCH` UI.** The experience card explicitly states this app's UI does
  not implement reverting a switch order today; no revert entry point, dialog, or page is included here,
  matching the backend LLD's own §7.
- **Sorting the Upgrade Paths Dialog's target list by the API's `sequence` field.** The experience card
  explicitly says `sequence` exists but is not used for ordering — targets render in API response order.
