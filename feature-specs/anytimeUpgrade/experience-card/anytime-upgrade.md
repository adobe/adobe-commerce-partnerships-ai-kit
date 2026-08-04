# Experience Card: Anytime Upgrade

- Upgrade Entry Point (Active Products, Customer Details Page)
- Upgrade Paths Dialog
- Upgrade Order Summary (Checkout Page)

figma-url:

---

## Overview

Anytime Upgrade lets a partner move a customer's active subscription to a higher-tier product before the subscription's anniversary date, instead of waiting for renewal. The upgrade is executed as a **switch order** — the existing subscription is cancelled and replaced with the new one, with prorated pricing applied for the remaining term.

A partner starts an upgrade from the **Active Products entry point** — an **Upgrade** action on a subscription's product card on the Customer Details page, driven by that subscription's available upgrade paths. From there, the flow continues to the **Upgrade Paths Dialog** (target selection) and then the **Upgrade Order Summary** checkout page (preview + place order), placing a `SWITCH` order.

The API layer also supports reverting a switch order (`PREVIEW_REVERT_SWITCH` / `REVERT_SWITCH`, within a 14-day window) - **this is not implemented in this app's UI today.

---

## Goals

- Partners can tell at a glance whether a subscription has an available upgrade, without leaving the Customer Details page.
- Partners cannot open the upgrade flow for a subscription whose currency isn't one the partner is contracted in, or that's missing an offer ID.
- Partners see an accurate price preview — including prorated amount — before committing to the switch order.
- Partners get clear feedback after submitting an upgrade order.

---

## Surface 1 — Upgrade Entry Point (Active Products, Customer Details Page)

### Entry point

An **Upgrade** action appears on a subscription's card in the Active Products list.

### When the action is shown

The action is shown for a given subscription when **all** of the following hold:

| Condition | Rule |
|---|---|
| Upgrade paths exist | `GET /v3/offer-switch-paths` (keyed by the subscription's ID) returns a non-empty `productUpgrades` entry with at least one `targetList` item |
| Offer ID present | The subscription has an `offerId` |


**Error handling:** If the eligibility call fails, it is a silent failure — no error toast, no retry affordance. The subscription's upgrade data simply resolves as empty, so the Upgrade button does not render. This looks identical to a subscription that legitimately has no upgrade paths.

---

## Surface 2 — Upgrade Paths Dialog

### Entry point

Pressing **Upgrade** on an Active Products card (Surface 1) opens the Upgrade Paths Dialog.

### Loading the dialog

- The list of upgrade targets (`targetBaseOfferId`, `switchType` per target) is **passed in** from the data already fetched for eligibility (Surface 1) — no new call is made to re-fetch upgrade paths.
- Product names for each target are resolved via a separate, **live** pricelist lookup per offer, made when the dialog opens. If that lookup fails or returns no match for an offer, the raw offer ID string is shown instead of a product name.
- Targets render in the order returned by the API's `targetList` — the response does carry a `sequence` field per target, but it is not used to sort the list.

**Loading state:** "Loading upgrade options..." is shown in place of the target list while names are resolving.

### Screen layout

- Header: "Available upgrades", with "Current product: **{product name}**"
- One radio option per target offer: product icon, product name (or offer ID fallback), offer ID
- Once a target is selected, a summary line appears: "Upgrade to: **{selected product name}**"

**User action:** User selects one target offer.

**Actions:**
- **Cancel** — closes the dialog, no data is captured
- **Proceed** — disabled until a target is selected; on press, captures the selected `targetOfferId` and its `switchType`, closes the dialog, and navigates to the Upgrade Order Summary (Surface 3)

There is no explicit empty-state message in this dialog — it is only ever opened when Surface 1 has already confirmed at least one upgrade path exists, so an empty list isn't a state the dialog needs to handle today.

---

## Surface 3 — Upgrade Order Summary (Checkout Page)

### Entry point

Reached from Surface 2's **Proceed** action. All data needed for the order — customer ID, reseller ID, source offer/subscription/currency, target offer, switch type, quantity — is carried via URL query parameters into the checkout page; nothing is passed through app context or session storage.

### Screen layout

**Breadcrumb:** Resellers > {reseller name} > {customer name} > Review upgrade

**Left panel — "Review upgrade details":**
- This purchase is for **{customer name}** on behalf of **{reseller name}**.
- Upgraded licenses will be immediately available and automatically assigned.
- Source → target product transition row: source product icon + "Licenses", a directional arrow, target product icon + "Licenses"
- Adobe is not responsible for billing these licenses to the customer or collecting payments.
- The amount is the partner price difference between **{source product name}** and **{target product name}** and pro-rated to the next anniversary date.

**Right panel — "Order summary":**
- Subheading: "{quantity} Licenses ({discount level text})" — same licenses-count-plus-discount-level pattern already used on the Renewal flow
- "Reflecting partner pricing for customer."
- Product card: target product icon + **{target product name}**, "Offer ID: {target offer ID}"
- Quantity field, labeled "Licenses":
  - Editable stepper (minimum 1, maximum = the source subscription's quantity at the time the upgrade was started) when the switch type is **partial-allowed**
  - **Read-only**, fixed at the full quantity, when the switch type is **full-only** — this switch type does not allow partial quantity upgrades
  - Line total for the current quantity shown to the right of the stepper
- Below the quantity field: "{per-license price} per license" and "({proratedDays} days proration)", with an **Apply discount code** link at the right
- **Update price** action
- "Estimated Total" heading with the total amount to the right, and "+local taxes apply" beneath the amount
- Terms line: By clicking "Place order", you will agree to pay Adobe for these licenses pursuant to Adobe's APC Agreement with you.
- **Place order** action

### Pricing preview

- The preview (`PREVIEW_SWITCH`) fires **automatically on page load** — it is not a step the partner has to trigger themselves.
- **Update price** re-runs the preview with the current quantity. It stays disabled until something has actually changed (a quantity edit) or while a preview request is already in flight — pressing it when nothing has changed is not possible.
- Applying or removing a discount/promo code also re-runs the preview immediately, independent of the Update price button.
- Preview request: one line item (target offer, quantity, any applied discount code) plus one cancelling item referencing the source subscription, **with that same quantity** — not the source subscription's full original quantity. A quantity edit updates both the line item and the cancelling item together; the partner is swapping N licenses from the source product to the target, not necessarily all of them.
- Preview response populates: per-license price, line total, and the estimated total (per the response's own currency, which may differ from the source currency initially assumed).

**Loading state:** shown while the preview request is in flight; Place order is disabled during this time.

**Error state:** if the preview fails, an inline message shows the backend's error message when available, falling back to "Failed to load pricing. Click 'Update price' to retry." if not.

### Submitting the order

**Place order** is disabled when: there's no line item yet, the estimated total hasn't resolved, a quantity/discount change is pending a fresh preview, a preview is currently in flight, or **the partner does not have write permission on the org** — this is the only point in the entire upgrade flow where write permission is checked.

On press, the switch order is submitted using the same request shape as the preview (target offer + quantity + source subscription as cancelling item **with that same quantity**), independent of the last preview response's own values.

**UI outcome — success:** navigates to a generic order-confirmation screen (the same one used for other order types), carrying the computed total, target product name, and quantity from local state — not literally the values returned by the switch order response.

**UI outcome — error:** a fixed, generic message — "Failed to place order. Please try again." — is shown. Unlike the preview error, the actual backend failure reason is **not** surfaced here. There is no automatic retry; the partner must press Place order again manually.

---

## Error Handling

Applies to both the preview and submit calls from the Upgrade Order Summary screen (Surface 3).
The **preview** and **plcae order** error path shows the backend's message as-is;

---
## Edge cases

| Scenario | Behaviour |
|---|---|
| No upgrade paths for a subscription | No Upgrade action shown; silent eligibility-API failure also hides it, indistinguishably from a genuine no-upgrade-paths case |
| Switch type is full-only | Quantity field is read-only on the Order Summary screen; partner cannot upgrade a partial quantity |
| Preview API fails on load | Inline error (backend message if available) on Order Summary screen; Place order stays disabled until a successful preview |
| Place order fails | Generic inline error only; partner stays on the Order Summary screen; must manually retry (no auto-retry, no backend detail shown) |
| User navigates back mid-flow | No order is placed; no side effects |