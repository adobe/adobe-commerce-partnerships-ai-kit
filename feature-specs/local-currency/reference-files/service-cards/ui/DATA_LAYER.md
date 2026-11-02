# DATA_LAYER.md — adobe-commerce-partnerships-ref-app (UI)

## Overview

| Field | Value |
|---|---|
| Data fetch library | TanStack Query v5 (`@tanstack/react-query`) for all cached reads and mutations; a handful of one-off reads/writes call `fetch` directly with no cache (see Auth/Cache notes per unit below) |
| Auth injection pattern | **None** — every fetch unit calls adobe-commerce-partnerships-ref-app's own `pages/api/*` routes with no `Authorization` header and no credential of any kind. The client never holds or sends a token; adobe-commerce-partnerships-ref-app's backend supplies its own service credential to the upstream Partner API (see the backend `CONNECTORS.md`). |
| Default `staleTime` / `gcTime` | No global defaults — `new QueryClient()` is constructed with no `defaultOptions` (`pages/_app.tsx:20`), so any query without an explicit `staleTime`/`gcTime` uses TanStack Query's library defaults (`staleTime: 0`, `gcTime: 5 minutes`). Every fetch unit in this app sets its own values explicitly except `PurchaseHistoryPanel`'s orders-history query, which explicitly sets `staleTime: 0, gcTime: 0`. |
| Error surface | Query errors: rendered inline per-page/component (`{error && <div>...}` blocks) with a "Try Again" button. Mutation errors: caught in the mutation's `onError` callback and shown via a toast (`ErrorToast`/`useToastState`) or local `error` state. |
| Optimistic update posture | Not used for server mutations (no manual cache-writing before a mutation resolves). `CartContext`'s cart operations (`addToCart`/`updateQuantity`/`removeFromCart`) *are* effectively optimistic in the sense that they mutate local/localStorage state immediately with no server round-trip — the cart itself has no server representation until checkout. |
| SSR pre-fetching | None — this is a pure CSR app (no `getServerSideProps`/`getStaticProps` anywhere in `pages/`); every query fires client-side after mount. |

## Fetch Layer Summary

| # | Fetch unit | Method | Endpoint | Cache key |
|---|---|---|---|---|
| 1 | [fetchPartnerDetails](#1-fetchpartnerdetails--partner-configuration) | GET | `/api/partnerDetails` | `['partnerDetails']` |
| 2 | [fetchResellersPage](#2-fetchresellerspage-paginated-reseller-list) | GET | `/api/resellers?type=getAllResellers` | `['resellers','list',page,limit]` |
| 3 | [searchResellers](#3-searchresellers--reseller-search) | GET | `/api/search?type=reseller` | `['resellers','search',term,page,limit]` |
| 4 | [fetchCustomersPage](#4-fetchcustomerspage-paginated-customer-list) | GET | `/api/customers?type=getAllCustomers` | `['customers','list',resellerId,page,limit]` |
| 5 | [searchCustomers](#5-searchcustomers--customer-search) | GET | `/api/search?type=customer` | `['customers','search',resellerId,term,page,limit]` |
| 6 | [getResellerDetails](#6-getresellerdetails--single-reseller-lookup) | GET | `/api/resellers?type=getResellerDetails` | `['reseller', resellerId]` |
| 7 | [fetchCustomerDetails](#7-fetchcustomerdetails--single-customer-lookup) | GET | `/api/customers?type=getCustomer` | `customerDetailKeys.customer(id)` → `['customerDetails','customer',id]` |
| 8 | [fetchCustomerSubscriptions](#8-fetchcustomersubscriptions--customers-active-subscriptions) | GET | `/api/subscriptions?type=getCustomerSubscriptions` | `customerDetailKeys.subscriptions(id)` |
| 9 | [fetchOrdersHistory](#9-fetchordershistory--paginated-order-history) | GET | `/api/orders?type=getOrdersHistory` | `['ordersHistory', customerId, page]` |
| 10 | [fetchPricelistPage](#10-fetchpricelistpage--catalog-price-list-page) | POST | `/api/pricelist` | `getPricelistQueryKey(segment, currency)` → `['pricelist-infinite', segmentCode, currency]` (infinite query) |
| 11 | [fetchProductPricing](#11-fetchproductpricing--single-offer-price-lookup) | POST | `/api/pricelist` | `['subscriptionPricing', offerId, region, currency, priceListType]` |
| 12 | [fetchOrderPreview](#12-fetchorderpreview--cart-price-preview) | POST | `/api/orders?type=Preview&fetch-price=true` | n/a — imperative call, not cached |
| 13 | [useCreateOrder](#13-usecreateorder--create-a-new-order-mutation) | POST | `/api/orders?type=NEW` | n/a (mutation) |
| 14 | [useReturnOrder](#14-usereturnorder--create-a-return-order-mutation) | POST | `/api/orders?type=Return` | n/a (mutation) |
| 15 | [useCreateOrderPreview](#15-usecreateorderpreview--create-an-order-preview-mutation-unused) | POST | `/api/orders?type=Preview` | n/a (mutation, unused) |
| 16 | [createCustomer (inline)](#16-createcustomer-inline--create-a-new-customer) | POST | `/api/customers?type=createCustomer` | n/a — imperative call, not cached |
| 17 | [enroll in 3YC (inline)](#17-enroll-in-3yc-inline--enroll-customer-in-three-year-commit) | PATCH | `/api/customers?type=updateCustomer` | n/a — imperative call, not cached |
| 18 | [fetchRenewalPreviewAndUpdateProducts](#18-fetchrenewalpreviewandupdateproducts--renewal-price-preview) | POST | `/api/orders?type=PreviewRenewal&fetch-price=true` | n/a — imperative call, not cached |
| 19 | [createRenewalSubscription](#19-createrenewalsubscription--add-a-new-product-to-renewal) | POST | `/api/subscriptions?type=createSubscription` | n/a — imperative call, not cached |
| 20 | [updateRenewalSubscription](#20-updaterenewalsubscription--update-subscription-auto-renewal) | PATCH | `/api/subscriptions[?reset-flex-discount-codes=true]` | n/a — imperative call, not cached |
| 21 | [createRenewalOrder](#21-createrenewalorder--submit-a-late-renewal-order) | POST | `/api/orders?type=RenewalOrder` | n/a — imperative call, not cached |

### 1. fetchPartnerDetails — Partner configuration
- **File:** `contexts/PartnerContext.tsx`
- **Auth injection:** none
- **Dependency guard:** none — fires unconditionally on `PartnerProvider` mount (which wraps the whole app)
- **Freshness:** `staleTime: 24h`, `gcTime: 7d`, `retry: 1` — this data changes essentially never during a session
- **Return shape:** `PartnerDetails` — `{ partnerName, marketSegments: {programType, marketSegment}[], currencies: {priceRegion, currency}[] }`
- **Consumers:** every page, via `usePartnerDetails()`

### 2. fetchResellersPage — Paginated reseller list
- **File:** `utils/AccountApis.ts::fetchResellersPage`, wrapped by `hooks/usePaginatedResellersList.ts`
- **Dependency guard:** `enabled: !isSearchMode` (mutually exclusive with unit 3)
- **Freshness:** `staleTime: 5min`, `gcTime: 10min`; `placeholderData: previousData => previousData` (keeps prior page visible while the next loads)
- **Return shape:** `{ resellers: ResellerMinimal[], totalCount, count, offset, limit, hasMore }`
- **Prefetch:** the hook also `queryClient.prefetchQuery`s the next *and* previous page on every page change

### 3. searchResellers — Reseller search
- **File:** `utils/AccountApis.ts::searchResellers`, wrapped by `usePaginatedResellersList.ts`
- **Dependency guard:** `enabled: isSearchMode` (i.e. `debouncedSearch.trim().length > 0`); search term is debounced 300ms via `useSearchDebounce`
- **Return shape:** same paginated shape as unit 2

### 4. fetchCustomersPage — Paginated customer list
- **File:** `utils/AccountApis.ts::fetchCustomersPage`, wrapped by `hooks/usePaginatedCustomersList.ts`
- **Dependency guard:** `enabled: !isSearchMode && !!resellerId`
- **Return shape:** `{ customers: CustomerMinimal[], totalCount, count, offset, limit, hasMore }`

### 5. searchCustomers — Customer search
- **File:** `utils/AccountApis.ts::searchCustomers`, wrapped by `usePaginatedCustomersList.ts`
- **Dependency guard:** `enabled: isSearchMode && !!resellerId`

### 6. getResellerDetails — Single reseller lookup
- **File:** `utils/AccountApis.ts::getResellerDetails`
- **Called directly via `useQuery`** (not wrapped in a custom hook) in `pages/customers.tsx`, `pages/customerdetails.tsx`, `pages/checkout.tsx` — same fetch function, three separately-declared `useQuery` calls with the same `['reseller', id]` key (so the cache is shared across pages even without a shared hook)
- **Dependency guard:** `enabled: !!resellerIdString`
- **Freshness:** `staleTime: 10min`, `gcTime: 10min`, `retry: 2`

### 7. fetchCustomerDetails — Single customer lookup
- **File:** `utils/AccountApis.ts::fetchCustomerDetails`, called via `useQuery` in `pages/customerdetails.tsx`
- **Dependency guard:** `enabled: !!customerIdString && router.isReady`
- **Freshness:** `staleTime: 0`, `gcTime: 0` (always refetched — this page needs the latest discounts/benefits state), `retry: 2`

### 8. fetchCustomerSubscriptions — Customer's active subscriptions
- **File:** `utils/customerDetailsUtils.ts::fetchCustomerSubscriptions`, called via `useQuery` in `pages/customerdetails.tsx`
- **Dependency guard:** `enabled: !!customerIdString && router.isReady`
- **Freshness:** `staleTime: 0`, `gcTime: 0`, `retry: 3`; also force-`refetch()` on an 8-second timer and whenever the "Products" tab is re-activated

### 9. fetchOrdersHistory — Paginated order history
- **File:** inline in `components/customerdetails/PurchaseHistoryPanel.tsx` (not extracted to `utils/`)
- **Dependency guard:** `enabled: !!customerId`
- **Freshness:** `staleTime: 0`, `gcTime: 0`, with `placeholderData` for pagination
- **Invalidated by:** unit 14 (`useReturnOrder`) on success, via `queryClient.invalidateQueries({ queryKey: ['ordersHistory', customerId] })`

### 10. fetchPricelistPage — Catalog price list page
- **File:** `utils/prefetchProducts.ts::fetchPricelistPage`, consumed via `useInfiniteQuery` in `pages/catalog.tsx`
- **Dependency guard:** `enabled: isPartnerDetailsReady && !!region && !!filters.currency`
- **Freshness:** `staleTime: 5min`, `gcTime: 10min`; `getNextPageParam` returns `undefined` once `hasMore` is false
- **Prefetch:** `prefetchAllMarketSegments`/`prefetchOtherMarketSegments` warm the cache for the other two market segments in the background after the active one loads
- **Return shape:** `{ offers: PriceListOffer[], hasMore, page, ...rest }`

### 11. fetchProductPricing — Single-offer price lookup
- **File:** `hooks/useProductPricing.ts::fetchProductPricing`, fanned out via `useQueries`
- **Dependency guard:** `enabled: !!request.offerId && !!region && !!request.currencyCode`
- **Freshness:** `staleTime: 10min`, `gcTime: 30min`, `retry: 2`
- **Consumers:** `ReturnOrderDialog` (product names for line items), renewal dialogs (current pricing for existing subscriptions)

### 12. fetchOrderPreview — Cart price preview
- **File:** `utils/cartUtils.ts::fetchOrderPreview`
- **Auth injection:** none; **not wrapped in `useQuery`** — called imperatively from `pages/checkout.tsx::handleRecalculate` and `components/catalogCart/CartDetails.tsx` whenever quantities/promo codes change
- **Return shape:** `Order` (`models/Order.ts`) — throws if `data.lineItems` is empty

### 13. useCreateOrder — Create a new order (mutation)
- **File:** `hooks/useOrderOperations.ts::useCreateOrder`
- **Retry:** custom — up to 3 attempts, only on 5xx or network errors, exponential backoff capped at 10s
- **Consumer:** `pages/checkout.tsx::placeOrder`

### 14. useReturnOrder — Create a return order (mutation)
- **File:** `hooks/useOrderOperations.ts::useReturnOrder`
- **Retry:** none (default TanStack Query mutation retry = 0)
- **Consumer:** `components/customerdetails/ReturnOrderDialog.tsx::handleReturn` — see `Standard Mutation Pattern` below

### 15. useCreateOrderPreview — Create an order preview (mutation, unused)
- **File:** `hooks/useOrderOperations.ts::useCreateOrderPreview`
- **Note:** exported but not called anywhere in the codebase — `fetchOrderPreview` (unit 12) duplicates this same `POST /api/orders?type=Preview` call as a plain imperative function instead. Two parallel implementations of the same operation; prefer `fetchOrderPreview` (the one actually used) as the reference when adding new preview call sites, or consolidate onto this hook.

### 16. createCustomer (inline) — Create a new customer
- **File:** `components/catalogCart/customerdetails/NewCustomer.tsx::createCustomer`
- **Not a TanStack Query mutation** — plain `async` function with manual `isLoading`/`error` state
- **Consumer:** the "Find or Create Customer" modal in the catalog checkout flow

### 17. enroll in 3YC (inline) — Enroll customer in three-year commit
- **File:** `components/ThreeYearCommit/ThreeYearCommit.tsx::handleInviteToEnroll`, URL built by `utils/threeYearCommitUtils.ts::buildApiUrl`
- **Not a TanStack Query mutation** — plain `async` function with manual `isEnrolling` state
- **Cache interaction:** on success, the caller (`pages/customerdetails.tsx`) manually calls `queryClient.invalidateQueries({ queryKey: customerDetailKeys.customer(customerIdString) })`

### 18. fetchRenewalPreviewAndUpdateProducts — Renewal price preview
- **File:** `utils/renewalUtils.ts` (also duplicated in `utils/renewalOrderUtils.ts` for the late-renewal flow) — both POST to `/api/orders?type=PreviewRenewal&fetch-price=true`
- **Not cached** — imperative call from `EditRenewalDialog`/`LateRenewalOrderDialog` whenever quantities or promo codes change, mirroring the `fetchOrderPreview` pattern (unit 12) but for renewals

### 19. createRenewalSubscription — Add a new product to renewal
- **File:** `utils/renewalUtils.ts` (`POST /api/subscriptions?type=createSubscription`)
- **Consumer:** `EditRenewalDialog` when a cart item is being added to the customer's renewal (not yet an existing subscription)

### 20. updateRenewalSubscription — Update subscription auto-renewal
- **File:** `utils/renewalUtils.ts` (`PATCH /api/subscriptions`, optionally with `?reset-flex-discount-codes=true`)
- **Consumer:** `EditRenewalDialog::toggleAutoRenewal`/quantity changes for existing subscriptions

### 21. createRenewalOrder — Submit a late renewal order
- **File:** `utils/renewalOrderUtils.ts` (`POST /api/orders?type=RenewalOrder`)
- **Consumer:** `components/renewal/lateRenewal/LateRenewalOrderDialog.tsx` — the "late renewal" flow (submitting a renewal order directly, past the normal auto-renewal window; see `MANUAL_RENEWAL_WINDOW_DAYS` in `utils/constants.ts`)

## Standard Fetch Pattern

Representative example — `hooks/usePaginatedResellersList.ts` (unit 2/3), annotated:

```ts
const { data, isLoading, error, isFetching, isPlaceholderData } = useQuery<PaginatedResellersResponse>({
  queryKey: [...getResellerListKey(currentPage, itemsPerPage)],   // (1) cache key — from utils/queryUtils.ts
  queryFn: () => fetchResellersPage(currentPage, itemsPerPage),   // plain fetch() wrapper, no auth header
  enabled: enabled && !isSearchMode,                              // (2) dependency guard
  placeholderData: previousData => previousData,                 // keep old page visible during refetch
  staleTime: STALE_TIME,                                          // 5 min
  gcTime: GC_TIME,                                                // 10 min
});
// (3) return shape: { resellers, totalCount, count, offset, limit, hasMore }
```
Every list-fetching hook in this app (`usePaginatedCustomersList`, `usePaginatedResellersList`) follows this exact shape: a "list" query and a "search" query, mutually `enabled`-gated by `isSearchMode`, both using the same query-key-generator + prefetch-adjacent-pages pattern.

## Standard Mutation Pattern

Representative example — `components/customerdetails/ReturnOrderDialog.tsx::handleReturn` (unit 14), annotated:

```ts
returnOrder.mutate(
  { customerId, referenceOrderId, externalReferenceId, currencyCode, lineItems }, // (1) payload → call
  {
    onSuccess: () => {
      showSuccess('Return submitted');
      setTimeout(() => {
        onClose();
        queryClient.invalidateQueries({ queryKey: ['ordersHistory', customerId] }); // (2) cache update
      }, 1500);
    },
    onError: (err: Error) => {
      showError(err.message || 'Failed to create return order'); // (3) error surface
    },
  }
);
```
Cache invalidation is always explicit (`queryClient.invalidateQueries`), never automatic — this app does not use `onSettled` or mutation-scoped `invalidatesTags`-style config. A short `setTimeout` delay (1.5s here) before invalidating/navigating is a recurring idiom to give the backend time to reflect a just-submitted write before the next read.

## Auth Token Injection

**None of the three options (A/B/C) apply.** No fetch unit in this codebase attaches any
`Authorization` header, cookie, or token — every call is a same-origin request to adobe-commerce-partnerships-ref-app's own
`/api/*` routes with `{ 'Content-Type': 'application/json' }` as the only header set. adobe-commerce-partnerships-ref-app's
server-side API routes are responsible for their own credential to the upstream Partner API (see
the backend service cards' `CONNECTORS.md`). There is no shared HTTP client/interceptor file —
every fetch unit calls the global `fetch()` directly.

## Error Handling Posture

**Fetch (query) errors** — rendered inline, always with a retry affordance:
```tsx
) : error ? (
  <div className={styles.errorMessage}>
    <h3>Unable to load customers</h3>
    <p>{error?.message || 'An unexpected error occurred'}</p>
    <Button variant="secondary" onPress={() => invalidateCustomers(queryClient)}>Try Again</Button>
  </div>
```

**Mutation errors** — surfaced via the shared toast components (`utils/ToastMessageUtils.tsx`) or
a per-component `useToastState()` hook, never thrown further up:
```tsx
onError: (error: any) => {
  const apiErrorMessage = error?.message || 'Failed to create order. Please try again.';
  setErrorMessage(apiErrorMessage);
  setShowErrorToast(true);
  createOrder.reset();
}
```

## Loading States

- **List pages** (`resellers`, `customers`, `catalog`): `LoadingSkeleton` (`components/LoadingSkeleton.tsx`, memoized) renders `count` placeholder cards on first load; while a background refetch is in flight (`isFetching && isPlaceholderData`), a small inline spinner overlays the existing list instead of replacing it.
- **Mutations / imperative actions**: full-overlay `ProgressCircle` (React Spectrum) with a text label, e.g. `<ProgressCircle isIndeterminate size="L" />` + "Creating customer...", blocking the form beneath it.
- **Tabs** (`customerdetails.tsx`): each tab panel manages its own independent loading state (`AccountDetailsSkeleton`, inline "Loading purchase history...", `LoadingSkeleton`) — switching tabs does not show a page-level spinner.

## Adding a New Read Unit — Checklist

1. Identify which capability in `UI_MODULE_INDEX.md` this read supports.
2. Name the fetch function `fetch<Noun>` or `search<Noun>` to match existing conventions (see units 2–9).
3. Define a cache key via a generator in `utils/queryUtils.ts` (or co-locate if it's a one-off, as units 10/11 do) — never inline a raw array literal for a key that will be invalidated elsewhere.
4. Set an `enabled` dependency guard for every required upstream value (id, region, currency, `router.isReady`) — every existing query in this app does this.
5. Set explicit `staleTime`/`gcTime` — do not rely on the (unset) global defaults.
6. Validate the response shape defensively (`data?.field || fallback`) — the client does not run Zod on responses; only the backend does.
7. Add a row to `Fetch Layer Summary` above.
8. Add the fetch unit to the relevant capability's entry in `UI_MODULE_INDEX.md`.