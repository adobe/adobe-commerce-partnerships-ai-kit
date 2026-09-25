# Experience Card: Partial Return

- Return Entry Point (Customer Details Page → Purchase History)
- Return Order Dialog

figma-url:

---

## Overview

Partners can return part of the quantity on a line item of a past order — a **partial return** — instead of only being able to return a line item in full. A partner can also submit multiple partial returns against the same line item over time, as long as the order stays within its return window; each request is checked against the line item's current remaining returnable quantity, not its original quantity.

This experience covers a single path: from a customer's **Purchase History**, a partner opens the **Return Order Dialog** for an eligible order and chooses, per line item, whether to return it and how many licenses to return.

---

## Goals

- Partners can tell at a glance which past orders are still eligible for a return, without leaving the Customer Details page.
- Partners cannot attempt a return on an order that is outside the return window, already fully processed as a return/switch, or not in a returnable status.
- Partners can return **fewer** licenses than a line item's full quantity, and see exactly how many licenses remain returnable before doing so.
- Partners cannot enter a return quantity that exceeds what is actually still returnable on a line item — remaining quantity already reflects any prior partial returns against it.
- Partners get clear feedback — success or failure — after submitting, and the Purchase History reflects the new return order once it's created.

---

## Surface 1 — Return Entry Point (Customer Details Page → Purchase History)

### Entry point

A **Return items** action appears in the Actions column of each row in the Purchase History table, on the Customer Details page's Purchase History tab.

### When the action is shown

The **Return items** action for a given order row is shown only when **all** of the following hold:

| Condition | Rule |
|---|---|
| Order status | Order status is **1000** (Fulfilled) |
| Order type | Order type is **not** `RETURN`, `SWITCH`, or `REVERT_SWITCH` — an already-returned, switch, or revert-switch order can't itself be returned (a switch order must instead use `REVERT_SWITCH`) |
| Within the return window | The order's creation date is **14 days old or less**

If any condition fails, the action is not shown for that row — there is no disabled/tooltip state, the button simply doesn't render.

---

## Surface 2 — Return Order Dialog

### Entry point

Clicking **Return items** on a Purchase History row opens the Return Order Dialog for that order.

### Loading the dialog

On open, the dialog resets any return selections/quantities from a previous use, then filters the order's line items to those with status **1000** and a remaining returnable quantity greater than 0, and renders one card per eligible line item.

For each line item, the **remaining returnable quantity** is its `remainingQuantity` if present, otherwise its full `quantity` — i.e. a line item that has never been partially returned is fully returnable; one that has already had a partial return processed shows only what's left. A line item with a remaining returnable quantity of 0 — already fully returned — is excluded from the dialog entirely; if every line item on the order has already been fully returned, the dialog shows *"No line items available for return"* instead of the list.

### Screen layout

- **Header** — "Return Order", with **Cancel** and **Return** actions.
- **Per-line-item cards** (one per eligible line item — a line item with no remaining returnable quantity is not shown at all:
    - Product icon and name (resolved from the pricelist by offer ID)
    - **"{remaining} of {original quantity} licenses returnable"**
    - **Return** toggle
    - When **Return** is toggled on: a **Quantity to return** field appears, labeled with the max returnable amount (e.g. "Quantity to return (max 55)"). It starts at a minimum of **1** and can be raised up to the remaining returnable quantity, and cannot be edited while a submission is in flight
    - When **Return** is toggled off: the line item shows its full original quantity as plain text (not editable) instead of the quantity field
    - Offer ID, shown for reference under the quantity control

The dialog can also be dismissed with the **Escape** key or by clicking outside it, equivalent to **Cancel**.

---

### Submitting the return

The **Return** action is disabled when: a submission is already in flight, no line item has **Return** toggled on, or the return already succeeded (so the action reads "Returned!" and can't be pressed again).

On submit:

1. The dialog builds the return payload from only the line items with **Return** toggled on and status **1000**, using each line's entered return quantity (starting at 1 by default, adjustable up to that line's remaining returnable quantity).
2. The return is placed with body `{ customerId, orderType: "RETURN", referenceOrderId, externalReferenceId, currencyCode, lineItems }`, `POST /v3/customers/{customerId}/orders`, where `referenceOrderId` is the original order's ID and each `lineItem` carries `extLineItemNumber`, `offerId`, `quantity` (the amount being returned, not the remaining quantity), and `currencyCode` when the line item has one.
3. **Success** — a "Return submitted" toast is shown, the **Return** button reads "Returned!", and after a short delay the dialog closes and the Purchase History list is refetched, so the new `RETURN` order (and the originating order's updated remaining quantity) appear once the backend reflects the change. The delay before refetching is intentional — it gives the backend a moment to process before the history is reloaded.
4. **Failure** — an error toast is shown, built from the API's error response as described in [Error Handling](#error-handling): a known return error code gets a partner-facing message; anything else falls back to the backend's generic `message`, surfaced verbatim.

---

## Error Handling

Applies to the submit call made from the Return Order Dialog.

Error responses come back in the shape `{ code, message, additionalDetails }`. The top-level `code`/`message` are generic and not specific to why the return failed (e.g. `code: "2120"`, `message: "Line item quantity out of range"`); the specific reason — including, when applicable, one of the return-specific codes below — is carried as a `"Reason: <CODE>"` entry inside `additionalDetails`:

The dialog builds its error toast as follows:

1. Scan `additionalDetails` for a `"Reason: <CODE>"` entry whose `<CODE>` matches one of the known return error codes below (a code → partner-facing-message lookup table).
2. If a match is found, show that **partner-facing message** instead of the raw backend text.
3. If no entry matches a known code, fall back to the top-level `message` (optionally with the raw `additionalDetails` lines appended), shown verbatim.

| Error Code | Partner-facing message |
|---|---|
| PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE | The offer is flagged as a Minimum Order Quantity (MOQ) offer. |
| SWITCH_ORDER_CANCELLATION_NOT_ALLOWED | The referenced order is a `SWITCH` order — `REVERT_SWITCH` must be used instead. |
| THREE_YEAR_COMMIT | The return would reduce a Three-Year Commit product family's quantity below its minimum commit quantity. |

The original order is left unchanged when a return is rejected — the dialog's remaining-quantity figures only update after a return actually succeeds and the history is refetched.
