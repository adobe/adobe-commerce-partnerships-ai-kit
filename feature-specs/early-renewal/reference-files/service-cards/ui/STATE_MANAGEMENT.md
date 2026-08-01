# State Management — Bridge UI

## Overview

Bridge uses two React Contexts for client-side state. There is no Zustand, Redux, or Jotai. TanStack Query manages server-state cache globally via a single `QueryClient` instance.

| Store | File | Scope | Browser storage |
|-------|------|-------|----------------|
| CartContext | `contexts/CartContext.tsx` | Global (app-wide) | `localStorage` — key: `bridge_cart` |
| PartnerContext | `contexts/PartnerContext.tsx` | Global (app-wide) | None |
| QueryClient | `pages/_app.tsx` | Global (module-level singleton) | None |

Provider hierarchy (from `pages/_app.tsx`):
```
QueryClientProvider
  └─ Provider (Spectrum, colorScheme: light)
       └─ PartnerProvider
            └─ CartProvider
                 └─ <Component />
```

---

## 1. CartContext

**File:** `contexts/CartContext.tsx`  
**Browser storage key:** `bridge_cart` (localStorage)

### State Shape

```ts
interface CartState {
  cartItemIdToQuantityMap: { [offerId: string]: number };
  cartItemIdToCartItemDetailsMap: { [offerId: string]: CartItemDetails };
  customerInfoInCart: { customerId?: string; resellerId?: string; customerName?: string };
  showCartModal: boolean;
}
```

`CartItemDetails` fields (per offerId):
```ts
{
  offerId: string;
  productName: string;
  price: number;
  currency: string;
  marketSegment: string;
  productFamily: string;
}
```

### Actions

| Action | Signature | Effect |
|--------|-----------|--------|
| `addToCart` | `(offerId, productName, price, currency, marketSegment, productFamily, quantity)` | Adds / increments item; sets `showCartModal: true` |
| `removeFromCart` | `(offerId)` | Removes item from both maps |
| `updateQuantity` | `(offerId, quantity)` | Updates quantity; removes if `quantity <= 0` |
| `clearCart` | `()` | Resets all cart state and clears `bridge_cart` from localStorage |
| `setCustomerInfoInCart` | `(info)` | Sets customer linked to cart |
| `clearCustomerInfoInCart` | `()` | Clears `customerInfoInCart` |
| `updateCartItemsFromAPI` | `(apiItems)` | Merges server-returned item data (prices, SKUs) into cart |
| `getTotalItems` | `() => number` | Returns sum of all quantities |
| `setShowCartModal` | `(show: boolean)` | Toggles cart sidebar |

### Initialization

```tsx
// On mount — reads localStorage
useEffect(() => {
  const saved = localStorage.getItem('bridge_cart');
  if (saved) setState(JSON.parse(saved));
}, []);

// On every state change — persists to localStorage
useEffect(() => {
  localStorage.setItem('bridge_cart', JSON.stringify(state));
}, [state]);
```

### Dependencies

None (no external stores or contexts read).

---

## 2. PartnerContext

**File:** `contexts/PartnerContext.tsx`  
**Browser storage:** None

### State Shape (Exposed)

```ts
interface PartnerContextValue {
  partnerDetails: PartnerDetails | null;
  partnerName: string;
  availableCurrencies: string[];
  region: string;
  regionCurrencies: Array<{ currency: string; region: string }>;
  marketSegments: string[];
  isLoading: boolean;
  error: Error | null;
}
```

### Initialization

Uses `useQuery` internally:
```ts
queryKey: ['partnerDetails']
queryFn: fetch('/api/partnerDetails').then(r => r.json())
staleTime: 24 * 60 * 60 * 1000  // 24 hours
gcTime: 7 * 24 * 60 * 60 * 1000 // 7 days
retry: 1
```

Loaded automatically when `PartnerProvider` mounts. No explicit trigger needed.

### Computed Values

| Field | Derived from |
|-------|-------------|
| `partnerName` | `partnerDetails.partnerName` or env `PARTNER_NAME` |
| `availableCurrencies` | `partnerDetails.currencies` array |
| `region` | `partnerDetails.region` |
| `regionCurrencies` | derived from `region` and `currencies` — currencies valid for the configured region |
| `marketSegments` | `partnerDetails.marketSegments` array |

### Dependencies

None (read `QueryClient` via `useQueryClient`).

---

## 3. QueryClient (TanStack Query)

**File:** `pages/_app.tsx`

```tsx
const queryClient = new QueryClient();
// module-level — single instance, persists across page navigations
```

No custom defaults. Individual queries set their own `staleTime`, `gcTime`, and `retry` values.

`ReactQueryDevtools` is included in development builds.

---

## 4. Browser Storage Key Registry

| Key | Storage | Shape | Set by | Read by | Cleared by |
|-----|---------|-------|--------|---------|-----------|
| `bridge_cart` | `localStorage` | `CartState` (JSON) | `CartContext` (every state change) | `CartContext` (on mount) | `CartContext.clearCart()` |

No other localStorage, sessionStorage, or cookie keys managed by the UI.
