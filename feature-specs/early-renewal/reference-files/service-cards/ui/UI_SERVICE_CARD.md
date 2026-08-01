# UI Service Card — Bridge

**Service:** bridge  
**Branch:** develop  
**Commit:** 8d0291b55fc79c5bc048e2a8e165f337c95b1358  
**Framework:** Next.js 15.5.15 (pages router, CSR-only)  
**Port:** 9000 (custom Node HTTP server via `server.js`)

---

## 1. What This UI Owns

Bridge is a full-stack Next.js ordering interface for Adobe VIP Marketplace direct partners. The UI layer owns:

- **Reseller + Customer navigation** — browse, search, paginate resellers and customers
- **Product catalog** — infinite-scroll VIP MP pricelist with market-segment / currency filters and cart management
- **Checkout** — multi-item cart review and new-order placement
- **Customer detail hub** — active subscriptions, account details, purchase history, upgrade paths, renewal editing, 3YC enrollment
- **Edit Renewal dialog** — per-subscription quantity/auto-renewal changes with promo code application and preview pricing
- **Order Confirmation** — post-purchase success screen

The UI talks exclusively to its own BFF layer (`pages/api/*`). No direct Partner API calls from the browser.

---

## 2. Source Structure

| Layer | Path | File count |
|-------|------|-----------|
| UI pages | `pages/` (non-api) | 8 |
| API route handlers (BFF) | `pages/api/` | 10 |
| React components | `components/` | ~50 |
| Client-side hooks | `hooks/` | 8 |
| React contexts (state) | `contexts/` | 2 |
| Utilities | `utils/` | 17 |
| TypeScript types | `types/` | 6 |
| Zod models | `models/` | 12 |
| Styles (CSS modules) | `styles/` | ~20 |
| Tests | `tests/` | 15 |

---

## 3. Routes (Quick Reference)

| Route | Page file | Purpose |
|-------|-----------|---------|
| `/` | `pages/index.tsx` | Redirect → `/resellers` |
| `/resellers` | `pages/resellers.tsx` | Paginated reseller list + search |
| `/customers?resellerId=` | `pages/customers.tsx` | Paginated customer list + search |
| `/customerdetails?resellerId=&customerId=` | `pages/customerdetails.tsx` | Full customer hub (tabs) |
| `/catalog` | `pages/catalog.tsx` | VIP MP product catalog + cart |
| `/checkout` | `pages/checkout.tsx` | Cart review + order placement |
| `/checkout/upgrade` | `pages/checkout/upgrade.tsx` | Offer upgrade / switch flow |
| `/orderConfirmation?...` | `pages/orderConfirmation.tsx` | Post-purchase success screen |

Auth guard: none server-side. Client-side redirects via `useEffect` (e.g. `/customers` redirects to `/resellers` if `resellerId` query param is missing). Cart protection via `useCartRouteGuard` hook.

---

## 4. Data Contracts (Summary)

### BFF Endpoints Consumed by UI

| Endpoint | Method | Purpose | Hook / caller |
|----------|--------|---------|--------------|
| `/api/resellers` | GET | Reseller list, search, detail | `usePaginatedResellersList`, inline `useQuery` |
| `/api/customers` | GET/POST/PATCH | Customer list, search, detail, create, update | `usePaginatedCustomersList`, inline `useQuery` |
| `/api/subscriptions` | GET/PATCH | Customer subscriptions, update auto-renewal | inline `useQuery`, `renewalUtils.updateRenewalSubscription` |
| `/api/orders` | GET/POST | Order history, new order, preview, renewal, switch | `useCreateOrder`, `useCreateOrderPreview`, `usePlaceSwitchOrder`, `renewalUtils.fetchRenewalPreviewAndUpdateProducts` |
| `/api/pricelist` | POST | Paginated VIP MP pricelist | `useInfiniteQuery` (catalog), `useProductPricing` |
| `/api/offer-switch-paths` | GET | Upgrade / switch paths per subscription | `useOfferSwitchPaths` |
| `/api/partnerDetails` | GET | Partner config (name, currencies, segments) | `PartnerContext` |
| `/api/recommendation` | GET | Product recommendations per customer | inline `useQuery` |

### Browser Storage Keys

| Key | Storage | Shape | Owner |
|-----|---------|-------|-------|
| `bridge_cart` | `localStorage` | `{cartItemIdToQuantityMap, cartItemIdToCartItemDetailsMap, customerInfoInCart, showCartModal}` | `CartContext` |

---

## 5. Companion Files

| File | When to Load | Purpose |
|------|-------------|---------|
| `UI_MODULE_INDEX.md` | Impact analysis, feature planning, code generation (load FIRST) | Capability → Page/Component/Hook/Store mapping with Key Decisions |
| `ROUTES.md` | Route planning, auth wiring, navigation code | Full route registry, auth guard pattern, navigation patterns |
| `DATA_LAYER.md` | Feature planning, code generation | Every fetch unit: endpoint, cache key, dependency guard, return shape |
| `STATE_MANAGEMENT.md` | Code generation involving cart or partner config | Store shapes, actions, init order, browser storage keys |
| `UI_CODE_PATTERNS.md` | Code generation | Naming, directory structure, design system usage, form patterns, test skeletons |
| `UI_PLATFORM.md` | Stack decisions, env wiring, build scripts | Runtime stack, env vars, build config, a11y baseline |
