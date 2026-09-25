# STATE_MANAGEMENT.md — adobe-commerce-partnerships-ref-app (UI)

## Overview

| Field | Value |
|---|---|
| Library | React Context + `useState`/`useEffect` (no Zustand/Redux/Jotai/Recoil) for client-owned UI state; TanStack Query for all server-derived state (see `DATA_LAYER.md`) |
| Store count | 2 — `CartContext`, `PartnerContext` |
| Init chain enforcement | Manual, via provider nesting order in `pages/_app.tsx` — not reactive/automatic. `PartnerProvider` wraps `CartProvider`, but `CartProvider` does not actually read from `PartnerContext` (no dependency in practice, despite the nesting) |
| Persisted stores | `CartContext` only — persists to `localStorage` under key `bridge_cart`. `PartnerContext` is not persisted; it is TanStack-Query-cached in memory only (`staleTime: 24h`) |

## 1. CartContext

- **File:** `contexts/CartContext.tsx`
- **Provider position:** `pages/_app.tsx` — innermost provider, wraps `<Component />` directly

**State shape:**
```ts
cartItemIdToQuantityMap: { [offerId: string]: number }
cartItemIdToCartItemDetailsMap: { [offerId: string]: Omit<CartItem, 'quantity'> }
customerInfoInCart: { customerId?, customerName?, resellerId?, resellerName?, makeAvailable?: LicenseAvailability }
showCartModal: boolean
// CartItem = { id, offerId, productName, pricePerUnit, quantity, currency, productFamily?, lineItemTotal?, marketSegment?, discountCode? }
```

**Actions:**
| Action | Signature | Effect |
|---|---|---|
| `addToCart` | `(offerId, productName, pricePerUnit, currency, marketSegment?, productFamily?, quantity=1)` | increments quantity map, upserts details map (preserves existing `id` UUID if item already present) |
| `removeFromCart` | `(offerId)` | deletes from both maps |
| `updateQuantity` | `(offerId, quantity)` | sets quantity, or calls `removeFromCart` if `quantity <= 0` |
| `clearCart` | `()` | resets both maps and `customerInfoInCart` to empty |
| `setCustomerInfoInCart` | `(customerInfo)` | replaces `customerInfoInCart` wholesale |
| `clearCustomerInfoInCart` | `()` | resets `customerInfoInCart` to `{}` (cart items untouched) |
| `updateCartItemsFromAPI` | `(updatedItems: CartItem[])` | re-keys both maps by the *new* `offerId` returned from an order-preview call, matching old→new entries by SKU (`getSKUFromOfferId`) — see below |

**Computed:** `getTotalItems()` — `Object.values(quantityMap).reduce(sum)`

**Initialization:** lazy, on mount, via `useEffect` reading `localStorage.getItem('bridge_cart')`:
```ts
useEffect(() => {
  const savedCart = loadCartFromLocalStorage();
  setCartItemIdToQuantityMap(savedCart.cartItemIdToQuantityMap);
  setCartItemIdToCartItemDetailsMap(savedCart.cartItemIdToCartItemDetailsMap);
  setCustomerInfoState(savedCart.customerInfoInCart);
  setIsInitialized(true);
}, []);
```
A second `useEffect` writes back to `localStorage` on every state change, gated by `isInitialized` (to avoid overwriting saved data with the empty initial state before the load effect runs).

**Consumer pattern:**
```ts
const { cartItemIdToQuantityMap, addToCart, getTotalItems } = useCart();
// useCart() throws 'useCart must be used within a CartProvider' if called outside the tree
```

**Rules:**
- `offerId` is the map key everywhere; when an order-preview call returns a *different* `offerId` for the same physical item (offerIds can change after preview — see `getSKUFromOfferId`, first 8 chars = SKU), reconciliation is done by SKU match, not by the old key — do not assume `offerId` is stable across a preview round-trip.
- Mixing offers from two different market segments in one cart is a business-rule violation enforced by the *page* (`pages/catalog.tsx::addProductToCart`), not by `CartContext` itself — the context has no market-segment validation.

## 2. PartnerContext

- **File:** `contexts/PartnerContext.tsx`
- **Provider position:** `pages/_app.tsx` — wraps `CartProvider` (outer)

**State shape:** (all derived via `React.useMemo` from one TanStack Query result, not independent `useState`)
```ts
partnerDetails: PartnerDetails | undefined   // { partnerName, marketSegments, currencies }
partnerName: string
availableCurrencies: Currency[]               // partnerDetails.currencies, verbatim
region: string                                // availableCurrencies[0].priceRegion
regionCurrencies: Currency[]                  // availableCurrencies filtered to `region`
marketSegments: string[]                      // dedup'd marketSegment values where programType === 'VIPMP'
isLoading: boolean
error: Error | null
```

**Actions:** none — this context is read-only from the consumer's perspective; its only "action" is the internal `useQuery` refetch, which no consumer triggers manually.

**Initialization:** eager — `useQuery({ queryKey: ['partnerDetails'], queryFn: fetchPartnerDetails, staleTime: 24h, gcTime: 7d, retry: 1 })` fires as soon as `PartnerProvider` mounts (i.e. on every page load, app-wide).

**Consumer pattern:**
```ts
const { partnerName, region, regionCurrencies, isLoading } = usePartnerDetails();
// usePartnerDetails() throws if called outside PartnerProvider
```

**Rules:**
- `region` is derived from `availableCurrencies[0]` — the *first* currency in the array determines the entire app's region for that session. If the backend ever returns currencies from more than one region, only the first one's region is used; `regionCurrencies` then silently filters out the rest.
- Pages must treat `partnerDetails` as possibly `undefined` during the initial load — `pages/catalog.tsx` gates its pricelist query on `isPartnerDetailsReady = !isPartnerDetailsLoading && availableCurrencies.length > 0` rather than trusting `isLoading` alone (a load that resolves with zero currencies is treated as not-ready).

## Initialization Order & Boot Dependency Chain

```
pages/_app.tsx mount
  └─ QueryClientProvider (queryClient = new QueryClient(), no defaults)
       └─ Spectrum Provider (colorScheme="light")
            └─ PartnerProvider          → fires fetchPartnerDetails() immediately
                 └─ CartProvider        → loads localStorage synchronously in an effect
                      └─ <Component/>   → page-level queries fire, most gated on
                                          `enabled: ... && router.isReady` and/or
                                          partner-context values (region, currency)
```

There is no explicit "wait for Partner before Cart" dependency in code — `CartProvider`'s
localStorage load does not read from `PartnerContext` at all. The *page-level* queries are what
actually depend on `PartnerContext` being ready (e.g. `pages/catalog.tsx`'s pricelist query is
gated on `isPartnerDetailsReady`), not the providers themselves.

## Cross-Store Dependencies

| Store / Query | Reads from | Required field | Behavior while waiting |
|---|---|---|---|
| Catalog pricelist query (`pages/catalog.tsx`) | `PartnerContext` | `region`, `regionCurrencies[0].currency` | Query stays `enabled: false`; page shows "Loading partner information..." then "Unable to load partner information" if `isPartnerDetailsReady` never becomes true |
| `pages/checkout.tsx` reseller query | `CartContext.customerInfoInCart.resellerId` | non-empty `resellerId` | Query stays disabled; no reseller name shown in the checkout breadcrumb |
| Renewal/pricing hooks (`useProductPricing`) | `PartnerContext.region` | `region` | Per-offer query disabled until `region` is truthy |

## Storage Key Registry

| Key | Type | Owner | Contents | Persistence | Shape version / migration |
|---|---|---|---|---|---|
| `bridge_cart` | `localStorage` | `contexts/CartContext.tsx` | `{ cartItemIdToQuantityMap, cartItemIdToCartItemDetailsMap, customerInfoInCart }` (JSON) | Indefinite — survives tab/browser close, cleared only by `clearCart()` or manual browser storage clear | No version field or migration logic — a shape change to `CartItem` requires a manual compatibility shim; `loadCartFromLocalStorage()` currently just falls back to `{}` for any of the three top-level keys if missing, but does not validate nested field shapes |

No other `localStorage`/`sessionStorage`/cookie usage was found anywhere in the codebase (grepped
`localStorage.setItem`, `localStorage.getItem`, `sessionStorage.`, `Cookies.set`, `cookie.set`).

## Cache Strategy

Per-domain TanStack Query freshness settings (see `DATA_LAYER.md → Fetch Layer Summary` for the
full per-unit table); grouped here by domain:

| Domain | staleTime | gcTime | Invalidation trigger |
|---|---|---|---|
| Partner details | 24h | 7d | never explicitly invalidated — relies on TTL expiry |
| Reseller/Customer lists & search | 5min | 10min | `invalidateResellers`/`invalidateCustomers` (`utils/queryUtils.ts`) on manual "Try Again" click only |
| Single reseller/customer detail | 0–10min (inconsistent — reseller detail uses 10min, customer detail uses 0) | matches staleTime | customer detail: 3-Year-Commit enrollment success (`customerDetailKeys.customer(id)`) |
| Subscriptions / order history | 0 | 0 | always refetched on mount/tab-focus; order history also explicitly invalidated after a successful return (unit 14 in `DATA_LAYER.md`) |
| Pricelist (catalog + single-offer pricing) | 5–10min | 10–30min | currency change (`invalidateMarketSegmentsForCurrency`) |