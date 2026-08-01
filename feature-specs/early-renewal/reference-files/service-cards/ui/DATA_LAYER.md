# Data Layer — Bridge UI

## 1. Overview

All data fetching goes through Next.js BFF routes (`/api/*`) on the same origin. No direct Partner API calls from the browser. No shared HTTP client or interceptor — auth token injection is server-side only (IMS token managed in `utils/imsTokenService.ts` on the BFF side).

**Query client config (`pages/_app.tsx`):**
```tsx
const queryClient = new QueryClient(); // module-level, no options
// Defaults: staleTime: 0, gcTime: 5min, retry: 3
```

Individual hooks override these defaults per-query.

---

## 2. Fetch Layer Summary

### Hooks (TanStack Query)

| Hook | File | Method | Endpoint | Cache Key | staleTime | gcTime | retry |
|------|------|--------|----------|-----------|-----------|--------|-------|
| `usePaginatedResellersList` | `hooks/usePaginatedResellersList.ts` | GET | `/api/resellers` | `['resellers','list',page,limit]` or `['resellers','search',term,page,limit]` | 5min | 10min | default |
| `usePaginatedCustomersList` | `hooks/usePaginatedCustomersList.ts` | GET | `/api/customers` | `['customers','list',resellerId,page,limit]` or `['customers','search',resellerId,term,page,limit]` | 5min | 10min | default |
| `useProductPricing` / `useProductNamesFromPricelist` | `hooks/useProductPricing.ts` | POST | `/api/pricelist` | `['subscriptionPricing',offerId,region,currency,priceListType]` | 10min | 30min | 2 |
| `useOfferSwitchPaths` | `hooks/useOfferSwitchPaths.ts` | GET | `/api/offer-switch-paths` | `['offerSwitchPaths',customerId,subscriptionId]` | 5min | 15min | default |

### Inline `useQuery` Calls (page-level)

| Page | Purpose | Cache Key | staleTime | gcTime |
|------|---------|-----------|-----------|--------|
| `customers.tsx` | Reseller detail (for breadcrumb) | `['reseller', resellerIdString]` | 10min | 10min |
| `customerdetails.tsx` | Customer detail | `customerDetailKeys.customer(id)` | 0 | 0 |
| `customerdetails.tsx` | Subscriptions | `customerDetailKeys.subscriptions(id)` | 0 | 0 |
| `customerdetails.tsx` | Reseller detail | `customerDetailKeys.reseller(id)` | 2min | 5min |

`staleTime: 0` on customer/subscription queries means they re-fetch on every mount. An 8-second delayed auto-refetch is also triggered after mount (`setTimeout(() => refetch(), 8000)`).

### Infinite Query (catalog)

| Page | Purpose | Query Key | Notes |
|------|---------|-----------|-------|
| `catalog.tsx` | Paginated pricelist browse | from `getPricelistQueryKey(marketSegment, currency)` | `useInfiniteQuery`, prefetches all market segments and adjacent pages |

### Imperative Fetches (not hooks)

| Function | File | Method | Endpoint | When Called |
|----------|------|--------|----------|------------|
| `fetchRenewalPreviewAndUpdateProducts` | `utils/renewalUtils.ts` | POST | `/api/orders?type=PREVIEW_RENEWAL` | On save in `EditRenewalDialog` |
| `updateRenewalSubscription` | `utils/renewalUtils.ts` | PATCH | `/api/subscriptions` | Per-subscription save in `EditRenewalDialog` |
| `fetchCustomerDetails` | `utils/AccountApis.ts` | GET | `/api/customers` | Passed as `queryFn` |
| `getResellerDetails` | `utils/AccountApis.ts` | GET | `/api/resellers` | Passed as `queryFn` |
| `fetchCustomerSubscriptions` | `utils/customerDetailsUtils.ts` | GET | `/api/subscriptions` | Passed as `queryFn` |
| `fetchOrderPreview` | `utils/cartUtils.ts` | POST | `/api/orders?type=Preview` | During checkout recalculation |

---

## 3. Standard Fetch Pattern

**TanStack Query hook (read):**
```tsx
const { data, isLoading, error } = useQuery({
  queryKey: ['reseller', resellerIdString],
  queryFn: async () => {
    const url = `/api/resellers?type=GET_RESELLER_DETAILS&id=${resellerIdString}`;
    const response = await fetch(url);
    if (!response.ok) throw new Error('Failed to fetch reseller details');
    return response.json();
  },
  enabled: !!resellerIdString && router.isReady,  // dependency guard
  staleTime: 10 * 60 * 1000,
  gcTime: 10 * 60 * 1000,
  retry: 2,
  retryDelay: 1000,
});
```

**Key rule:** Always guard with `enabled: !!<param> && router.isReady` when the query depends on a URL param. Router params are undefined on first render before hydration.

---

## 4. Auth Token Injection

**Pattern:** None in the browser. All `/api/*` calls are same-origin fetches with no explicit auth header. The BFF API routes handle IMS token fetching server-side via `utils/imsTokenService.ts`. The UI never sees or handles the token.

```tsx
// In every page/component — just a plain fetch, no auth header:
const response = await fetch(`/api/customers?type=GET_CUSTOMER&customerId=${id}`);
```

---

## 5. Standard Mutation Pattern

**TanStack Query `useMutation` (new order):**
```tsx
// hooks/useOrderOperations.ts
export function useCreateOrder() {
  return useMutation({
    mutationFn: async (payload: CreateOrderPayload) => {
      const response = await fetch('/api/orders?type=NEW', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message || 'Failed to create order');
      }
      return response.json();
    },
    retry: 3,
    retryDelay: (attempt) => Math.min(1000 * 2 ** attempt, 30000), // exponential backoff
  });
}
```

**Cache invalidation posture:** Mutations do NOT invalidate query cache. After a successful order, the user is navigated away (to `/orderConfirmation`). After subscription updates, the subscription query is re-fetched by the calling component via `queryClient.invalidateQueries`.

**Error surfacing:** Mutations surface errors via the `mutation.error` state or by catching in the calling component — displayed as toast (`ErrorToast` / `WarningToast`).

**Optimistic updates:** None used.

---

## 6. PartnerContext Data (Shared App-Wide)

```
Hook: usePartnerDetails()   (from contexts/PartnerContext.tsx)
Internal cache key: ['partnerDetails']
Endpoint: GET /api/partnerDetails
staleTime: 24 hours
gcTime: 7 days
retry: 1
```

Exposes: `partnerDetails`, `partnerName`, `availableCurrencies`, `region`, `regionCurrencies`, `marketSegments`, `isLoading`, `error`.

Initialized via `PartnerProvider` in `pages/_app.tsx`. Available in every component.

---

## 7. Cache Key Factories (`utils/queryUtils.ts`)

```ts
getCustomerListKey(resellerId, page, limit)       // ['customers', 'list', ...]
getCustomerSearchKey(resellerId, term, page, limit) // ['customers', 'search', ...]
getResellerListKey(page, limit)                   // ['resellers', 'list', ...]
getResellerSearchKey(term, page, limit)           // ['resellers', 'search', ...]
customerDetailKeys.customer(id)                   // ['customers', 'detail', id]
customerDetailKeys.subscriptions(id)              // ['customers', id, 'subscriptions']
customerDetailKeys.reseller(id)                   // ['resellers', 'detail', id]
```

Use these factories — never construct raw cache key arrays for customers/resellers.

---

## 8. Error Handling Posture

| Scenario | UI behaviour |
|----------|-------------|
| Query fetch error | Error state shown inline in page (not redirect); retry button shown |
| Mutation error | `ErrorToast` shown; form remains interactive |
| 3xx from API | Not handled — follows redirect |
| Network failure | TanStack Query retries per config, then surfaces `error` |
| Empty result | Empty state rendered inline (not error) |
