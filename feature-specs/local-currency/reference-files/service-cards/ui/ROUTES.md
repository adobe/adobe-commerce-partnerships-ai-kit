# ROUTES.md — adobe-commerce-partnerships-ref-app (UI)

## Overview

| Field | Value |
|---|---|
| Router | Next.js 15 Pages Router — file-based, `pages/*.tsx` |
| Auth guard mechanism | None detected — no `middleware.ts`, no route wrapper/HOC, no `isAuthenticated`/`accessToken` check anywhere in `pages/` or `components/` |
| Auth guard file | n/a |
| Root redirect | `pages/index.tsx` → `router.replace('/resellers')` on mount (client-side redirect, not a Next.js redirect config) |
| Unauthenticated redirect | n/a — no concept of an authenticated session exists client-side |
| Layout hierarchy | `pages/_app.tsx` wraps every page in `QueryClientProvider` → Spectrum `Provider` (`colorScheme="light"`) → `PartnerProvider` → `CartProvider` → page `Component`. Each page then renders its own `<Layout activePage=... partnerName=...>` (`components/Layout/Layout.tsx`), which renders `Header` + `Sidebar` + the page's children. |

## Route Registry

| Path | File | Path params | Query params | Auth required | Pre-load requirements |
|---|---|---|---|---|---|
| `/` | `pages/index.tsx` | none | none | No | none — immediately redirects |
| `/resellers` | `pages/resellers.tsx` | none | none (search is local component state, not a query param) | No | `usePartnerDetails()` for header partner name (non-blocking) |
| `/customers` | `pages/customers.tsx` | none | `resellerId` (required — redirects to `/resellers` via `router.push` if absent once `router.isReady`) | No | `resellerId` must be present in the URL |
| `/customerdetails` | `pages/customerdetails.tsx` | none | `resellerId`, `customerId` (both read via `router.query`, no redirect guard if absent — page renders with fallback empty strings) | No | waits on `router.isReady` before rendering (`if (!router.isReady) return <Loading>`) |
| `/catalog` | `pages/catalog.tsx` | none | none (filters are local state) | No | `usePartnerDetails()` — the pricelist `useInfiniteQuery` is gated `enabled: isPartnerDetailsReady && !!region && !!filters.currency` |
| `/checkout` | `pages/checkout.tsx` | none | none | No | Reads `customerInfoInCart` from `CartContext` (populated by `/catalog` → `FindOrCreateCustomer` flow); no hard redirect if empty — renders "No items in cart." |
| `/orderConfirmation` | `pages/orderConfirmation.tsx` | none | `customerId`, `currencyCode`, `totalAmount`, `customerName`, `resellerId`, `products` (JSON-encoded array) — all passed by the caller (`checkout.tsx`) via `router.push` with a `URLSearchParams` query string | No | none |
| n/a (framework files) | `pages/_app.tsx`, `pages/_document.tsx` | — | — | — | Root providers and HTML document shell, not routable pages |

## Auth Guard Pattern

No authentication or authorization guard exists anywhere in this codebase — every route is
publicly reachable by URL, and every API call from the client relies on the backend's own
service-level credential (see the backend `PLATFORM.md`/`SERVICE_CARD.md` fragile-areas notes: the
Next.js API routes accept any caller and authenticate themselves to the upstream Partner API).
There is no `isAuthenticated`, `accessToken`, or session check pattern to copy for new pages.

**Rule for new pages:** do not assume an auth context exists — none is wired up. If auth is later
introduced, it does not yet have a convention in this codebase to follow.

## Navigation Patterns

**Programmatic navigation** — every page uses `next/router`'s `useRouter()` and calls
`router.push(...)` or `router.replace(...)` directly (no navigation abstraction/wrapper hook):

```tsx
router.push({
  pathname: '/customerdetails',
  query: { resellerId: resellerId, customerId: customer.customerId },
});
```

**State-passing strategy across routes:**

| From → To | Mechanism | Example |
|---|---|---|
| `/resellers` → `/customers` | URL query param | `?resellerId=<id>` |
| `/customers` → `/customerdetails` | URL query param | `?resellerId=<id>&customerId=<id>` |
| `/catalog` → `/checkout` | `CartContext` (localStorage-backed), not URL | `customerInfoInCart` set by `FindOrCreateCustomer` before navigating |
| `/checkout` → `/orderConfirmation` | URL query params (JSON-stringified `products` array embedded in the query string) | `router.push(\`/orderConfirmation?${queryParams.toString()}\`)` |
| `/customerdetails` → `/catalog` | Plain `next/link` (`<Link href="/catalog">Browse the product catalog</Link>`) | no state passed |

**Back-navigation / breadcrumbs:** `components/NavigationPanel.tsx` renders a manual breadcrumb
trail (not browser back) built per-page as an `items: { label, href }[]` array; clicking a
non-last item calls `router.push(href)`. It renders nothing (`return null`) if `items.length <= 1`.

**Route-leave guard:** `hooks/useCartRouteGuard.ts` listens to the Next.js Router's
`routeChangeStart` event. If the cart is non-empty and the target URL doesn't match
`shouldAllowNavigation(url)`, it throws `'Route change aborted.'` to cancel the navigation and
opens `ClearCartConfirmationDialog` instead. Used on `/customerdetails` and `/checkout`.

## Route Constants

Inline strings — no route-constants file for the pages above. The one exception is
`utils/constants.ts`:
```ts
export const ROUTES = { CUSTOMER_DETAILS: '/customerdetails' } as const;
export const buildCustomerDetailsUrl = (customerId: string, resellerId: string): string =>
  `${ROUTES.CUSTOMER_DETAILS}?customerId=${customerId}&resellerId=${resellerId}`;
```
This is used only by the 3-Year-Commit enrollment flow (`components/ThreeYearCommit/ThreeYearCommit.tsx`) to navigate back to a customer's detail page after enrollment. Every other cross-page `router.push` call in the codebase inlines its path string and query object directly.

## Error Routes

| Case | Component / Behavior |
|---|---|
| 404 (unknown path) | Next.js default 404 page — no custom `pages/404.tsx` exists |
| Unhandled render error | No custom `pages/_error.tsx` and no `ErrorBoundary` component found anywhere in the codebase — a render-time exception in any page will surface Next.js's default error overlay (dev) or a blank/broken page (prod) |
| Auth failure | n/a — no auth exists |
| Access denied | n/a — no authorization exists |
| Per-request data-fetch failure | Handled locally per page/component — inline `{error ? <div>...</div> : ...}` blocks with a "Try Again" button that calls `queryClient.invalidateQueries(...)` (see `DATA_LAYER.md → Error Handling Posture`) |

## Notes for Code Generation

- No auth guard exists — do not add one to a new page unless explicitly asked; there's no existing pattern to match.
- Canonical param names already in use: `resellerId`, `customerId`, `subscriptionId`, `orderId` — reuse these exact names for consistency; do not introduce new casings (e.g. `reseller_id`).
- IDs are passed exclusively via URL query params (`router.query`), never path params — this app uses no dynamic route segments (no `[id].tsx` files anywhere in `pages/`).
- Every page reads query params defensively: `Array.isArray(x) ? x[0] : x` or `typeof x === 'string' ? x : x?.[0] || ''`, because Next.js query values can be `string | string[] | undefined`. Follow this pattern for any new page reading `router.query`.
- Every page wraps its content in `<Layout activePage="..." partnerName="...">`; `ActivePage` is a closed union (`'resellers' | 'catalog' | 'customers' | 'checkout'`, `components/Layout/Sidebar.tsx:7`) — note that `/customerdetails` and `/orderConfirmation` reuse `'customers'`/no-Layout respectively rather than having their own sidebar entries. A new top-level page needing its own sidebar entry must extend this union and add an entry to `Sidebar.tsx`'s `navigationItems`.