# LLD — Early Renewal — UI

**Design source:** No Figma URL in the experience card (`figma-url:` field is empty) — this LLD is
derived entirely from the experience card description
(`feature-specs/early-renewal/experience-card/early-renewal.md`). All visual spacing/color values
are therefore approximate — follow the existing `LateRenewalOrderDialog` stylesheet's values as the
concrete reference (see §3) rather than inventing new ones.

**Backend LLD consumed:** `docs/ai-kit/LLD/backend/early-renewal-apisepc-lld.md` — confirms
`POST /api/orders?type=PreviewRenewal` and `POST /api/orders?type=RenewalOrder` are already fully
implemented for both the Customer Details dialog and the Catalog's new-product entry point
(Operation 4 — same two routes, `subscriptionId` omitted on new-product lines), and prescribes
adding `renewedQuantity?: number` to `models/Subscription.ts`. **This UI LLD depends on that backend
LLD's `models/Subscription.ts` change being applied** — every early-renewal UI behavior below
(window-rollover check, segmentation, badge, quantity clamp, new-products-only guard) reads
`subscription.renewedQuantity`, which does not exist as a typed field until that change lands. It is
not re-specified here since it is the same physical file, already fully scoped in that document.

---

## §1 — Summary

Early Renewal lets a partner manually renew a customer's subscriptions before the Anniversary Date
(AD) — either from the Customer Details page (renewing products the customer already owns) or from
the Catalog (adding a brand-new product the customer doesn't yet own). The feature adds three pieces:

1. **Early Renew entry point** — an `ActionButton` in `CustomerProductsOverviewPanel`'s Anniversary
   Date row, shown only when the customer is inside its early-renewal window (Customer Details).
2. **Early Renewal Order Dialog** — a new dialog where the partner reviews eligible subscriptions
   split into an **Existing** segment (renew up to current quantity) and an **Additional** segment
   (extra seats, unbounded quantity), previews pricing, optionally applies a discount code per
   line, and submits an actual `RENEWAL` order (Customer Details).
3. **"At Early Renewal" Catalog entry point** — a third `LICENSE_AVAILABILITY` option in the
   Catalog's existing "Add customer details" step, routing to a dedicated `/checkout/renewal`
   page when the partner is adding a product the customer doesn't already own while that customer
   is inside its early-renewal window (Catalog).

This repo already has a **structurally near-identical sibling feature** — late (post-AD, manual
renewal window) renewal, implemented as `RenewalWindowClosesBanner` → `LateRenewalOrderDialog`
(`components/renewal/lateRenewal/`) backed by `utils/renewalOrderUtils.ts`. That module's network
functions (`fetchRenewalOrderPreview`, `placeRenewalOrder`) and data-shaping helpers
(`prepareRenewalData`, `renewalItemsToLineItems`, `initializeRenewalStates`,
`calculateTotalLicenses`) are generic over "a list of subscriptions with quantities and per-line
renew toggles" — they contain no late-renewal-specific business rule — so this LLD **reuses them
directly** rather than forking, for both the Customer Details dialog and the Catalog checkout page.
Only the genuinely new pieces (AD-window eligibility, the renewedQuantity-based rollover check,
Existing/Additional segmentation, and the Catalog's new-products-only guard) are new code, isolated
in a new `utils/earlyRenewalUtils.ts`, a new `EarlyRenewalOrderDialog` component that otherwise
mirrors `LateRenewalOrderDialog`'s structure closely, and a new `pages/checkout/renewal.tsx` page
that mirrors `pages/checkout.tsx`.

**Note on how the Catalog surface was derived:** a structurally similar flow already exists in a
separate, full-featured sibling app (an "At Early Renewal" license-availability option, routed to
a dedicated early-renewal checkout page). That surface of this LLD was **derived independently**
against *this* reference app's actual current architecture — its own `FindOrCreateCustomer` /
`ExistingCustomer` / `checkout.tsx` files, its own `LICENSE_AVAILABILITY` constant shape, and its
own (simpler, single-mutation-hook) checkout pattern — rather than transcribing the sibling's file
structure. The **domain rules** (new-products-only guard, already-in-progress guard, prorated
early-renewal pricing, AD-rollover messaging) match the sibling's confirmed behavior; the
**implementation shape** below is this repo's own.

**User journey — Customer Details:** partner opens a customer's detail page → (if in window) sees
an "Early renew" button next to "Edit renewal order" → clicks it → dialog loads eligible
subscriptions, splits them into Existing/Additional, and fetches a price preview for the active
segment → partner adjusts quantities/toggles/discount codes → partner clicks "Renew now" → order is
placed → dialog closes, a success toast shows the order ID, and the subscriptions query is
invalidated so the Products list reflects the new `renewedQuantity` without a manual page refresh.

**User journey — Catalog:** partner adds a product the customer doesn't already own to the cart →
opens "Add customer details" → selects an existing customer → if that customer is inside its
early-renewal window and doesn't already own any cart item, selects "At Early Renewal" →
continues to a dedicated early-renewal checkout page → reviews prorated pricing and (on a customer's
first early renewal this cycle) AD-rollover/post-order-restriction messaging → places the order →
redirected to the existing order confirmation page.

---

## §2 — Data Flow

### Flow 1 — Entry-point eligibility (Customer Details page render)

```
Trigger: CustomerProductsOverviewPanel renders (subscriptions + customer already loaded by the
         parent page's existing customerDetailKeys.customer / customerDetailKeys.subscriptions
         queries — no new fetch)

  → isInEarlyRenewalWindow(anniversaryDate, subscriptions)   [utils/earlyRenewalUtils.ts, NEW]
      - true if daysUntilAnniversary(anniversaryDate) is in [1, 30]
      - OR true if any subscription has renewedQuantity != null && renewedQuantity >= 0
  → CustomerProductsOverviewPanel renders an "Early renew" ActionButton next to
    "Edit renewal order" inside the existing anniversary-date row, only when the check is true
    AND hasManualRenewal is false (the existing ternary already hides this whole row when
    hasManualRenewal is true — Early Renew inherits that precedence for free, no new logic needed)
  ← no network call — pure client-side derivation over already-loaded data
```

### Flow 2 — Dialog opens (partner clicks "Early renew")

```
Trigger: partner clicks "Early renew"

  → setIsEarlyRenewalDialogOpen(true)   [CustomerProductsOverviewPanel.tsx]
  → EarlyRenewalOrderDialog mounts open, receives `subscriptions` as a prop (already-loaded data,
    same pattern as LateRenewalOrderDialog/EditRenewalDialog — no dialog-owned fetch)
  → computeSubscriptionsEligibleForRenewalUpdate(subscriptions)   [utils/customerDetailsUtils.ts,
    REUSED — status 1000/1009 filter]
  → splitSubscriptionsByRenewalProgress(eligibleSubscriptions)   [utils/earlyRenewalUtils.ts, NEW]
      → { existing: [...], additional: [...] }  (renewedQuantity < currentQuantity → existing;
         else → additional)
  → active segment defaults to 'EXISTING', or 'ADDITIONAL' if existing is empty
  → initializeExistingQuantities(existingSubs)   [utils/earlyRenewalUtils.ts, NEW — defaults each
    quantity to getRemainingExistingQuantity(sub), i.e. currentQuantity - renewedQuantity] for the
    Existing segment
  → initializeAdditionalQuantities(additionalSubs)   [utils/earlyRenewalUtils.ts, NEW — defaults
    every quantity to 0, since "additional" means extra seats beyond what's already owned]
  → initializeRenewalStates(existingSubs) → all `true`   [utils/renewalOrderUtils.ts, REUSED]
  → initializeAdditionalRenewalStates(additionalSubs) → all `false`   [utils/earlyRenewalUtils.ts,
    NEW — nothing is added by default]
  → fetchRenewalOrderPreview(customerId, lineItems, currency)   [utils/renewalOrderUtils.ts,
    REUSED — POST /api/orders?type=PreviewRenewal, orderType: "PREVIEW_RENEWAL", with the active
    segment's lineItems]
      → backend: see backend LLD Operation 1 (already implemented, unchanged)
  ← response: { lineItems[].pricing, lineItems[].proratedDays, pricingSummary[] }
  ← updateSubscriptionsWithPreview(...)   [utils/renewalOrderUtils.ts, REUSED] maps pricing back
    onto localSubscriptionState
  ← dialog renders per-subscription cards for the active segment with price, badge, quantity field
  Error path: preview fetch throws → showError(message) → ErrorToast (per-component, not global)
```

### Flow 3 — Submit ("Renew now")

```
Trigger: partner clicks "Renew now" (enabled once total selected quantity in the active segment > 0)

  → prepareRenewalData(activeSegmentSubs, renewalStates, quantities)   [utils/renewalOrderUtils.ts,
    REUSED] → RenewalDataToSubmit[] (only lines with their renew toggle on)
  → renewalItemsToLineItems(renewalData)   [utils/renewalOrderUtils.ts, REUSED] → LineItem[]
  → generateExternalReferenceId()   [utils/commonUtils.ts, REUSED]
  → placeRenewalOrder(customerId, externalReferenceId, currencyCode, lineItems)
    [utils/renewalOrderUtils.ts, REUSED] → POST /api/orders?type=RenewalOrder,
    body { customerId, externalReferenceId, currencyCode, lineItems, orderType: "RENEWAL" }
      → backend: see backend LLD Operation 2 (already implemented, unchanged)
  ← response: { orderId, status, lineItems[], links }
  ← on success:
      - dialog closes
      - queryClient.invalidateQueries({ queryKey: customerDetailKeys.subscriptions(customerId) })
        [NEW for this dialog — see §4 Design Decisions]
      - onSuccess(orderId) bubbles to CustomerProductsOverviewPanel.handleRenewalSuccess (REUSED,
        unchanged) → SuccessToast: "Renewal order submitted successfully! Order ID: {orderId}"
  ← on failure: onError(message) bubbles to handleRenewalError (REUSED, unchanged) → ErrorToast
    with the backend's message field surfaced verbatim (no per-code translation — see backend LLD
    §5 and experience card's Error Handling section)
```

### Flow 4 — "Make licenses available" eligibility (existing-customer step of Find/Create Customer)

```
Trigger: partner has a product in the cart, opens "Add customer details", stays on the
         "Find an existing customer" tab, and selects a reseller + customer in ExistingCustomer.tsx

  → fetchCustomerDetails(customerId)   [utils/AccountApis.ts, REUSED — already called here today]
  → fetchCustomerSubscriptions(customerId)   [utils/customerDetailsUtils.ts, REUSED — NEW call site;
    not fetched in this component today]
  → eligibleSubscriptions = computeSubscriptionsEligibleForRenewalUpdate(subscriptions)
    [utils/customerDetailsUtils.ts, REUSED]
  → isInEarlyRenewalWindow(customerDetails.cotermDate, eligibleSubscriptions)
    [utils/earlyRenewalUtils.ts, REUSED — same 1–30-day / rolled-over check as Flow 1]
  → customerOwnsAnyCartOffer(eligibleSubscriptions, Object.keys(cartItemIdToQuantityMap))
    [utils/earlyRenewalUtils.ts, NEW — true if any cart offer's SKU matches a SKU the customer
    already has a subscription for]
  → hasEarlyRenewalOrderPlaced(eligibleSubscriptions)   [utils/earlyRenewalUtils.ts, NEW — the same
    renewedQuantity-populated predicate `isInEarlyRenewalWindow` uses internally, exported on its
    own so this flow can ask it independently — see §4]
  ← "At Early Renewal" radio is disabled when NOT isInEarlyRenewalWindow OR customerOwnsAnyCartOffer
  ← "Now" radio is disabled when hasEarlyRenewalOrderPlaced (an early renewal is already in progress —
    only renewal-type orders are accepted for this contract until the term ends)
  ← no network call beyond the two reads above — this is otherwise a pure client-side derivation,
    same posture as Flow 1 above
```

### Flow 5 — Continue with "At Early Renewal" selected

```
Trigger: partner selects the "At Early Renewal" radio and clicks "Continue" in
         FindOrCreateCustomer.tsx

  → setCustomerInfoInCart({ customerId, customerName, resellerId, makeAvailable: 'earlyRenewal' })
    [contexts/CartContext.tsx, REUSED — same call already made for 'now' / 'atRenewal']
  → onClose() (closes the Find/Create Customer overlay)
  → router.push('/checkout/renewal')   [NEW route, distinct from the plain 'now' path's
    router.push('/checkout')]
  ← no network call — routing only
```

### Flow 6 — Early renewal checkout page loads

```
Trigger: pages/checkout/renewal.tsx mounts (customerInfoInCart already populated from Flow 5)

  → fetchCustomerDetails(customerInfoInCart.customerId)   [utils/AccountApis.ts, REUSED]
  → fetchCustomerSubscriptions(customerInfoInCart.customerId)   [utils/customerDetailsUtils.ts, REUSED]
  → hasEarlyRenewalOrderPlaced(computeSubscriptionsEligibleForRenewalUpdate(subscriptions))
    [utils/earlyRenewalUtils.ts, REUSED from Flow 4 — determines whether THIS order will be the one
    that rolls the customer's AD forward, for the rollover note / post-order banner]
  → cartItemsToLineItems(items)   [utils/cartUtils.ts, REUSED verbatim — already produces line items
    with no subscriptionId, which is exactly the shape a new-product line needs]
  → fetchRenewalOrderPreview(customerId, lineItems, currencyCode)   [utils/renewalOrderUtils.ts,
    REUSED from Flow 2 — POST /api/orders?type=PreviewRenewal, orderType: "PREVIEW_RENEWAL"]
      → backend: see backend LLD Operation 4 (unchanged)
  ← response: { lineItems[].pricing, lineItems[].proratedDays, pricingSummary[] }
  ← updateCartItemsWithSkuMapping(items, previewData.lineItems)   [utils/cartUtils.ts, REUSED verbatim]
  ← prorationDays = max(...previewData.lineItems.map(li => li.proratedDays ?? 0))
  ← page renders CheckoutReviewDetails (title "Review Details - Early Renewal", extra bullet:
    "This early renewal covers {prorationDays} days" [+ ", and moves the anniversary date to {AD}"
    when !hasEarlyRenewalOrderPlaced]) and CheckoutSummary (title "Early Renewal Summary", banner:
    "After this order, only renewal orders can be placed for this customer until the current term
    ends on {AD}." when !hasEarlyRenewalOrderPlaced)
  Error path: preview fetch throws → same ErrorToast pattern as /checkout today
```

### Flow 7 — Submit ("Place order" on the early-renewal checkout)

```
Trigger: partner clicks "Place order" on pages/checkout/renewal.tsx

  → placeRenewalOrder(customerId, externalReferenceId, currencyCode, lineItems)
    [utils/renewalOrderUtils.ts, REUSED from Flow 3 — POST /api/orders?type=RenewalOrder,
    orderType: "RENEWAL"]
      → backend: see backend LLD Operation 4 (unchanged)
  ← response: { orderId, status, lineItems[], links }
  ← on success: clearCart() [REUSED] → router.push('/orderConfirmation?...') with the same query-param
    shape /checkout already builds (customerId, currencyCode, totalAmount, customerName, resellerId,
    products) — REUSED verbatim, no new confirmation-page work needed
  ← on failure: same ErrorToast pattern as /checkout today, backend message surfaced verbatim (no
    per-code translation — same posture as Flows 1–3, see backend LLD §5)
```

---

## §3 — Change Summary

| File | Action | Layer | Reason |
|------|--------|-------|--------|
| `utils/earlyRenewalUtils.ts` | CREATE | Util | AD-window eligibility check (`isInEarlyRenewalWindow`, `hasEarlyRenewalOrderPlaced`), renewedQuantity-based rollover check, Existing/Additional segmentation, Existing-segment and Additional-segment quantity/toggle initializers, deadline formatting, and the Catalog's new-products-only guard (`customerOwnsAnyCartOffer`) — none of this exists today |
| `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` | CREATE | Component | New dialog for placing an early renewal order — no equivalent exists; structurally mirrors `LateRenewalOrderDialog.tsx` but adds segmentation, badges, and per-segment quantity rules |
| `styles/earlyRenewal/EarlyRenewalOrderDialog.module.css` | CREATE | Style | Stylesheet for the new dialog, mirroring `styles/lateRenewal/LateRenewalOrderDialog.module.css`'s layout classes plus new classes for the segmented control, badge, and new-product note |
| `components/customerdetails/CustomerProductsOverviewPanel.tsx` | MODIFY | Component | Add the "Early renew" entry-point button and dialog-open state, mount the new dialog — same pattern already used for `isEditRenewalDialogOpen`/`isLateRenewalDialogOpen` |
| `utils/constants.ts` | MODIFY | Constant | `LICENSE_AVAILABILITY` needs a third value, `EARLY_RENEWAL: 'earlyRenewal'`, alongside the existing `NOW` / `AT_RENEWAL` |
| `components/catalogCart/customerdetails/ExistingCustomer.tsx` | MODIFY | Component | Add the "At Early Renewal" radio, fetch the selected customer's subscriptions (not fetched here today), compute the two eligibility guards, disable/auto-switch radios accordingly, show helper text |
| `components/catalogCart/customerdetails/ExistingCustomer.module.css` | MODIFY | Style | Add a helper-text class for the disabled-radio explanation, following this file's existing `.radioOption` / `.radioLabel` naming |
| `components/catalogCart/FindOrCreateCustomer.tsx` | MODIFY | Component | Route to `/checkout/renewal` instead of `/checkout` when `makeAvailable === LICENSE_AVAILABILITY.EARLY_RENEWAL` |
| `components/checkout/CheckoutReviewDetails.tsx` | MODIFY | Component | Add optional `title` and `extraBullets` props (both default to today's behavior) so the early-renewal checkout can relabel the panel and append the proration / AD-rollover bullets without forking the component |
| `components/checkout/CheckoutSummary.tsx` | MODIFY | Component | Add optional `title`, `placeOrderButtonLabel`, and `banner` props (all default to today's behavior) so the early-renewal checkout can relabel the summary and show the post-order restriction banner |
| `components/checkout/Checkout.module.css` | MODIFY | Style | Add one new class for the post-order restriction banner |
| `pages/checkout/renewal.tsx` | CREATE | Page | Dedicated early-renewal checkout page — structurally mirrors `pages/checkout.tsx` but previews/places a `RENEWAL` order instead of a `NEW` order, and computes the proration/rollover copy |

One new route is added — `/checkout/renewal` — for the Catalog's early-renewal checkout; the
Customer Details pieces add no new routes, living entirely within the existing `/customerdetails`
page. No new data-fetch unit is introduced beyond what's reused from `utils/renewalOrderUtils.ts` —
see §4 for why reuse was chosen over forking a parallel `fetchEarlyRenewalOrderPreview`/
`placeEarlyRenewalOrder`.

---

### `utils/earlyRenewalUtils.ts` — CREATE

```ts
import { SubscriptionToRenew } from '../types/lateRenewal';
import { getSKUFromOfferId } from './commonUtils';

/** Days from today (UTC midnight) until the given anniversary date. Negative if already past. */
export const getDaysUntilAnniversary = (anniversaryDate?: string): number | null => {
  if (!anniversaryDate) return null;
  const ad = new Date(anniversaryDate);
  if (isNaN(ad.getTime())) return null;
  const today = new Date();
  const utcToday = Date.UTC(today.getFullYear(), today.getMonth(), today.getDate());
  const utcAd = Date.UTC(ad.getFullYear(), ad.getMonth(), ad.getDate());
  return Math.round((utcAd - utcToday) / (1000 * 60 * 60 * 24));
};

/**
 * True if at least one subscription already has a populated renewedQuantity (>= 0) — meaning an
 * early renewal was already placed this cycle. Exported on its own (not inlined into
 * isInEarlyRenewalWindow) because the Catalog entry point needs to ask this question independently
 * of the day-window check — it drives disabling "Now", not "At Early Renewal".
 */
export const hasEarlyRenewalOrderPlaced = (subscriptions: SubscriptionToRenew[]): boolean =>
  subscriptions.some(
    sub => sub.renewedQuantity !== undefined && sub.renewedQuantity !== null && sub.renewedQuantity >= 0
  );

/**
 * Early Renew is shown when EITHER:
 *  - today is 1-30 days before the Anniversary Date, OR
 *  - hasEarlyRenewalOrderPlaced(subscriptions) — an early renewal was already placed this cycle,
 *    which is why AD has rolled forward. Checking renewedQuantity directly (rather than
 *    re-deriving a shifted day window) is simpler and does not need to track how many times AD
 *    has rolled.
 */
export const isInEarlyRenewalWindow = (
  anniversaryDate: string | undefined,
  subscriptions: SubscriptionToRenew[]
): boolean => {
  const days = getDaysUntilAnniversary(anniversaryDate);
  const withinWindow = days !== null && days >= 1 && days <= 30;
  return withinWindow || hasEarlyRenewalOrderPlaced(subscriptions);
};

/** "Submit early renewal before {date}" deadline — the Anniversary Date itself, formatted. */
export const formatEarlyRenewalDeadline = (anniversaryDate?: string): string => {
  if (!anniversaryDate) return '';
  const ad = new Date(anniversaryDate);
  if (isNaN(ad.getTime())) return '';
  return ad.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
};

/**
 * Splits already-eligible (status 1000/1009) subscriptions into Existing (still has quantity
 * left to renew) vs Additional (fully renewed — only extra seats/new products left to add).
 */
export const splitSubscriptionsByRenewalProgress = (
  subscriptions: SubscriptionToRenew[]
): { existing: SubscriptionToRenew[]; additional: SubscriptionToRenew[] } => {
  const existing: SubscriptionToRenew[] = [];
  const additional: SubscriptionToRenew[] = [];
  subscriptions.forEach(sub => {
    const current = sub.currentQuantity ?? 0;
    const renewed = sub.renewedQuantity ?? 0;
    (renewed < current ? existing : additional).push(sub);
  });
  return { existing, additional };
};

/** Remaining renewable quantity for the Existing segment's per-line quantity clamp. */
export const getRemainingExistingQuantity = (sub: SubscriptionToRenew): number =>
  Math.max(0, (sub.currentQuantity ?? 0) - (sub.renewedQuantity ?? 0));

/**
 * Existing-segment quantities default to the remaining renewable amount
 * (currentQuantity - renewedQuantity), not the full currentQuantity — a subscription that has
 * already had some quantity early-renewed this cycle can only renew what's left.
 */
export const initializeExistingQuantities = (
  subscriptions: SubscriptionToRenew[]
): { [key: string]: number } => {
  const quantities: { [key: string]: number } = {};
  subscriptions.forEach(sub => {
    if (sub.subscriptionId) quantities[sub.subscriptionId] = getRemainingExistingQuantity(sub);
  });
  return quantities;
};

/**
 * Additional-segment quantities default to 1 — the minimum valid "additional" quantity is 1 seat
 * (0 extra seats is meaningless); whether a product is added at all is controlled by the
 * per-line renew toggle (see initializeAdditionalRenewalStates), not by this quantity.
 */
export const initializeAdditionalQuantities = (
  subscriptions: SubscriptionToRenew[]
): { [key: string]: number } => {
  const quantities: { [key: string]: number } = {};
  subscriptions.forEach(sub => {
    if (sub.subscriptionId) quantities[sub.subscriptionId] = 1;
  });
  return quantities;
};

/** Additional-segment renew toggles default OFF — nothing is added unless the partner opts in. */
export const initializeAdditionalRenewalStates = (
  subscriptions: SubscriptionToRenew[]
): { [key: string]: boolean } => {
  const states: { [key: string]: boolean } = {};
  subscriptions.forEach(sub => {
    if (sub.subscriptionId) states[sub.subscriptionId] = false;
  });
  return states;
};

/**
 * True if any offer currently in the cart matches (by SKU — first 8 chars of the offer ID, same
 * comparison already used by CartContext.updateCartItemsFromAPI and
 * AddAndEditCustomerRenewalOrderDialog) a subscription the customer already holds. Early renewal
 * from the Catalog is new-products-only — if the cart contains something the customer already
 * owns, that product must be renewed/added-to from the Customer Details dialog instead.
 */
export const customerOwnsAnyCartOffer = (
  subscriptions: SubscriptionToRenew[],
  cartOfferIds: string[]
): boolean => {
  const ownedSKUs = new Set(
    subscriptions.map(sub => sub.offerId).filter((id): id is string => !!id).map(getSKUFromOfferId)
  );
  return cartOfferIds.some(offerId => ownedSKUs.has(getSKUFromOfferId(offerId)));
};
```

- Imports `SubscriptionToRenew` from the existing `types/lateRenewal.ts` — no new type is defined;
  once the backend LLD's `models/Subscription.ts` change adds `renewedQuantity?: number` to
  `Subscription`, it flows through automatically since `SubscriptionToRenew extends Subscription`.
- Imports `getSKUFromOfferId` from `utils/commonUtils.ts` (REUSED) — the same SKU-truncation helper
  already used for cart/subscription matching elsewhere in this repo (`CartContext`,
  `AddAndEditCustomerRenewalOrderDialog`).
- `computeSubscriptionsEligibleForRenewalUpdate` (status 1000/1009 filter) is **not** redefined
  here — it's imported from `utils/customerDetailsUtils.ts` at each call site (the dialog, and
  `ExistingCustomer.tsx` / `pages/checkout/renewal.tsx`).

---

### `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` — CREATE

**Props:**

| Prop | Type | Required | Notes |
|------|------|----------|-------|
| `isOpen` | `boolean` | Yes | — |
| `onClose` | `() => void` | Yes | — |
| `onSuccess` | `(orderId: string) => void` | Yes | Same contract as `LateRenewalOrderDialog` |
| `onError` | `(errorMessage: string) => void` | Yes | Same contract as `LateRenewalOrderDialog` |
| `subscriptions` | `SubscriptionToRenew[]` | Yes | All of the customer's subscriptions, as already passed into `CustomerProductsOverviewPanel` |
| `customerId` | `string \| undefined` | Yes | — |
| `anniversaryDate` | `string \| undefined` | Yes | Sourced the same way `EditRenewalDialog`/`LateRenewalOrderDialog`'s callers already have it available, from `customer?.cotermDate` — used for the deadline banner and passed through to `RenewalDialogHeader` |

**State variables:**

| Variable | Type | Default | Triggers re-fetch? |
|----------|------|---------|---------------------|
| `activeSegment` | `'EXISTING' \| 'ADDITIONAL'` | `'EXISTING'`, or `'ADDITIONAL'` if the Existing list is empty on open | Yes — segment switch re-runs the preview for the newly active segment |
| `existingSubs` / `additionalSubs` | `SubscriptionToRenew[]` | computed via `splitSubscriptionsByRenewalProgress` on open-transition | No — recomputed only on open |
| `localSubscriptionState` | `SubscriptionToRenew[]` | the active segment's subs (with price fields attached after preview) | No — updated in place after preview |
| `quantities` | `{ [subscriptionId]: number }` | `initializeExistingQuantities` (Existing) / `initializeAdditionalQuantities` (Additional) | No — local edit |
| `renewalStates` | `{ [subscriptionId]: boolean }` | `initializeRenewalStates` (Existing, all `true`) / `initializeAdditionalRenewalStates` (Additional, all `false`) | No — local edit |
| `isLoadingPrices`, `hasInitialPreview`, `isRenewing`, `discountLevelsMap`, `needsRecalculation`, `pricingSummaries`, `toastState` | (same types as `LateRenewalOrderDialog`) | (same defaults) | same triggers as `LateRenewalOrderDialog` |

**Derived values:**

- `eligibleSubscriptions = computeSubscriptionsEligibleForRenewalUpdate(subscriptions)`
- `{ existing, additional } = splitSubscriptionsByRenewalProgress(eligibleSubscriptions)`
- `totalLicenses = calculateTotalLicenses(localSubscriptionState, quantities, renewalStates)` (reused, unchanged)
- `earlyRenewalDeadline = formatEarlyRenewalDeadline(anniversaryDate)`

**Behaviour:**

- On open-transition (`isOpen && !prevIsOpenRef.current`, same ref-tracked pattern as
  `LateRenewalOrderDialog`): recompute `existing`/`additional`, set `activeSegment` (default
  `'EXISTING'`, `'ADDITIONAL'` if `existing.length === 0`), initialize `localSubscriptionState`,
  `quantities`, `renewalStates` for the active segment, clear `pricingSummaries`/`discountLevelsMap`,
  clear toast state.
- If `eligibleSubscriptions.length === 0`: render *"No active subscriptions available for early
  renewal"* instead of the segmented control/product list (matches the experience card's exact
  copy, no trailing period).
- Segmented control: use `Tabs`/`TabList`/`Tab`/`TabPanel` from `@react-spectrum/s2` (existing
  design-system pattern already used for the Customer Detail Hub's own top-level tabs — see
  `UI_MODULE_INDEX.md` 1.3) with two `Tab`s, "Existing" and "Additional". A tab is `isDisabled` when
  its segment's subscription list is empty. Below the tab list, static descriptive text per segment:
  - Existing — *"Renew current licences at the annual price"*
  - Additional — *"Renew extra seats beyond current quantity at prorated price"*
- Switching `activeSegment` re-initializes `localSubscriptionState`/`quantities`/`renewalStates` for
  the newly active segment (from the already-computed `existing`/`additional` arrays — no re-fetch
  of subscriptions, only a fresh preview call) and clears `pricingSummaries`.
- Auto-preview on open and on segment switch: same `useEffect` pattern as `LateRenewalOrderDialog`
  (`updatePricesFromPreview` on open once, and again whenever `activeSegment` changes), calling
  `fetchRenewalOrderPreview(customerId, lineItems, currency)` with the active segment's
  `lineItems` built via `renewalItemsToLineItems(prepareRenewalData(...))` — **reused verbatim**.
- Deadline banner: `{earlyRenewalDeadline && <div className={styles.deadline}>Submit early renewal before {earlyRenewalDeadline}</div>}`.
- License/discount-level summary + "Update price" button: reuse the `discountLevelsMap` rendering
  block and `UpdatePriceButton` component as-is. This dialog
  must wire it as `onPress={() => updatePricesFromPreview()}` (an arrow-function wrapper, not the
  bare function reference) so the button always triggers a fresh preview for the current
  `localSubscriptionState` with no arguments forwarded.
- Per-subscription card (one per subscription in the active segment's `localSubscriptionState`):
  - Product name via `useProductNamesFromPricelist` (reused hook)
  - Renewed-quantity badge — rendered only when `(sub.renewedQuantity ?? 0) > 0`:
    - Existing tab: `{sub.renewedQuantity} of {sub.currentQuantity ?? 0} renewed`
    - Additional tab: `{sub.renewedQuantity} renewed`
  - `Switch` — "Renew" / "Do not renew" toggle (`renewalStates`)
  - `NumberField` quantity — Existing: `minValue={1}`, `maxValue={getRemainingExistingQuantity(sub)}`; Additional: `minValue={1}`, no `maxValue` (unbounded) — 0 is not a valid "additional" quantity, so both segments share the same floor
  - Computed per-license price + line total (from `sub.pricePerLicense`/`sub.lineItemPartnerPrice`, same fields `updateSubscriptionsWithPreview` populates)
  - Offer ID box
  - `PromoCodeButton` (reused as-is — `offerId`, `productName`, `appliedCode`, `onApply`, `onRemove`, `isLoading`)
- New-product note (rendered once, below the product list, not per-card): *"Adding a new product?
  New products are added from the Catalog as a separate early-renewal order — they can't be added
  here."*
- `EstimatedTotal` per currency from `pricingSummaries`, same rendering as `LateRenewalOrderDialog`.
- `RenewalDialogHeader`: `title="Early renew"`, `anniversaryDate`, `onCancel={onClose}`,
  `onSave={handleRenewNow}`, `isSaving={isRenewing}`, `isSaveDisabled={totalLicenses === 0}`,
  `saveButtonLabel="Renew now"` — reused with no new props needed.
- `handleRenewNow`: same shape as `LateRenewalOrderDialog.handleRenewNow`, calling
  `placeRenewalOrder(customerId, externalReferenceId, currencyCode, lineItems)`, **plus** on
  success: `queryClient.invalidateQueries({ queryKey: customerDetailKeys.subscriptions(customerId) })`
  before calling `onSuccess(orderId)` — see §4 for why this differs from `LateRenewalOrderDialog`.
- Loading overlay while `isRenewing`: *"Submitting early renewal order"* (same pattern/markup as
  `LateRenewalOrderDialog`'s `styles.loadingOverlay`, different copy).
- `ErrorToast` for preview failures (per-component `useToastState`, same as `LateRenewalOrderDialog`).
- Accessibility: dialog uses the same overlay/backdrop pattern as `LateRenewalOrderDialog`
  (`createPortal`-free plain fixed-position `div`s here — matches existing sibling exactly, not the
  `role="dialog"`/`aria-modal` pattern documented for `PromoCodeDialog`-style dialogs in
  `UI_CODE_PATTERNS.md` B.6; this LLD does not change that inconsistency, it only mirrors the
  closer/more relevant sibling for this feature).

---

### `styles/earlyRenewal/EarlyRenewalOrderDialog.module.css` — CREATE

Reuse the layout classes from `styles/lateRenewal/LateRenewalOrderDialog.module.css` verbatim
(`.backdrop`, `.dialog`, `.content`, `.deadline`, `.licensesInfo`, `.licensesCount`, `.productList`,
`.productCard`, `.productHeader`, `.productInfo`, `.productDetails`, `.productName`, `.renewToggle`,
`.productBody`, `.quantityControls`, `.rightSection`, `.totalPrice`, `.pricePerLicense`,
`.offerIdBox`, `.promoCodeButtonWrapper`, `.loadingOverlay`, `.loadingContent`, `.spinner`,
`.loadingText` — ⚠ NOT CAPTURED exact pixel/color values, no Figma reference; copy the sibling
file's values as-is). New classes needed, approximate intent only:

```css
.segmentedControl {
  /* wraps the Tabs/TabList — margin-bottom to separate from description text below */
}
.segmentDescription {
  /* small, muted helper text below the tab list, one line per active segment */
}
.renewedBadge {
  /* small pill/chip badge next to product name — neutral/positive tone, not a Spectrum Badge
     component per the experience card's plain-text framing, but a Spectrum Badge (fillStyle: bold,
     variant: neutral) is an acceptable substitute per existing design-system conventions */
}
.newProductNote {
  /* muted helper text block below the product list, same tone as .pricePerLicense */
}
```

---

### `components/customerdetails/CustomerProductsOverviewPanel.tsx` — MODIFY

Current relevant behaviour (read in full — see the panel's existing ternary at the top of its
render): the Anniversary Date row (with the "Edit renewal order" button) only renders in the
`else` branch of `hasManualRenewal && !renewalOrderSubmitted ? <RenewalWindowClosesBanner /> : (...)`.

Changes:

1. Add state: `const [isEarlyRenewalDialogOpen, setIsEarlyRenewalDialogOpen] = useState(false);`
   (same pattern as the existing `isEditRenewalDialogOpen`/`isLateRenewalDialogOpen`).
2. Import `isInEarlyRenewalWindow` from `../../utils/earlyRenewalUtils` and
   `EarlyRenewalOrderDialog` from `../renewal/earlyRenewal/EarlyRenewalOrderDialog`.
3. Inside the existing anniversary-date row (the `else` branch, alongside the "Edit renewal order"
   `ActionButton`), add a second conditional `ActionButton`:
   ```tsx
   {isInEarlyRenewalWindow(anniversaryDate, subscriptions) && (
     <ActionButton onPress={() => setIsEarlyRenewalDialogOpen(true)}>
       Early renew
     </ActionButton>
   )}
   ```
   This automatically inherits the existing `hasManualRenewal` precedence — the whole row (and
   therefore this button) is not rendered at all when a late-renewal banner is showing, with no new
   gating logic required. Update `.editRenewalButtonWrapper` in
   `styles/customerdetails/ProductsPanel.module.css` to `display: flex; align-items: center; gap:
   8px;` (alongside its existing `margin-right: 16px`) so the two `ActionButton`s render side by
   side, next to each other, matching the experience card's "Early renew appears next to Edit
   renewal order" placement.
4. Mount the dialog near the existing `EditRenewalDialog`/`LateRenewalOrderDialog` mounts:
   ```tsx
   <EarlyRenewalOrderDialog
     isOpen={isEarlyRenewalDialogOpen}
     onClose={() => setIsEarlyRenewalDialogOpen(false)}
     onSuccess={handleRenewalSuccess}
     onError={handleRenewalError}
     subscriptions={subscriptions}
     customerId={customerId}
     anniversaryDate={anniversaryDate}
   />
   ```
   `handleRenewalSuccess`/`handleRenewalError` are the **existing, unchanged** handlers already used
   by `LateRenewalOrderDialog` — no new toast plumbing needed at the panel level (the subscriptions
   cache invalidation happens inside the dialog itself, see §3 and §4).

No other part of this file changes — `hasManualRenewal`, the 3YC banners, `ActiveProducts`, and
`PersonalizedRecommendations` are untouched.

---

### `utils/constants.ts` — MODIFY

```ts
export const LICENSE_AVAILABILITY = {
  NOW: 'now',
  AT_RENEWAL: 'atRenewal',
  EARLY_RENEWAL: 'earlyRenewal',
} as const;
```

Purely additive — `LicenseAvailability` (the inferred union type) gains `'earlyRenewal'` as a third
member; every existing switch/comparison against `LICENSE_AVAILABILITY.NOW` / `.AT_RENEWAL` is
unaffected.

---

### `components/catalogCart/customerdetails/ExistingCustomer.tsx` — MODIFY

**New imports:** `useCart` (`contexts/CartContext.tsx`, for `cartItemIdToQuantityMap`),
`fetchCustomerSubscriptions` (`utils/customerDetailsUtils.ts`),
`computeSubscriptionsEligibleForRenewalUpdate` (`utils/customerDetailsUtils.ts`),
`isInEarlyRenewalWindow`, `customerOwnsAnyCartOffer`, `hasEarlyRenewalOrderPlaced`
(`utils/earlyRenewalUtils.ts`).

**New data fetch:** a second `useQuery` alongside the existing `customerDetails` query:

```ts
const { data: subscriptions = [] } = useQuery({
  queryKey: ['customerSubscriptions', selectedCustomer?.customerId],
  queryFn: () => fetchCustomerSubscriptions(selectedCustomer!.customerId),
  enabled: selectedCustomer !== null,
  retry: 1,
  staleTime: 5 * 1000,
});
```

Same `enabled` / `retry` / `staleTime` posture as the existing `customerDetails` query in this file —
both fire together once a customer is selected.

**New derived values:**

```ts
const eligibleSubscriptions = computeSubscriptionsEligibleForRenewalUpdate(subscriptions);
const isEarlyRenewalWindowOpen = isInEarlyRenewalWindow(
  customerDetails?.cotermDate,
  eligibleSubscriptions
);
const ownsCartOffer = customerOwnsAnyCartOffer(
  eligibleSubscriptions,
  Object.keys(cartItemIdToQuantityMap)
);
const isEarlyRenewalDisabled = !isEarlyRenewalWindowOpen || ownsCartOffer;
const isNowDisabled = hasEarlyRenewalOrderPlaced(eligibleSubscriptions);
```

Pass raw `customerDetails.cotermDate` (ISO) to `isInEarlyRenewalWindow`, not the already-formatted
`anniversaryDate` display string used elsewhere in this file — matches how Flow 1 calls it.

**Radio group changes:**

```tsx
<RadioGroup label="Make licenses available:" value={licenseAvailability} onChange={...}>
  <Radio value={LICENSE_AVAILABILITY.NOW} isDisabled={isNowDisabled}>
    Now
  </Radio>
  <Radio
    value={LICENSE_AVAILABILITY.AT_RENEWAL}
    isDisabled={isAtRenewalDisabled(anniversaryDate)}
  >
    At renewal
  </Radio>
  <Radio value={LICENSE_AVAILABILITY.EARLY_RENEWAL} isDisabled={isEarlyRenewalDisabled}>
    At Early Renewal
  </Radio>
</RadioGroup>
{isNowDisabled && (
  <div className={styles.radioHelperText}>
    An early renewal has been detected for this customer; only renewal orders can be
    placed until the current term ends.
  </div>
)}
{!isNowDisabled && isEarlyRenewalDisabled && licenseAvailability === LICENSE_AVAILABILITY.EARLY_RENEWAL && (
  <div className={styles.radioHelperText}>
    Early renewal here supports new products only. Some products in your cart is
    already on this customer's account. Renew or add seats to existing products from the customer's
    detail screen.
  </div>
)}
```

Only one helper text is ever relevant at a time given the mutual-exclusivity of the two disabled
states in practice; both literal strings are taken verbatim from the experience card.

**Auto-switch effect (new, alongside the existing "At Renewal" auto-switch effect):**

```ts
useEffect(() => {
  if (isNowDisabled && licenseAvailability === LICENSE_AVAILABILITY.NOW) {
    setLicenseAvailability(LICENSE_AVAILABILITY.EARLY_RENEWAL);
  }
}, [isNowDisabled, licenseAvailability]);

useEffect(() => {
  if (isEarlyRenewalDisabled && licenseAvailability === LICENSE_AVAILABILITY.EARLY_RENEWAL) {
    setLicenseAvailability(LICENSE_AVAILABILITY.NOW);
  }
}, [isEarlyRenewalDisabled, licenseAvailability]);
```

Same pattern as the existing `isAtRenewalDisabled` auto-switch effect in this file — reactively
steers the selection away from whichever option just became disabled, defaulting back to `NOW`
unless `NOW` itself is the disabled one (in which case the first effect takes it to
`EARLY_RENEWAL` instead). `onCustomerFullDetailsUpdate` (existing effect, unchanged) already
re-fires whenever `licenseAvailability` changes, so no additional wiring is needed to propagate the
auto-switched value up to `FindOrCreateCustomer`.

---

### `components/catalogCart/customerdetails/ExistingCustomer.module.css` — MODIFY

Add one class, following this file's existing naming convention:

```css
.radioHelperText {
  font-family: inherit;
  font-size: 12px;
  font-weight: 400;
  line-height: 16px;
  color: var(--Alias-content-neutral-subdued-default, #505050);
  margin-top: -4px;
}
```

---

### `components/catalogCart/FindOrCreateCustomer.tsx` — MODIFY

In `handleContinue`, add one branch alongside the existing `AT_RENEWAL` branch:

```ts
if (customerData.makeAvailable === LICENSE_AVAILABILITY.EARLY_RENEWAL) {
  if (onClose) {
    onClose();
  }
  router.push('/checkout/renewal');
  return;
}
```

Placed after `setCustomerInfoInCart(...)` (unchanged) and before the existing `AT_RENEWAL` check, so
the ordering of the three branches is: `EARLY_RENEWAL` → `AT_RENEWAL` → default (`NOW`, pushes
`/checkout`, unchanged). No other part of this file changes.

---

### `components/checkout/CheckoutReviewDetails.tsx` — MODIFY

**New props (both optional, both default to today's exact behavior):**

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `title` | `string` | `'Review details'` | Renders in place of the hardcoded heading text |
| `extraBullets` | `string[]` | `[]` | Rendered as additional `<li>` items, same markup/icon as the existing four bullets, appended after them |

No existing bullet text changes and no props are required at any current call site — `/checkout`
(unchanged) does not pass either prop.

---

### `components/checkout/CheckoutSummary.tsx` — MODIFY

**New props (all optional, all default to today's exact behavior):**

| Prop | Type | Default | Notes |
|------|------|---------|-------|
| `title` | `string` | `'Order Summary'` | Replaces the hardcoded `<Heading level={2}>Order Summary</Heading>` text |
| `placeOrderButtonLabel` | `string` | `'Place Order'` | Replaces the hardcoded button label |
| `banner` | `string \| undefined` | `undefined` | When present, rendered as a dismissable-free inline notice above the totals block, using the new `.postOrderBanner` class (see below) |

No existing behavior changes for `/checkout` (unchanged) — none of the three props are passed there.

---

### `components/checkout/Checkout.module.css` — MODIFY

Add one new class:

```css
.postOrderBanner {
  font-family: inherit;
  font-size: 13px;
  font-weight: 400;
  line-height: 150%;
  color: #9a6700;
  background-color: #fff8e6;
  border: 1px solid #f0dca0;
  border-radius: 6px;
  padding: 8px 12px;
  margin-bottom: 4px;
}
```

Approximate intent only — no existing warning-banner precedent in this repo to copy exact values
from; warm/amber background and text, small font, rounded corners.

---

### `pages/checkout/renewal.tsx` — CREATE

Structurally mirrors `pages/checkout.tsx` (same `Layout` / `NavigationPanel` / two-column shell,
same `useCartRouteGuard` usage, same cart-to-`items`-state mapping effect), with these differences:

| Aspect | `/checkout` (unchanged) | `/checkout/renewal` (new) |
|--------|------------------------|---------------------------|
| Preview call | `fetchOrderPreview` (`utils/cartUtils.ts`) → `type=Preview` | `fetchRenewalOrderPreview` (`utils/renewalOrderUtils.ts`, REUSED) → `type=PreviewRenewal` |
| Place-order call | `useCreateOrder()` mutation → `type=NEW` | `placeRenewalOrder` (`utils/renewalOrderUtils.ts`, REUSED), called directly inside a local async handler with local `isPlacingOrder` state (this function is a plain async call, not a `useMutation` hook, matching how `EarlyRenewalOrderDialog` already calls it above) |
| Extra data needed | none beyond cart + reseller | also fetches `fetchCustomerDetails` (for `cotermDate`) and `fetchCustomerSubscriptions` (for `hasEarlyRenewalOrderPlaced`) for the selected customer, same `useQuery` shapes already used elsewhere in this repo |
| `CheckoutReviewDetails` | default props | `title="Review Details - Early Renewal"`, `extraBullets={[...]}` (see below) |
| `CheckoutSummary` | default props | `title="Early Renewal Summary"`, `placeOrderButtonLabel="Place order"`, `banner={...}` (see below) |
| Breadcrumb last item | `{ label: 'Checkout', href: undefined }` | `{ label: 'Early renewal checkout', href: undefined }` |
| Post-success navigation | `/orderConfirmation?...` (REUSED query-param shape) | identical — REUSED verbatim |

**Deriving `extraBullets` and `banner`:**

```ts
const prorationDays = Math.max(
  0,
  ...(previewData?.lineItems?.map(li => li.proratedDays ?? 0) ?? [0])
);
const willRollOverAD = !hasEarlyRenewalOrderPlaced(
  computeSubscriptionsEligibleForRenewalUpdate(subscriptions)
);
const formattedAD = customerDetails?.cotermDate
  ? formatEarlyRenewalDeadline(customerDetails.cotermDate)  // REUSED from utils/earlyRenewalUtils.ts
  : '';

const extraBullets = prorationDays
  ? [
      `This early renewal covers ${prorationDays} days${
        willRollOverAD && formattedAD ? `, and moves the anniversary date to ${formattedAD}` : ''
      }.`,
    ]
  : [];

const banner =
  willRollOverAD && formattedAD
    ? `After this order, only renewal orders can be placed for this customer until the current term ends on ${formattedAD}.`
    : undefined;
```

**Route guard:** identical `shouldAllowNavigation` logic as `/checkout` (allow `/orderConfirmation`,
and `/customerdetails` for the same `customerId`) — `useCartRouteGuard` is reused with the same
options shape, just instantiated on this page too.

**Error handling:** same `ErrorToast` pattern as `/checkout` — preview and place-order failures both
surface `error.message` verbatim (no per-code translation, consistent with Flows 1–3).

---

## §4 — Design Decisions

| Decision | Why | Trade-off | Enforcement |
|----------|-----|-----------|-------------|
| Reuse `utils/renewalOrderUtils.ts`'s `fetchRenewalOrderPreview` and `placeRenewalOrder` directly instead of forking `fetchEarlyRenewalOrderPreview`/`placeEarlyRenewalOrder` | Both functions are already generic over an arbitrary `LineItem[]` and contain no late-renewal-specific business rule — they just POST to `PREVIEW_RENEWAL`/`RENEWAL` with whatever line items they're given | None — the file and its exports (`fetchRenewalOrderPreview`, `renewalItemsToLineItems`) have already been renamed to renewal-mechanics-neutral names (previously `lateRenewalUtils.ts` / `fetchLateRenewalOrderPreview` / `lateRenewalItemsToLineItems`), so there's no naming mismatch for either dialog to inherit | Not enforced by a linter — relies on this LLD's documented rationale; `utils/renewalOrderUtils.ts` is the shared file, `LateRenewalOrderDialog.tsx` was updated to the new names alongside this feature |
| Existing-segment quantities default to the remaining renewable amount (`currentQuantity - renewedQuantity`) via a dedicated `initializeExistingQuantities`, rather than reusing `initializeQuantities` (which defaults to full `currentQuantity`) | A subscription that already has a `renewedQuantity` from a prior early renewal this cycle can only renew what's left — defaulting to the full `currentQuantity` would exceed the per-line `maxValue` clamp (`getRemainingExistingQuantity`) and mismatch what's actually submittable | One small new initializer function vs. reusing `initializeQuantities` as-is | `utils/earlyRenewalUtils.ts` — `initializeExistingQuantities` |
| Additional-segment quantities/toggles default to `1`/`false` instead of reusing `initializeQuantities`/`initializeRenewalStates` (which default to `currentQuantity`/`true`) | "Additional" means extra seats beyond what the customer already owns — defaulting to `currentQuantity` would be semantically wrong (that's what they already have, not what they're adding). Quantity defaults to `1`, not `0`, because 0 extra seats is not a valid "additional" quantity — `minValue={1}` on the NumberField enforces this floor whenever the line is toggled on; whether a product is added at all is controlled entirely by the renew toggle (default off) | Two small new initializer functions vs. reusing the existing ones as-is | `utils/earlyRenewalUtils.ts` — `initializeAdditionalQuantities`, `initializeAdditionalRenewalStates` |
| `hasEarlyRenewalOrderPlaced` exported as its own function rather than inlined inside `isInEarlyRenewalWindow` | The Catalog entry point needs to ask "has an early renewal already started for this contract" independently of the day-window check (it drives disabling `Now`, not `At Early Renewal`); the Customer Details entry point only ever needs the combined `isInEarlyRenewalWindow` check, but inlining the predicate there would leave the Catalog flow unable to query it standalone | None — `isInEarlyRenewalWindow`'s public behavior is unchanged either way, this is a small internal factoring choice | `utils/earlyRenewalUtils.ts` — `isInEarlyRenewalWindow` delegates to `hasEarlyRenewalOrderPlaced` |
| `renewedQuantity`-presence check (not a re-derived, day-shifted window) drives the "already rolled over" half of `isInEarlyRenewalWindow` | Simpler and doesn't need to track how many times AD has rolled over or re-derive a 366–395-day window; the experience card specifies exactly this OR condition | None significant — this is strictly simpler than the day-math alternative | `utils/earlyRenewalUtils.ts` — `isInEarlyRenewalWindow` |
| Early Renewal invalidates the subscriptions query cache (`customerDetailKeys.subscriptions(customerId)`) on submit success, unlike `LateRenewalOrderDialog` (which does not) | The experience card explicitly flags that submitting today does not refresh subscription data, and that "if immediate update is expected, it needs to be added as new behavior, not assumed to exist." `AddAndEditCustomerRenewalOrderDialog` already establishes this exact fix (`refetchSubscriptions()`) as a working precedent in this codebase | `EarlyRenewalOrderDialog` becomes slightly inconsistent with its closer structural sibling (`LateRenewalOrderDialog`, which still has the staleness gap) — acceptable since this feature is being built fresh and the fix is directly called out by the feature spec | `EarlyRenewalOrderDialog.tsx` — calls `useQueryClient().invalidateQueries(...)` directly; no prop plumbing through `pages/customerdetails.tsx` needed since `useQueryClient()` and `customerDetailKeys` are both globally available |
| "Early renew" button placed inside the existing anniversary-date row's `else` branch rather than as a new top-level conditional | The existing `hasManualRenewal && !renewalOrderSubmitted ? banner : (...)` ternary already suppresses this entire row when a late-renewal banner is showing — placing Early Renew here gets that precedence rule for free instead of re-implementing it | None | `components/customerdetails/CustomerProductsOverviewPanel.tsx` — existing ternary structure |
| Segmented control implemented with Spectrum `Tabs`/`TabList`/`Tab`/`TabPanel` rather than a fully custom two-button toggle | This repo already uses this exact Spectrum pattern for switching between related content in the same view (Customer Detail Hub's own tabs) — reusing it keeps a11y (keyboard nav, ARIA) handled automatically per `UI_PLATFORM.md`'s a11y baseline, instead of hand-rolling focus management for a custom control | Slightly heavier than a bare button pair, but consistent with the rest of the app | `@react-spectrum/s2` `Tabs` component — handles ARIA/keyboard automatically |
| A brand-new `/checkout/renewal` page rather than branching `/checkout` internally on `makeAvailable` | `/checkout` already has a fully worked-out `NEW`-order flow (its own preview/place-order calls, its own copy); branching it internally on order type would tangle two different order lifecycles (`NEW` vs `RENEWAL`) into one component's state machine | A second page with some structural duplication vs. `/checkout` (mitigated by reusing `CheckoutReviewDetails`/`CheckoutSummary`/`cartItemsToLineItems` as-is) | `pages/checkout/renewal.tsx` — separate route, separate page component |
| `CheckoutReviewDetails` / `CheckoutSummary` extended with optional, default-preserving props instead of forked into `EarlyRenewal*` variants | The only actual differences are copy (title, one extra bullet, a banner) — no layout/behavior fork is needed, so a full duplicate component would be pure repetition for a handful of strings | Two components gain a small, optional prop surface; every existing call site (`/checkout`) is unaffected since all new props default to today's exact strings | `components/checkout/CheckoutReviewDetails.tsx`, `components/checkout/CheckoutSummary.tsx` |
| New-products-only guard implemented as a SKU-set membership check (`customerOwnsAnyCartOffer`), reusing the existing `getSKUFromOfferId` truncation helper | This repo already compares offers by their first-8-characters SKU in two other places (`CartContext.updateCartItemsFromAPI`, `AddAndEditCustomerRenewalOrderDialog`'s existing-product matching) — reusing the same comparison keeps "does the customer already own this" consistent everywhere it's asked | None | `utils/earlyRenewalUtils.ts` — `customerOwnsAnyCartOffer` |

---

## §5 — Acceptance Criteria Coverage

| Goal (from experience card) | Covered by |
|-----------------------------|-----------|
| Partners can tell at a glance whether a customer is currently eligible for early renewal, without navigating away from the Customer Details page | `utils/earlyRenewalUtils.ts` (`isInEarlyRenewalWindow`), `components/customerdetails/CustomerProductsOverviewPanel.tsx` (MODIFY) |
| Partners cannot accidentally trigger an early renewal outside the eligible window | `utils/earlyRenewalUtils.ts` (`isInEarlyRenewalWindow` gates the button's existence entirely) |
| Partners can distinguish, within the dialog, between renewing a customer's existing licenses (up to current quantity) and adding additional seats beyond current quantity | `utils/earlyRenewalUtils.ts` (`splitSubscriptionsByRenewalProgress`, `getRemainingExistingQuantity`), `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` (segmented control + per-segment quantity clamp) |
| Partners see accurate, up-to-date pricing (per-license price, discount level, prorated days) before confirming the order | `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` (preview flow, reusing `fetchRenewalOrderPreview`/`updateSubscriptionsWithPreview`) |
| Partners get clear feedback — success or failure — after submitting, with the customer record reflecting the renewed quantity immediately after | `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` (`handleRenewNow` + query invalidation), `CustomerProductsOverviewPanel.tsx`'s existing `handleRenewalSuccess`/`handleRenewalError` (reused unchanged); Catalog path: `pages/checkout/renewal.tsx`'s existing `/orderConfirmation` redirect (reused verbatim) / `ErrorToast` on failure |
| Discount code apply/remove per subscription line | `components/renewal/earlyRenewal/EarlyRenewalOrderDialog.tsx` (`PromoCodeButton`, reused) |
| Error codes 2164 / 3115 / 3121 / 3131 / 3132 / 3133 surfaced to the partner (no per-code translation) | `placeRenewalOrder`/`fetchRenewalOrderPreview` (reused, already throw the backend's verbatim message) → `ErrorToast` / `onError`, on both entry points |
| "Existing before Additional" ordering rule (a product's Existing quantity must be fully renewed and processed before it's eligible for Additional) | ⚠ Not enforced client-side beyond the segment split itself — the split is computed from the subscription's current `renewedQuantity`/`currentQuantity` snapshot (which only reflects a *processed* prior order, since that's what the backend field represents), so the UI naturally reflects this once the backend has processed the prior order. There is no separate "processing in progress" state surfaced to the partner if they try to renew Additional while an Existing order for the same product is still processing — the backend's `3131` error code is the actual enforcement point in that case |
| Partners can add a new product to a customer's early renewal from the Catalog | `pages/checkout/renewal.tsx` (CREATE), `FindOrCreateCustomer.tsx` routing branch |
| Partners cannot create an ambiguous order mixing new products with owned products | `utils/earlyRenewalUtils.ts` (`customerOwnsAnyCartOffer`) disables "At Early Renewal" whenever the cart contains something the customer already owns |
| "Now" disabled while an early renewal is already in progress for the contract | `utils/earlyRenewalUtils.ts` (`hasEarlyRenewalOrderPlaced`), `ExistingCustomer.tsx` |
| Partner is told the anniversary date will move, and warned about the post-order restriction, on their first early renewal this cycle | `pages/checkout/renewal.tsx` — `willRollOverAD` derivation feeding `extraBullets` / `banner` |

---

## §6 — Out of Scope

- **`models/Subscription.ts` change** (`renewedQuantity?: number`) — already fully specified in the
  backend LLD; not duplicated here since it's the same file in the same repo.
- **Backend-side enforcement of the AD window, final-3YC-term block, or EOL/EOS SKU eligibility** —
  entirely the Adobe Partner API's responsibility (surfaced via error codes), not a UI concern; see
  backend LLD §5.
- **A dedicated "Early Renewal" entry in `UI_MODULE_INDEX.md`/other service cards** — service cards
  are regenerated fresh on the next `/generate-ui-service-card` run per the skill's own convention;
  not hand-edited by this LLD.