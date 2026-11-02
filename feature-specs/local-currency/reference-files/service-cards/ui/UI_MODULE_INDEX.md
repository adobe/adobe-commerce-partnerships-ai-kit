# UI_MODULE_INDEX.md — adobe-commerce-partnerships-ref-app (UI)

## Overview

adobe-commerce-partnerships-ref-app's UI is a Next.js Pages Router CSR application that lets an Adobe partner-operations user
browse resellers and their customers, view a customer's active products/subscriptions and
purchase history, browse a product catalog, build a cart, and place new/renewal/return orders
against the Adobe Commerce Partner API (via adobe-commerce-partnerships-ref-app's own backend — see the backend service
cards). There is no end-user authentication anywhere in the app (`UserProfileModal.tsx` shows a
static "Service account authenticated" placeholder with no real identity) — every capability below
is reachable by anyone who can load the app.

## Module Taxonomy

| Layer | Path | Count |
|---|---|---|
| Pages | `pages/*.tsx` (excluding `api/`, `_app`, `_document`) | 7 |
| Components | `components/**/*.tsx` | 43 |
| Contexts | `contexts/*.tsx` | 2 |
| Hooks | `hooks/*.ts` | 7 |
| Utils (client-relevant) | `utils/*.ts(x)` | 19 |
| Types | `types/*.ts` | 7 |
| Models (shared with backend) | `models/*.ts` | 11 |
| Styles | `styles/**/*.css` | 30 CSS Modules |

## Capability Taxonomy

| Domain | Capabilities |
|---|---|
| Navigation | Root Redirect |
| Reseller Management | Browse & Search Resellers |
| Customer Management | Browse & Search Customers, View Customer Details (shell), View Account Details |
| Purchase History | View Purchase History, Return an Order |
| Subscriptions & Renewal | View Active Products & Renewal Overview, Edit Renewal, Late Renewal Submission |
| Three-Year Commit | Enroll Customer in 3YC |
| Catalog & Cart | Browse Product Catalog, Manage Shopping Cart, Find or Create Customer |
| Checkout | Checkout & Place Order, View Order Confirmation |
| Cross-cutting | User Profile (placeholder), Partner Configuration Loading |

## Capability Catalogue

### Root Redirect
- **Page:** `pages/index.tsx`
- **Components:** none (inline JSX)
- **Hierarchy:** `Home`
- **Fetch units:** none
- **Stores:** none
- **Key Decisions:**
  - Rendering: CSR. **Enforcement:** `useEffect(() => router.replace('/resellers'), [router])` — a client-side effect, not a Next.js `redirects()` config entry, so this redirect only fires after JS hydrates.
  - Auth dependency: none.
  - Stability: STABLE.

### Browse & Search Resellers
- **Page:** `pages/resellers.tsx`
- **Components:** `Layout`, `NavigationPanel`, `LoadingSkeleton`, Spectrum `SearchField`/`Button`
- **Hierarchy:** `ResellersPage` → `Layout` → (`NavigationPanel`, reseller card grid, `LoadingSkeleton`)
- **Fetch units:** [#2 fetchResellersPage](DATA_LAYER.md#2-fetchresellerspage-paginated-reseller-list), [#3 searchResellers](DATA_LAYER.md#3-searchresellers-reseller-search) — via `usePaginatedResellersList`
- **Stores:** `PartnerContext` (header partner name only)
- **Key Decisions:**
  - Rendering: CSR. Fail-posture: open — an error still renders the page shell with an inline error message + "Try Again" button, not a redirect or hard crash. **Enforcement:** `pages/resellers.tsx` error branch calls `invalidateResellers(queryClient)`.
  - Auth dependency: none.
  - Stability: STABLE.
- **Call graph:**
  1. `usePaginatedResellersList` fires list or search query based on `isSearchMode`
  2. user clicks a reseller card → `router.push({ pathname: '/customers', query: { resellerId } })`

### Browse & Search Customers
- **Page:** `pages/customers.tsx`
- **Components:** `Layout`, `NavigationPanel`, `LoadingSkeleton`
- **Hierarchy:** `CustomersPage` → `Layout` → (`NavigationPanel`, customer card grid, `LoadingSkeleton`)
- **Fetch units:** [#6 getResellerDetails](DATA_LAYER.md#6-getresellerdetails-single-reseller-lookup) (page header name), [#4 fetchCustomersPage](DATA_LAYER.md#4-fetchcustomerspage-paginated-customer-list), [#5 searchCustomers](DATA_LAYER.md#5-searchcustomers-customer-search) — via `usePaginatedCustomersList`
- **Stores:** none directly (reads `resellerId` from the URL, not from context)
- **Key Decisions:**
  - Rendering: CSR. Auth dependency: none — but has a **route-param dependency**: redirects to `/resellers` if `resellerId` is missing once `router.isReady`. **Enforcement:** `pages/customers.tsx:27-31`.
  - Fail-posture: open (same inline-error + retry pattern as Resellers).
  - Stability: STABLE.
- **Call graph:**
  1. read `resellerId` from `router.query`; redirect to `/resellers` if absent
  2. `useQuery(['reseller', resellerId])` for header name (unit #6)
  3. `usePaginatedCustomersList` for the card grid (units #4/#5)
  4. click a card → `router.push({ pathname: '/customerdetails', query: { resellerId, customerId } })`

### View Customer Details (shell)
- **Page:** `pages/customerdetails.tsx`
- **Components:** `Layout`, `NavigationPanel`, Spectrum `Tabs`/`TabList`/`Tab`/`TabPanel`, `CartDetails`, `ThreeYearCommit`, `ClearCartConfirmationDialog`, `SuccessToast`
- **Hierarchy:** `CustomerDetailsPage` → `Layout` → `Tabs` → (`TabPanel[products]` → `CustomerProductsOverviewPanel`, `TabPanel[account-details]` → `AccountDetailsPanel`, `TabPanel[purchase-history]` → `PurchaseHistoryPanel`)
- **Fetch units:** [#8 fetchCustomerSubscriptions](DATA_LAYER.md#8-fetchcustomersubscriptions-customers-active-subscriptions), [#7 fetchCustomerDetails](DATA_LAYER.md#7-fetchcustomerdetails-single-customer-lookup), [#6 getResellerDetails](DATA_LAYER.md#6-getresellerdetails-single-reseller-lookup)
- **Stores:** `CartContext` (cart button/modal), `PartnerContext` (`regionCurrencies`)
- **Key Decisions:**
  - Rendering: CSR, with an explicit `if (!router.isReady) return <Loading>` gate before any query fires. **Enforcement:** `pages/customerdetails.tsx:147-149`.
  - Auth dependency: none.
  - Fail-posture: split — customer/subscription errors render inline per-section, not a page-level failure.
  - Stability: STABLE.
- **Call graph:**
  1. `router.isReady` gate → read `resellerId`/`customerId` from query
  2. fire subscriptions (#8), customer details (#7), reseller (#6) queries in parallel
  3. subscriptions auto-`refetch()` every 8s and on Products-tab (re)activation
  4. cart button → `showCartModal` (state in `CartContext`) → renders `CartDetails` → "Review Order" → `router.push('/checkout')`

### View Account Details
- **Component:** `components/customerdetails/AccountDetailsPanel.tsx` (+ `AccountDetailsSkeleton`)
- **Hierarchy:** rendered inside "View Customer Details (shell)"'s `account-details` tab
- **Fetch units:** none — pure presentational, consumes the `customer` object already fetched by the parent page
- **Domain:** formatting helpers from `utils/customerDetailsUtils.ts` (`formatAddress`, `formatAnniversaryDate`, `formatDiscountLevel`, `formatAdmins`)
- **Key Decisions:**
  - Rendering: CSR, purely derived from parent state — no independent fetch or loading state of its own beyond the boolean `isLoading` prop passed down.
  - Stability: STABLE.

### View Purchase History & Return an Order
- **Component:** `components/customerdetails/PurchaseHistoryPanel.tsx` (+ `OrderDetailsDialog`, `ReturnOrderDialog`)
- **Hierarchy:** "View Customer Details (shell)" `purchase-history` tab → `PurchaseHistoryPanel` → (on row click) `OrderDetailsDialog`; (on "Return" action) `ReturnOrderDialog`
- **Fetch units:** [#9 fetchOrdersHistory](DATA_LAYER.md#9-fetchordershistory-paginated-order-history) (list), [#11 fetchProductPricing](DATA_LAYER.md#11-fetchproductpricing-single-offer-price-lookup) (product names in the return dialog), [#14 useReturnOrder](DATA_LAYER.md#14-usereturnorder-create-a-return-order-mutation) (mutation)
- **Domain:** `isOrderWithinReturnWindow` / `RETURN_WINDOW_DAYS = 14` (`utils/customerDetailsUtils.ts`) gates which orders show a "Return" action
- **Key Decisions:**
  - Optimistic update: not used — the return mutation waits for `onSuccess` before invalidating and closing.
  - Fail-posture: closed for the mutation itself (button disabled while `isSubmitting`), open for the list (errors render inline, don't block the tab).
  - Stability: STABLE.
- **Call graph:**
  1. `PurchaseHistoryPanel` fetches order history page (#9)
  2. user opens `ReturnOrderDialog` for an order within the return window
  3. dialog fetches product names for line items (#11) and lets the user select which line items to return
  4. `useReturnOrder().mutate(...)` (#14) → on success: toast, 1.5s delay, close dialog, invalidate `['ordersHistory', customerId]`

### View Active Products & Renewal Overview
- **Component:** `components/customerdetails/CustomerProductsOverviewPanel.tsx` (+ `ActiveProducts`, `PersonalizedRecommendations`, `ThreeYearCommitDetailsDialog`, `ViewTermsFor3YCBanner`, `EnrollTo3YCBanner`, `RenewalWindowClosesBanner`, `LateRenewalOrderDialog`, `EditRenewalDialog`)
- **Hierarchy:** "View Customer Details (shell)" `products` tab → `CustomerProductsOverviewPanel` → conditionally renders 3YC banners/dialogs, late-renewal banner/dialog, and the edit-renewal dialog based on customer benefit/subscription state
- **Fetch units:** [#11 fetchProductPricing](DATA_LAYER.md#11-fetchproductpricing-single-offer-price-lookup) (via `useProductNamesFromPricelist`)
- **Domain:** `shouldShow3YCEnrollmentBanner`, `has3YCCommitmentPending`, `isLGACustomer`, `getPriceListType` (`utils/customerDetailsUtils.ts`); `calculateDaysUntilClose` (`utils/renewalOrderUtils.ts`)
- **Key Decisions:**
  - Which banner/dialog shows is entirely derived from `customer.benefits`/`subscriptions` state, not a feature flag. **Enforcement:** `shouldShow3YCEnrollmentBanner`/`has3YCCommitmentPending` functions, called directly in the component's render logic.
  - Stability: STABLE.

### Edit Renewal
- **Component:** `components/renewal/EditRenewalDialog.tsx` (+ `RenewalDialogHeader`, `EstimatedTotal`, `LicenseInfoSection`, `PromoCodeButton`/`PromoCodeDialog`)
- **Hierarchy:** opened from "View Active Products & Renewal Overview"
- **Fetch units:** [#18 fetchRenewalPreviewAndUpdateProducts](DATA_LAYER.md#18-fetchrenewalpreviewandupdateproducts-renewal-price-preview), [#19 createRenewalSubscription](DATA_LAYER.md#19-createrenewalsubscription-add-a-new-product-to-renewal), [#20 updateRenewalSubscription](DATA_LAYER.md#20-updaterenewalsubscription-update-subscription-auto-renewal)
- **Domain:** `utils/renewalUtils.ts` — `convertSubscriptionsToExistingRenewalProducts`, `toggleAutoRenewal`, `applyPromoCode`/`removePromoCode`, `computeDiscountLevelsMap`
- **Key Decisions:**
  - Existing subscriptions (`type='EXISTING'`) use `updateRenewalSubscription` (#20); cart items being added to the renewal (`type='NEW'`) use `createRenewalSubscription` (#19) — two different write paths for what looks like one "save" action. **Enforcement:** `type` discriminant on `RenewalProduct` (`types/AddAndEditCustomerRenewalOrder.ts`), branched in `renewalUtils.ts`.
  - `processSettledResults` (backend-shared `utils/apiError.ts`) is reused client-side here to summarize multiple parallel subscription-update calls (one per changed product) into a single success/failure toast.
  - Stability: STABLE.

### Late Renewal Submission
- **Component:** `components/renewal/lateRenewal/LateRenewalOrderDialog.tsx` (+ `RenewalWindowClosesBanner`)
- **Hierarchy:** opened from "View Active Products & Renewal Overview" when a subscription's renewal window is closing/closed
- **Fetch units:** [#21 createRenewalOrder](DATA_LAYER.md#21-createrenewalorder-submit-a-late-renewal-order)
- **Domain:** `MANUAL_RENEWAL_WINDOW_DAYS = 14` (`utils/constants.ts`), `calculateDaysUntilClose` (`utils/renewalOrderUtils.ts`)
- **Key Decisions:**
  - This is a distinct order-creation path (`ORDER_API_TYPE.RENEWAL_ORDER`) from the normal auto-renewal update path (#20) — used specifically when a subscription has fallen outside its automatic renewal window. **Enforcement:** `renewalOrderUtils.ts:199` builds a `POST /api/orders?type=RenewalOrder` call, distinct from `renewalUtils.ts`'s `PATCH /api/subscriptions`.
  - Stability: STABLE.

### Enroll Customer in 3YC
- **Component:** `components/ThreeYearCommit/ThreeYearCommit.tsx` (+ `EnrollTo3YCBanner`, `ViewTermsFor3YCBanner`, `3YearCommitDetailsDialog`, `LearnMoreDialog`)
- **Hierarchy:** opened from "View Customer Details (shell)" (`onEnrollClick` from `CustomerProductsOverviewPanel`)
- **Fetch units:** [#17 enroll in 3YC (inline)](DATA_LAYER.md#17-enroll-in-3yc-inline-enroll-customer-in-three-year-commit)
- **Domain:** `MIN_LICENSES = 10`, `MIN_CONSUMABLES = 1000` (`utils/threeYearCommitUtils.ts`) — minimum commitment thresholds enforced client-side before the PATCH is even attempted
- **Key Decisions:**
  - Two distinct post-submit flows based on current route: if already on `/customerdetails`, calls `onEnrollSuccess` callback (shows a toast, invalidates the customer-detail query) and closes; otherwise (from catalog/checkout), navigates to `buildCustomerDetailsUrl(...)` and *does not* call `onClose()` first, to keep the loading overlay visible through the navigation. **Enforcement:** `router.pathname === ROUTES.CUSTOMER_DETAILS` branch, `ThreeYearCommit.tsx:117-131`.
  - Not idempotent — no dedupe on repeated "Invite to enroll" clicks beyond the `isDisabled={isEnrolling}` button guard.
  - Stability: STABLE.

### Browse Product Catalog
- **Page:** `pages/catalog.tsx`
- **Components:** `Layout`, `NavigationPanel`, `CatalogFiltersComponent`, `CatalogProductCard`, `CartDetails`, `FindOrCreateCustomer`, `ErrorToast`
- **Hierarchy:** `CatalogPage` → `Layout` → (`CatalogFiltersComponent`, infinite product grid of `CatalogProductCard`, conditionally `CartDetails` modal / `FindOrCreateCustomer` modal)
- **Fetch units:** [#10 fetchPricelistPage](DATA_LAYER.md#10-fetchpricelistpage-catalog-price-list-page) (infinite scroll, via `IntersectionObserver`)
- **Stores:** `PartnerContext` (currency/region/market-segment gating), `CartContext` (`addToCart`, cart modal)
- **Key Decisions:**
  - The pricelist query is disabled until `PartnerContext` is fully ready (`isPartnerDetailsReady && !!region && !!filters.currency`) — a **cross-store dependency** documented in `STATE_MANAGEMENT.md`. **Enforcement:** `pages/catalog.tsx:158`.
  - Business rule enforced at this layer, not in `CartContext`: adding a product from a different market segment than what's already in the cart is rejected with an error toast, not silently allowed. **Enforcement:** `addProductToCart` (`pages/catalog.tsx:71-100`).
  - Background prefetch of the other two market segments after the active one loads (`prefetchOtherMarketSegments`) — a UX optimization, not required for correctness.
  - Stability: STABLE.
- **Call graph:**
  1. wait for `isPartnerDetailsReady`; default `filters.currency` to `regionCurrencies[0]` once, guarded so it never overwrites a user change
  2. `useInfiniteQuery` (#10) fires once `region`/`currency` are known
  3. client-side filter pipeline: discount-level filter (`04`/`T7` only) → type filter → category filter → debounced search filter
  4. `IntersectionObserver` on a sentinel div triggers `fetchNextPage()`
  5. "Add to cart" → `addToCart` (`CartContext`) → cart modal → "Continue" → `FindOrCreateCustomer` modal

### Manage Shopping Cart
- **Component:** `components/catalogCart/CartDetails.tsx`
- **Hierarchy:** rendered as a modal from both `pages/catalog.tsx` and `pages/customerdetails.tsx`
- **Fetch units:** [#12 fetchOrderPreview](DATA_LAYER.md#12-fetchorderpreview-cart-price-preview) (recalculates prices on quantity change)
- **Stores:** `CartContext` (all cart mutations flow through here — see `STATE_MANAGEMENT.md`)
- **Key Decisions:**
  - Optimistic update: yes, at the local-state level — quantity changes update `CartContext` immediately; the price-recalculation call (#12) follows asynchronously and reconciles via `updateCartItemsWithSkuMapping`. **Enforcement:** `utils/cartUtils.ts::updateCartItemsWithSkuMapping`.
  - Stability: STABLE.

### Find or Create Customer
- **Component:** `components/catalogCart/FindOrCreateCustomer.tsx` (+ `ExistingCustomer`, `NewCustomer`, `AddAndEditCustomerRenewalOrderDialog`)
- **Hierarchy:** opened from "Browse Product Catalog"'s cart flow, tabbed between "Existing" and "New" customer selection
- **Fetch units:** [#16 createCustomer (inline)](DATA_LAYER.md#16-createcustomer-inline-create-a-new-customer) (New tab only); Existing tab reuses `ResellerSelector`'s paginated reseller fetch units (#2/#3) plus a customer search (component not fully traced — see `components/common/CustomerSelector.tsx`)
- **Stores:** `CartContext` (`setCustomerInfoInCart`, `clearCustomerInfoInCart`)
- **Key Decisions:**
  - Cross-component imperative handle: `NewCustomer` exposes its internal submit function to the parent via an `onCreateCustomerRef` callback prop (a ref-passing pattern, not `forwardRef`/`useImperativeHandle`), so `FindOrCreateCustomer`'s single "Continue" button can trigger either tab's submit logic. **Enforcement:** `createCustomerRef.current()` call in `handleContinue`.
  - Stability: STABLE.

### Checkout & Place Order
- **Page:** `pages/checkout.tsx`
- **Components:** `Layout`, `NavigationPanel`, `CheckoutReviewDetails`, `CheckoutSummary`, `ErrorToast`, `ClearCartConfirmationDialog`
- **Hierarchy:** `CheckoutPage` → `Layout` → (`CheckoutReviewDetails`, `CheckoutSummary` → `PromoCodeDialog` via `PromoCodeButton`)
- **Fetch units:** [#6 getResellerDetails](DATA_LAYER.md#6-getresellerdetails-single-reseller-lookup), [#12 fetchOrderPreview](DATA_LAYER.md#12-fetchorderpreview-cart-price-preview), [#13 useCreateOrder](DATA_LAYER.md#13-usecreateorder-create-a-new-order-mutation)
- **Stores:** `CartContext` (source of truth for `items`/`customerInfoInCart`)
- **Key Decisions:**
  - Route-leave guard: `useCartRouteGuard` blocks navigation away from checkout (except to `/orderConfirmation` or back to the *same* customer's `/customerdetails`) while the cart is non-empty. **Enforcement:** `shouldAllowNavigation` callback, `pages/checkout.tsx:47-56`.
  - Not idempotent: `placeOrder` generates a fresh `externalReferenceId` per attempt via `generateExternalReferenceId()` — a retried/duplicate click produces a genuinely new order reference, not a deduped one.
  - Fail-posture: closed for placing the order (button reflects `createOrder.isPending`, errors block via toast + `createOrder.reset()`); open for price recalculation (a stale price shown with an error toast, cart still usable).
  - Stability: STABLE.
- **Call graph:**
  1. hydrate `items` from `CartContext` on mount/change
  2. on first render with a customer selected, auto-`handleRecalculate()` once (`initialRecalculationDone` ref guard)
  3. quantity/promo-code changes → `handleRecalculate` → `fetchOrderPreview` (#12) → `updateCartItemsWithSkuMapping` → `updateCartItemsFromAPI` (CartContext)
  4. "Place order" → `useCreateOrder().mutate(...)` (#13) → on success: `clearCart()`, build query params, `router.push('/orderConfirmation?...')`

### View Order Confirmation
- **Page:** `pages/orderConfirmation.tsx`
- **Components:** none (self-contained, does not use `Layout`)
- **Fetch units:** none — every field is read from the URL query string set by "Checkout & Place Order"
- **Key Decisions:**
  - This page has no independent data source at all; reloading it directly (bookmarked URL) still renders correctly as long as the query string is intact, but a bare `/orderConfirmation` with no params renders with fallback placeholder text ("Customer", total `$0.00`). **Enforcement:** none — every field defaults via `|| fallback`.
  - Stability: STABLE.

### User Profile (placeholder)
- **Components:** `components/Layout/Header.tsx`, `components/UserProfileModal.tsx`
- **Fetch units:** none
- **Key Decisions:**
  - Entirely static — the modal body is the literal string "Service account authenticated". No real user identity, session, or profile data exists anywhere in this application. **Enforcement:** none — confirms the no-auth finding in `ROUTES.md`.
  - Stability: STABLE (as a placeholder; not a real feature).

## Cross-Reference: Route → Capability

| Route | Capability |
|---|---|
| `/` | Root Redirect |
| `/resellers` | Browse & Search Resellers |
| `/customers` | Browse & Search Customers |
| `/customerdetails` | View Customer Details (shell) → Account Details / Purchase History+Return / Active Products+Renewal / 3YC Enrollment |
| `/catalog` | Browse Product Catalog → Manage Shopping Cart → Find or Create Customer |
| `/checkout` | Checkout & Place Order |
| `/orderConfirmation` | View Order Confirmation |

## Cross-Reference: Fetch Unit → Capabilities

| Fetch unit | Capabilities that use it |
|---|---|
| #1 fetchPartnerDetails | every page (via `PartnerContext`, app-wide) |
| #2/#3 reseller list/search | Browse & Search Resellers, Find or Create Customer (via `ResellerSelector`) |
| #4/#5 customer list/search | Browse & Search Customers |
| #6 getResellerDetails | Browse & Search Customers, View Customer Details (shell), Checkout & Place Order |
| #7 fetchCustomerDetails | View Customer Details (shell) |
| #8 fetchCustomerSubscriptions | View Customer Details (shell) |
| #9 fetchOrdersHistory | View Purchase History & Return an Order |
| #10 fetchPricelistPage | Browse Product Catalog |
| #11 fetchProductPricing | View Purchase History & Return an Order, View Active Products & Renewal Overview |
| #12 fetchOrderPreview | Manage Shopping Cart, Checkout & Place Order |
| #13 useCreateOrder | Checkout & Place Order |
| #14 useReturnOrder | View Purchase History & Return an Order |
| #15 useCreateOrderPreview | none (unused) |
| #16 createCustomer | Find or Create Customer |
| #17 enroll in 3YC | Enroll Customer in 3YC |
| #18 fetchRenewalPreviewAndUpdateProducts | Edit Renewal |
| #19 createRenewalSubscription | Edit Renewal |
| #20 updateRenewalSubscription | Edit Renewal |
| #21 createRenewalOrder | Late Renewal Submission |

## Cross-Reference: Store → Capabilities It Gates

| Store | Capabilities |
|---|---|
| `CartContext` | Manage Shopping Cart, Find or Create Customer, Checkout & Place Order, View Customer Details (shell) (cart button/modal) |
| `PartnerContext` | Browse Product Catalog (hard-gates the pricelist query), every other page (header display + currency/region derivation) |

## Cross-Reference: Browser Storage Keys

| Key | Storage type | Set by | Read by |
|---|---|---|---|
| `bridge_cart` | `localStorage` | `CartContext` (every cart/customer-info state change) | `CartContext` (on mount, to hydrate initial state) |

## Cross-Cutting Modules

| Module | Purpose | Used by |
|---|---|---|
| `components/Layout/Layout.tsx` (+ `Header`, `Sidebar`) | App shell: logo, user-profile trigger, sidebar navigation | every page except `/` and `/orderConfirmation` |
| `components/NavigationPanel.tsx` | Manual breadcrumb trail | `/resellers`, `/customers`, `/customerdetails`, `/checkout`, `/catalog` |
| `components/LoadingSkeleton.tsx` | Placeholder cards during initial list load | `/resellers`, `/customers`, `ResellerSelector` |
| `utils/ToastMessageUtils.tsx` (`ErrorToast`/`SuccessToast`/`WarningToast`) + `hooks/useToastState.ts` | Transient notification UI | nearly every capability with a mutation |
| `hooks/useCartRouteGuard.ts` | Blocks in-app navigation away from an in-progress cart | `/customerdetails`, `/checkout` |
| `contexts/CartContext.tsx`, `contexts/PartnerContext.tsx` | Global client state (see `STATE_MANAGEMENT.md`) | app-wide |

## Type Inventory

| Kind | Count |
|---|---|
| Pages | 7 (+ `_app`, `_document`) |
| Stores/Contexts | 2 |
| Fetch units (queries + mutations + imperative calls) | 21 |
| Reusable components (`components/common/*`, `components/Layout/*`, `NavigationPanel`, `LoadingSkeleton`) | 9 |
| Form-bearing components (manual validation, no form library) | 3 (`NewCustomer`, `ThreeYearCommit`, `PromoCodeDialog`) |
| Util files (client-relevant) | 19 |
| Shared Zod model files (imported from backend `models/`) | 11 |
| Plain TS type files (`types/*.ts`) | 7 |

## Forbidden / Do-Not-Add Rules

- **Do not add `react-hook-form` to a new form.** It's a declared dependency but is unused everywhere in this codebase — every existing form uses manual `useState` + per-field validator functions (`UI_CODE_PATTERNS.md → B.6`). Introducing it in one new form would create a third, inconsistent form pattern alongside the two that already exist (manual `isInvalid`/`errorMessage`, and Spectrum's `validate` prop).
- **Do not add a global state library** (Redux/Zustand/Jotai) for new cross-page state — extend `CartContext`/`PartnerContext` or add a new Context following their exact shape (state + actions object, `useContext` hook that throws outside its provider).
- **Do not attach an `Authorization` header or any credential to a client-side `fetch` call.** No fetch unit in this app does this; adobe-commerce-partnerships-ref-app's backend supplies its own credential. Adding one would be inconsistent with every existing fetch unit and would not correspond to anything the backend expects from the client.
- **Do not assume a dynamic Next.js route segment (`[id].tsx`) is the convention for IDs.** Every ID in this app travels via a URL *query* param (`?customerId=...`), never a path segment — see `ROUTES.md`.