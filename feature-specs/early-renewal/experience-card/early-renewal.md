# Experience Card: Early Renewal

- Early Renew Entry Point (Customer Details Page)
- Early Renewal Order Dialog
- Add New Product Entry Point (Product Catalog → Find Existing Customer)

figma-url:

---

## Overview

Partners can manually renew a customer's subscriptions before the Anniversary Date (AD) — an **early renewal** — instead of waiting for auto-renewal to run on the Renewal Date. Early renewal is only offered during a fixed window before AD, is invoiced immediately, and rolls the AD forward once the first early renewal order for a contract succeeds.

This experience covers two ways a partner can start an early renewal:

1. **Early Renew Entry Point** — an action on the Customer Details page's Anniversary Date section, visible only while the customer is inside its early-renewal window. Opens the **Early Renewal Order Dialog** (Surface 2), which handles renewing the customer's **existing** products (up to and beyond current quantity).
2. **Add New Product Entry Point** — starting from the product Catalog instead of an existing customer's record, a partner can add a **brand-new** product (one the customer doesn't already own) into an early renewal order (Surface 3). This is the only path for adding a new product to an early renewal — Surface 2's dialog explicitly excludes new products (see the note at the end of Surface 2).

Both paths place the same kind of order against the same customer contract and are subject to the same eligibility window and error codes — they differ only in what they let the partner add (existing products vs. new products) and where the partner starts from.

---

## Goals

- Partners can tell at a glance whether a customer is currently eligible for early renewal, without navigating away from the Customer Details page.
- Partners cannot accidentally trigger an early renewal outside the eligible window.
- Partners can distinguish, within the Early Renewal Order Dialog, between renewing a customer's **existing** licenses (up to current quantity) and adding **additional** seats beyond current quantity.
- Partners can add a **new** product to a customer's early renewal from the Catalog.
- Partners cannot create an ambiguous order that mixes new products with products the customer already owns — the two paths (Surface 2 vs. Surface 3) stay cleanly separated.
- Partners see accurate, up-to-date pricing (per-license price, discount level, prorated days) before confirming the order, on either path.
- Partners get clear feedback — success or failure — after submitting, with the customer record reflecting the renewed quantity immediately after.

---

## Surface 1 — Early Renew Entry Point (Customer Details Page)

### Entry point

The **Early renew** action appears in the Anniversary Date section of the Customer Products Overview panel on the Customer Details page, next to **Edit renewal order**.

### When the action is shown

The Anniversary Date section (and the Early renew action within it) only renders when the customer's anniversary date is a valid, parseable date. Within that section, **Early renew** is shown when **either** of the following is true:

| Condition | Rule |
|---|---|
| Within the early-renewal window | Today falls **1–30 days before AD** (AD−30 … AD−1) |
| AD already rolled over | At least one of the customer's subscriptions has a **`renewedQuantity` ≥ 0** (i.e., the field is populated) — this means an early renewal was already placed this cycle, which is why AD rolled forward, so the action stays available for the remaining eligible quantity/products |

If neither condition holds — the customer is on/past AD with no early renewal recorded this cycle, or more than 30 days out with no early renewal recorded — **Early renew** is not shown.

---

## Surface 2 — Early Renewal Order Dialog

### Entry point

Clicking **Early renew** opens the Early Renewal Order Dialog.

### Loading the dialog

On open, the dialog:

1. Filters the customer's subscriptions to those eligible for renewal — status **Active (1000)** or **Scheduled (1009)**. If no subscriptions qualify, the dialog shows: *"No active subscriptions available for early renewal"*.
2. Splits eligible subscriptions into two segments, which map to the two order shapes the API allows for early renewal — an order renews either a product's existing quantity, or additional seats/new products beyond it, never both:
    - **Existing** — renews a subscription's current licenses, up to its current quantity (`renewedQuantity < currentQuantity`). A product must be fully renewed here — its entire current quantity early-renewed and that order processed — before it becomes eligible for the Additional tab.
    - **Additional** — extra seats beyond a product's current quantity, for products that have already been fully renewed in a prior, processed Existing order. At the API level, new products can also be combined into an Additional order alongside additional seats of existing products — this dialog, however, only surfaces additional seats of existing subscriptions (see note below on new products).
   Existing and Additional licenses can never be submitted in the same order — the segmented control enforces this by only letting the partner act on one segment's subscriptions at a time. If **Existing** has nothing left, the dialog defaults to the **Additional** segment.
3. Triggers a renewal preview to populate per-license price, discount level, and prorated days: `POST /v3/customers/{customerId}/orders` with `orderType: "PREVIEW_RENEWAL"`, sending one `lineItem` per subscription in the active segment (`extLineItemNumber`, `offerId`, `subscriptionId`, `quantity`, `currencyCode`, plus `flexDiscountCodes` for any line with a discount code already applied). Per response line item: `pricing.netPartnerPrice` populates the per-license price shown on the card, `pricing.lineItemPartnerPrice` populates the line total, and `proratedDays` populates the prorated-days figure; `pricingSummary[].totalLineItemPartnerPrice` (per currency) populates the Estimated total.

> **Important:** each response line item's own `offerId` — not the `offerId` the request was sent with — is what the card stores from that point on, and what later gets submitted with the order. Adobe's offer IDs carry a discount-level suffix that depends on the requested quantity, so the preview call is also what corrects the offer ID to the discount level actually applicable at that quantity. Submitting the pre-preview offer ID would risk placing the order at the wrong discount level.

**Loading state:** shown while the preview request is in flight.

**Error state:** if the preview fails, an inline error toast shows the failure message returned by the API.

---

### Screen layout

- **Deadline banner** — *"Submit early renewal before {date}"*, where `{date}` is formatted like "Jul 28, 2026". This is not simply the customer's raw anniversary date — once the customer has already had an AD roll-over (i.e., is in the 366–395-day-out window), the deadline calculation rolls the displayed date back by one year so it still reflects the *current* early-renewal cutoff.
- **License/transaction summary** — total license count and discount level for the active segment, with an **Update price** action to re-run the preview.
- **Segmented control — Existing / Additional** — switches which group of subscriptions is being edited. A segment is disabled (no tooltip) when it has no eligible subscriptions, with static descriptive text below each:
    - Existing — *"Renew current licences at the annual price"*
    - Additional — *"Renew extra seats beyond current quantity at prorated price"*
- **Per-subscription cards** (one per eligible subscription in the active segment):
    - Product icon and name
    - Renewed-quantity badge — shown only when `renewedQuantity > 0` for that subscription. Reads **"{renewedQuantity} of {currentQuantity} renewed"** on the Existing tab, and **"{renewedQuantity} renewed"** on the Additional tab.
    - Renew / Do-not-renew toggle
    - Quantity field — on **Existing**, clamped to the subscription's remaining renewable quantity (`currentQuantity − renewedQuantity`); on **Additional**, minimum of **1** with no upper bound — the partner can enter any quantity of extra seats, starting from 1 (0 is not a valid "additional" quantity)
    - Computed per-license price and line total (from the preview)
    - Offer ID
    - **Apply discount code** button — opens a small dialog to enter a code for that line; once applied, the button is replaced with a badge showing the code and a remove action
- **Estimated total**, shown per currency when subscriptions span multiple currencies.

> **Note:** Adding a brand-new product is not supported from this dialog — an inline note tells the partner: *"Adding a new product? New products are added from the Catalog as a separate early-renewal order — they can't be added here."*

---

### Submitting the order

The **Renew now** action is disabled when the total selected license quantity across the active segment is 0.

On submit:

1. The dialog builds the renewal payload from the currently selected/edited subscription lines (only lines with **Renew** toggled on are included), using each line's preview-corrected `offerId` (see the note in [Loading the dialog](#loading-the-dialog)).
2. The order is placed with body `{ customerId, externalReferenceId, lineItems, orderType: "RENEWAL" }`, `POST /v3/customers/{customerId}/orders`.
3. **Success** — the dialog closes and a success toast is shown: *"Renewal order submitted successfully! Order ID: {orderId}"*. This also suppresses the "Renewal window closes" banner (shown when the customer has a subscription eligible for manual/late renewal) for the rest of the session.
4. **Failure** — an error toast shows the message returned by the API. There is currently no per-error-code translation layer for early-renewal-specific errors (see [Error Handling](#error-handling)) — the backend's `message` field is surfaced verbatim.

> **Note:** Submitting does **not** explicitly refetch or invalidate the customer's subscription data. The Products list will only pick up the new `renewedQuantity` via whatever background refresh already runs on that page (a periodic timer, or refetching on returning to the Products tab) — there is no refetch call wired to this success path today. If the UI LLD is expected to show updated quantities immediately, this needs to be added as new behavior, not assumed to already exist.

---

## Surface 3 — Add New Product to Early Renewal (Product Catalog)

### Entry point

The way to add a **brand-new** product to a customer's early renewal is from the product Catalog.

The journey starts like an ordinary purchase: the partner adds a product to the cart from the Catalog page, opens the cart, and continues to attach a customer to the order. On the **existing-customer** side of that step, once a reseller and customer are picked, a **"Make licenses available"** choice appears with three options:

- **Now** — a normal purchase order, unrelated to renewal.
- **At Early Renewal** — bundles the cart's new product(s) into an early renewal order for the selected customer.
- **At renewal** — defers the product to the customer's normal renewal date (unchanged, existing behavior).

### When "At Early Renewal" is available

| Condition | Rule |
|---|---|
| Customer must be inside their early-renewal window | Same **either/or** rule as [Surface 1](#surface-1--early-renew-entry-point-customer-details-page): today falls **1–30 days before AD** (AD−30 … AD−1), **or** AD has already rolled over this cycle — at least one of the customer's subscriptions has a populated **`renewedQuantity` ≥ 0**, meaning an early renewal was already placed this cycle |
| Cart must contain only products the customer doesn't already own | If any product in the cart matches a product the customer already has a subscription for, **At Early Renewal** is disabled — this path is new-products-only. Renewing or adding seats to a product the customer already owns is done from the Customer Details dialog (Surface 2) instead. |

Conversely, if the customer already has an early renewal in progress for this contract, the plain **Now** option and **At Renewal** options are disabled, since only renewal-type orders are allowed until the current term ends. When **At Early Renewal** ends up as the only enabled choice (both **Now** and **At renewal** are disabled), a single message appears below the radio group: *"An early renewal has been detected for this customer, only renewal orders can be placed until the current term ends."*

### Early renewal checkout

Choosing **At Early Renewal** and continuing takes the partner to a checkout experience dedicated to early renewal — distinct from the checkout used by the plain **Now** option — where:

- The review and summary panels are explicitly labeled as an early renewal (e.g. "Review Details - Early Renewal" / "Early Renewal Summary"), so the partner isn't left assuming this is a normal purchase.
- Pricing is recalculated through the same renewal preview used in Surface 2 (`orderType: "PREVIEW_RENEWAL"`), so the per-license price and discount level reflect the early-renewal rate rather than a standard new-purchase rate — populated from the same `pricing.netPartnerPrice` / `pricing.lineItemPartnerPrice` / `proratedDays` response attributes described in Surface 2's [Loading the dialog](#loading-the-dialog). As in Surface 2, each response line's own `offerId` (discount-level-corrected for the requested quantity) replaces the offer ID the cart item was added with, and that corrected `offerId` is what gets submitted with the order below — not the one the product was added to the cart with.
- Proration is called out explicitly — e.g. *"This early renewal covers {N} days"* — followed by a note that the anniversary date will move to a new date, if this is the customer's first early renewal this cycle (i.e. AD is about to roll forward).
- If this is the customer's first early renewal this cycle, a banner warns that only renewal orders can be placed for this customer until the current term ends, naming the date.

### Submitting

Placing the order sends the cart's line items as a `RENEWAL` order (`POST /v3/customers/{customerId}/orders`, `orderType: "RENEWAL"`) — the same order type and endpoint Surface 2's "Renew now" uses — but the line items reference only the new offer(s), with no existing subscription attached, since these are new products rather than renewals of something the customer already holds.

On success, the partner is taken to the standard order confirmation screen, same as a normal purchase. On failure, the same error codes as Surface 2 apply (see [Error Handling](#error-handling)) — there is no per-error-code translation here either.

---

## Error Handling

Applies to the preview and submit calls made from both the Early Renewal Order Dialog (Surface 2) and the Catalog's early-renewal checkout (Surface 3).

The API can reject an early/late renewal request with the codes below (see the [API spec](../APISpec/early-renewal-apisepc.md#error-codes-specific-to-early-renewals) for full detail). None of these currently have a partner-friendly message mapping in the client — the backend-provided `message` is shown as-is in the error toast:

| Error Code | Condition |
|---|---|
| 2164 | All subscriptions on the order are already renewed. |
| 3115 | Invalid customer or subscription ID. |
| 3121 | A partially renewed existing subscription is combined in the same order with another line item (fully renewed subscription or new product) — existing licenses must be fully renewed first. |
| 3131 | An early renewal is already in progress for this contract; a restricted order type was submitted. |
| 3132 | The product in the request is not eligible for this order type (for example, an EOS SKU, or an EOL SKU on a non-3YC customer). |
| 3133 | Customer is in the final term of a three-year commitment (3YC) and has not recommitted — early renewal is blocked. |
