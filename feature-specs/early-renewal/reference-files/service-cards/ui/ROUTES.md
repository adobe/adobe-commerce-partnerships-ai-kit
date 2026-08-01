# Routes — Bridge UI

## 1. Route Registry

| Route | Page File | Params | Auth Required | Pre-load |
|-------|-----------|--------|---------------|---------|
| `/` | `pages/index.tsx` | — | none | — |
| `/resellers` | `pages/resellers.tsx` | — | none | `PartnerContext` must resolve |
| `/customers` | `pages/customers.tsx` | `?resellerId=<id>` | none (client redirect) | — |
| `/customerdetails` | `pages/customerdetails.tsx` | `?resellerId=<id>&customerId=<id>` | none | — |
| `/catalog` | `pages/catalog.tsx` | optional `?customerId=&resellerId=` | none | `PartnerContext` must resolve |
| `/checkout` | `pages/checkout.tsx` | — | none (cart guard) | CartContext populated |
| `/checkout/upgrade` | `pages/checkout/upgrade.tsx` | — | none | — |
| `/orderConfirmation` | `pages/orderConfirmation.tsx` | `?customerId=&currencyCode=&totalAmount=&customerName=&resellerId=&products=` | none | — |

---

## 2. Auth Guard Pattern

**Mechanism:** None enforced at the routing layer. No `middleware.ts`. No `getServerSideProps` redirect.

**Client-side redirect pattern** (used in `/customers`):
```tsx
// pages/customers.tsx
useEffect(() => {
  if (!resellerId && router.isReady) {
    router.push('/resellers');
  }
}, [resellerId, router.isReady, router]);
```

**Cart guard** (`useCartRouteGuard`) — intercepts navigation away from `/checkout` or `/catalog` when the cart has items:
```tsx
// hooks/useCartRouteGuard.ts
router.events.on('routeChangeStart', handler);
// handler throws 'Route change aborted.' to cancel navigation
// renders ClearCartConfirmationDialog if cart non-empty
```
Guard is instantiated in `CheckoutPage` and `CustomerDetailsPage`. It accepts a `shouldAllowNavigation` predicate to whitelist specific navigation targets (e.g. `/orderConfirmation`, same customer's details page).

**Enforcement:** not enforced at server or middleware level — relies on client-side effect and hook.

---

## 3. Navigation Patterns

| Pattern | Where Used |
|---------|-----------|
| `router.push({ pathname, query })` | Navigate to `/customerdetails` from customer card, `/checkout` after cart fill |
| `router.replace('/resellers')` | Index page redirect |
| `Link` component | Not widely used — router.push preferred |
| `NavigationPanel` | Breadcrumb component rendered at top of each page |

**Route parameter access:**
```tsx
const { resellerId, customerId } = router.query;
// Always guard: router.isReady && !!customerId before enabling queries
const customerIdString = typeof customerId === 'string' ? customerId : customerId?.[0] || '';
```

---

## 4. Route-to-Capability Map

| Route | Capability |
|-------|-----------|
| `/resellers` | Reseller Management |
| `/customers` | Customer Management |
| `/customerdetails` (Products tab) | Active Products, Edit Renewal, Upgrade Paths, 3YC |
| `/customerdetails` (Account tab) | Account Details |
| `/customerdetails` (History tab) | Purchase History |
| `/catalog` | Product Catalog + Cart |
| `/checkout` | Checkout |
| `/checkout/upgrade` | Offer Switch / Upgrade |
| `/orderConfirmation` | Order Confirmation |
