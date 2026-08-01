# UI Module Index — Bridge

## 1. Capability Catalogue

---

### 1.1 Reseller List

| Field | Value |
|-------|-------|
| Domain | Reseller Management |
| Route | `/resellers` |
| Page | `pages/resellers.tsx` |
| Component hierarchy | `ResellersPage` → `Layout` → `NavigationPanel` + `LoadingSkeleton` |
| Fetch units | `usePaginatedResellersList` (list + search, 300ms debounce) |
| Contexts | `PartnerContext` (partner name for header) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR — no `getServerSideProps`
- **Auth dependency:** none
- **Optimistic update:** not used
- **Fail-posture:** open — error rendered inline with retry button
- **Stability:** STABLE

**Notes:** Prefetches next and prev pages on every render. Search mode uses `['resellers','search',term,page,limit]` cache key.

---

### 1.2 Customer List

| Field | Value |
|-------|-------|
| Domain | Customer Management |
| Route | `/customers?resellerId=` |
| Page | `pages/customers.tsx` |
| Component hierarchy | `CustomersPage` → `Layout` → `NavigationPanel` + `LoadingSkeleton` |
| Fetch units | `usePaginatedCustomersList`, inline `useQuery` for reseller detail (breadcrumb) |
| Contexts | none (resellerId from URL) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR
- **Auth dependency:** `resellerId` URL param — redirects to `/resellers` if missing
- **Optimistic update:** not used
- **Fail-posture:** open — error inline with retry
- **Stability:** STABLE
- **Enforcement:** `useEffect` redirect — not enforced at router/middleware level

---

### 1.3 Customer Detail Hub

| Field | Value |
|-------|-------|
| Domain | Customer Management |
| Route | `/customerdetails?resellerId=&customerId=` |
| Page | `pages/customerdetails.tsx` |
| Component hierarchy | `CustomerDetailsPage` → `Layout` → `Tabs` → `CustomerProductsOverviewPanel` / `AccountDetailsPanel` / `PurchaseHistoryPanel` |
| Fetch units | `customerDetailKeys.customer`, `customerDetailKeys.subscriptions`, `customerDetailKeys.reseller`, `useQuery` for recommendation |
| Contexts | `PartnerContext` (regionCurrencies), `CartContext` (cart state, modal) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR — `staleTime: 0` on customer and subscription queries (always re-fetches)
- **Auth dependency:** `customerId` URL param; no redirect guard
- **Optimistic update:** not used
- **Fail-posture:** open — error inline per tab
- **Stability:** STABLE
- **Cart guard:** `useCartRouteGuard` instantiated here — blocks navigation away if cart has items
- **Auto-refetch:** delayed `setTimeout(() => refetch(), 8000)` after mount to pick up API-side propagation

---

### 1.4 Active Products & Subscriptions

| Field | Value |
|-------|-------|
| Domain | Customer Management |
| Route | `/customerdetails` (Products tab) |
| Page | `pages/customerdetails.tsx` |
| Component hierarchy | `CustomerProductsOverviewPanel` → `ActiveProducts` → `UpgradePathsDialog` |
| Fetch units | `useOfferSwitchPaths` (per subscription), `useProductNamesFromPricelist` |
| Contexts | `PartnerContext` (regionCurrencies), `CartContext` (add-to-cart for re-orders) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR
- **Fail-posture:** open — switch path availability shown conditionally; no hard block
- **Stability:** STABLE

---

### 1.5 Edit Renewal

| Field | Value |
|-------|-------|
| Domain | Subscription Management |
| Route | Dialog on `/customerdetails` Products tab |
| Page | `pages/customerdetails.tsx` → `CustomerProductsOverviewPanel` |
| Component hierarchy | `EditRenewalDialog` → `RenewalDialogHeader` + `RenewalProductCard` + `PromoCodeButton` → `PromoCodeDialog` |
| Fetch units | `useProductNamesFromPricelist`; imperative: `fetchRenewalPreviewAndUpdateProducts` (POST preview on save), `updateRenewalSubscription` (PATCH per subscription on save) |
| Contexts | `PartnerContext` (not directly — offerId/currency sourced from subscriptions) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR, dialog overlay
- **Renewal preview trigger:** fired on save (not on each promo code apply/remove)
- **Promo code:** stored in component state as `product.discountCode`; clearing code resets price to 0 until preview re-runs
- **Save flow:** `Promise.allSettled` over per-subscription PATCH calls; partial failure surfaced as warning toast
- **Fail-posture:** open — partial failures surfaced via toast; successful subscriptions saved
- **Stability:** STABLE

---

### 1.6 Product Catalog

| Field | Value |
|-------|-------|
| Domain | Order Management |
| Route | `/catalog` |
| Page | `pages/catalog.tsx` |
| Component hierarchy | `CatalogPage` → `Layout` → `CatalogFiltersComponent` + `CatalogProductCard` + `CartDetails` + `FindOrCreateCustomer` |
| Fetch units | `useInfiniteQuery` (pricelist — `POST /api/pricelist`) |
| Contexts | `PartnerContext` (availableCurrencies, region), `CartContext` (add, update, remove items) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR, infinite scroll
- **Market segment guard:** cannot mix products from different market segments in same cart — enforced in `addProductToCart`
- **Cart guard:** `useCartRouteGuard` via `CartContext` (items persist across page navigations within same session)
- **Prefetch:** `prefetchAllMarketSegments` on first load; `prefetchOtherMarketSegments` on filter change
- **Fail-posture:** open — error toast for add-to-cart failures
- **Stability:** STABLE

---

### 1.7 Checkout

| Field | Value |
|-------|-------|
| Domain | Order Management |
| Route | `/checkout` |
| Page | `pages/checkout.tsx` |
| Component hierarchy | `CheckoutPage` → `Layout` → `CheckoutReviewDetails` + `CheckoutSummary` |
| Fetch units | `useCreateOrder` (mutation — POST `/api/orders?type=NEW`); imperative: `fetchOrderPreview` (POST preview during recalculation) |
| Contexts | `CartContext` (items, customer info, clear after order) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR
- **Order preview:** auto-triggered on mount and on quantity change to compute estimated total
- **Cart guard:** `useCartRouteGuard` — only allows navigation to `/orderConfirmation` or back to same customer's details
- **Post-order:** clears cart, navigates to `/orderConfirmation?...` with order summary in query params
- **Fail-posture:** closed on order failure — error toast, stays on checkout page
- **Stability:** STABLE

---

### 1.8 Three Year Commit (3YC)

| Field | Value |
|-------|-------|
| Domain | Customer Management |
| Route | Dialog/banner on `/customerdetails` Products tab |
| Page | `pages/customerdetails.tsx` |
| Component hierarchy | `CustomerProductsOverviewPanel` → `EnrollTo3YCBanner` / `ViewTermsFor3YCBanner` → `ThreeYearCommit` → `3YearCommitDetailsDialog` |
| Fetch units | inline PATCH `/api/customers` (on enrollment) |
| Contexts | `PartnerContext` (marketSegments for eligibility check) |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR, dialog overlay
- **Eligibility:** computed client-side from `customer.benefits` array — no separate API call
- **Stability:** STABLE

---

### 1.9 Order Confirmation

| Field | Value |
|-------|-------|
| Domain | Order Management |
| Route | `/orderConfirmation` |
| Page | `pages/orderConfirmation.tsx` |
| Fetch units | none — all data passed via query params |
| Contexts | none |
| Feature flags | none |

**Key Decisions:**
- **Rendering:** CSR — data from URL query params only (no API calls)
- **Stability:** STABLE

---

## 2. Module Taxonomy

| Layer | Path | Files | Notes |
|-------|------|-------|-------|
| Pages | `pages/` | 8 UI + 10 API | API routes are the BFF layer |
| Contexts | `contexts/` | 2 | CartContext, PartnerContext |
| Hooks | `hooks/` | 8 | TanStack Query wrappers + utility hooks |
| Components — layout | `components/Layout/` | 4 | Header, Sidebar, Layout, index |
| Components — customerdetails | `components/customerdetails/` | 9 | Per-tab panels, dialogs |
| Components — renewal | `components/renewal/` | 5 + `lateRenewal/` | EditRenewalDialog + sub-components |
| Components — catalog/cart | `components/catalogCart/` | ~4 | CartDetails, FindOrCreateCustomer |
| Components — checkout | `components/checkout/` | ~4 | CheckoutReviewDetails, CheckoutSummary |
| Components — 3YC | `components/ThreeYearCommit/` | 5 | Banners, dialogs |
| Components — common | `components/common/` | ~5 | PromoCodeButton, PromoCodeDialog, EstimatedTotal, LicenseInfoSection |
| Utils | `utils/` | 17 | renewalUtils, cartUtils, customerDetailsUtils, queryUtils, AccountApis, etc. |
| Models | `models/` | 12 | Zod schemas |
| Types | `types/` | 6 | TypeScript interfaces |
| Tests | `tests/` | 15 | 14 integration + 1 unit dir |

---

## 3. Cross-Reference Indexes

### Hook → Endpoint

| Hook | BFF Endpoint | HTTP |
|------|-------------|------|
| `usePaginatedResellersList` | `/api/resellers` | GET |
| `usePaginatedCustomersList` | `/api/customers` | GET |
| `useProductPricing` / `useProductNamesFromPricelist` | `/api/pricelist` | POST |
| `useOfferSwitchPaths` | `/api/offer-switch-paths` | GET |
| `useCreateOrder` | `/api/orders?type=NEW` | POST |
| `useCreateOrderPreview` | `/api/orders?type=Preview` | POST |
| `usePreviewSwitchOrder` | `/api/orders?type=PREVIEW_SWITCH` | POST |
| `usePlaceSwitchOrder` | `/api/orders?type=SWITCH` | POST |
| `PartnerContext` | `/api/partnerDetails` | GET |

### Component → Context

| Component | CartContext | PartnerContext |
|-----------|-----------|---------------|
| `CatalogPage` | read + write | read |
| `CustomerDetailsPage` | read + write | read |
| `CheckoutPage` | read + clear | — |
| `ActiveProducts` | write (add-to-cart) | read |
| `RenewalDialog` | — | — |

---

## 4. Cross-Cutting Modules

| Module | File | Purpose | Scope |
|--------|------|---------|-------|
| `Layout` | `components/Layout/Layout.tsx` | Header + Sidebar wrapper | All main pages (not checkout/orderConfirmation) |
| `Header` | `components/Layout/Header.tsx` | Top nav bar with partner name | Global |
| `Sidebar` | `components/Layout/Sidebar.tsx` | Left nav with active page highlight | Global |
| `NavigationPanel` | `components/NavigationPanel.tsx` | Breadcrumb navigation | Per-page |
| `LoadingSkeleton` | `components/LoadingSkeleton.tsx` | Loading placeholder card grid | Per-list page |
| `ErrorToast` / `SuccessToast` / `WarningToast` | `utils/ToastMessageUtils.tsx` | Toast notification components | Per-component (not global) |
| `useToastState` | `hooks/useToastState.ts` | Local toast state management | Per-component |
| `ClearCartConfirmationDialog` | `components/customerdetails/ClearCartConfirmationDialog.tsx` | Cart clear confirmation | Used by `useCartRouteGuard` |
| `createClientLogger` | `utils/logger.ts` | Pino browser-side logger | Per-page / per-component |

No global `ErrorBoundary` or `Suspense` boundaries detected.
