
---
service: adobe-commerce-partnerships-ref-app (UI)
owner: NOT CAPTURED — requires org/team ownership metadata not present in this repo
stack: Next.js 15 (Pages Router) + TypeScript, React 18, TanStack Query v5, React Context, Adobe @react-spectrum/s2
rendering: CSR only (no getServerSideProps/getStaticProps anywhere in pages/)
last_updated: 2026-09-24
---

# UI_SERVICE_CARD.md — adobe-commerce-partnerships-ref-app

## 1. What We Do

adobe-commerce-partnerships-ref-app's UI is a Next.js Pages Router single-page application used by Adobe partner-operations
staff to manage the reseller/customer commerce lifecycle: browse resellers and their customers,
view a customer's account details, active subscriptions, and purchase history, enroll customers
in a three-year commit program, manage subscription renewals (including a late-renewal path), and
browse a product catalog to build a cart and place new/return/renewal orders. It is a pure
client-side-rendered app — every data fetch happens after mount via TanStack Query hooks calling
adobe-commerce-partnerships-ref-app's own Next.js API routes, which in turn call the Adobe Commerce Partner API (see the
backend service cards for that side). There is no authentication provider of any kind: no login
page, no session, no token — `UserProfileModal.tsx` is a static placeholder reading "Service
account authenticated," and no route or fetch call is gated on any identity.

## 2. What We Explicitly Do Not Do

- **We do not authenticate end users.** No login flow, session, or identity exists anywhere in this app; every route and API call is reachable by anyone who can load the page.
- **We do not attach credentials to any outbound fetch.** Every client-side `fetch()` call is a plain same-origin request to `/api/*` with no `Authorization` header — credentialing to the upstream Partner API is entirely the backend's responsibility.
- **We do not persist anything server-side from the client.** The only client-owned persistent state is `localStorage` (`bridge_cart`) — everything else is either React Context in-memory state or the TanStack Query cache, both of which reset on a full page reload (except the cart).
- **We do not server-render or statically generate any page.** There is no SEO, no pre-fetched HTML content, and no `getServerSideProps`/`getStaticProps` anywhere — every page is a loading skeleton until client JS hydrates and fires its queries.
- **We do not use a form library, global state library (Redux/Zustand/Jotai), or a router other than Next.js's file-based Pages Router**, despite `react-hook-form` being a declared (unused) dependency.

## 3. Data Contracts

### APIs We Consume

| # | Fetch unit | Method | Endpoint | Purpose | Stability |
|---|---|---|---|---|---|
| 1 | [fetchPartnerDetails](DATA_LAYER.md#1-fetchpartnerdetails-partner-configuration) | GET | `/api/partnerDetails` | Partner name, market segments, currencies, region | STABLE |
| 2 | [fetchResellersPage](DATA_LAYER.md#2-fetchresellerspage-paginated-reseller-list) | GET | `/api/resellers?type=getAllResellers` | Paginated reseller list | STABLE |
| 3 | [searchResellers](DATA_LAYER.md#3-searchresellers-reseller-search) | GET | `/api/search?type=reseller` | Reseller name/ID search | STABLE |
| 4 | [fetchCustomersPage](DATA_LAYER.md#4-fetchcustomerspage-paginated-customer-list) | GET | `/api/customers?type=getAllCustomers` | Paginated customer list for a reseller | STABLE |
| 5 | [searchCustomers](DATA_LAYER.md#5-searchcustomers-customer-search) | GET | `/api/search?type=customer` | Customer name/ID search within a reseller | STABLE |
| 6 | [getResellerDetails](DATA_LAYER.md#6-getresellerdetails-single-reseller-lookup) | GET | `/api/resellers?type=getResellerDetails` | Single reseller lookup by ID | STABLE |
| 7 | [fetchCustomerDetails](DATA_LAYER.md#7-fetchcustomerdetails-single-customer-lookup) | GET | `/api/customers?type=getCustomer` | Single customer lookup by ID | STABLE |
| 8 | [fetchCustomerSubscriptions](DATA_LAYER.md#8-fetchcustomersubscriptions-customers-active-subscriptions) | GET | `/api/subscriptions?type=getCustomerSubscriptions` | Customer's active subscriptions | STABLE |
| 9 | [fetchOrdersHistory](DATA_LAYER.md#9-fetchordershistory-paginated-order-history) | GET | `/api/orders?type=getOrdersHistory` | Paginated order history for a customer | STABLE |
| 10 | [fetchPricelistPage](DATA_LAYER.md#10-fetchpricelistpage-catalog-price-list-page) | POST | `/api/pricelist` | Catalog price list page (infinite scroll) | STABLE |
| 11 | [fetchProductPricing](DATA_LAYER.md#11-fetchproductpricing-single-offer-price-lookup) | POST | `/api/pricelist` | Single-offer price lookup | STABLE |
| 12 | [fetchOrderPreview](DATA_LAYER.md#12-fetchorderpreview-cart-price-preview) | POST | `/api/orders?type=Preview&fetch-price=true` | Cart/order price preview | STABLE |
| 13 | [useCreateOrder](DATA_LAYER.md#13-usecreateorder-create-a-new-order-mutation) | POST | `/api/orders?type=NEW` | Place a new order | STABLE |
| 14 | [useReturnOrder](DATA_LAYER.md#14-usereturnorder-create-a-return-order-mutation) | POST | `/api/orders?type=Return` | Return line items on an existing order | STABLE |
| 15 | [useCreateOrderPreview](DATA_LAYER.md#15-usecreateorderpreview-create-an-order-preview-mutation-unused) | POST | `/api/orders?type=Preview` | Order preview (mutation form) | IN FLUX — declared, never called (unit 12 duplicates it as a plain function) |
| 16 | [createCustomer (inline)](DATA_LAYER.md#16-createcustomer-inline-create-a-new-customer) | POST | `/api/customers?type=createCustomer` | Create a customer from the catalog checkout flow | STABLE |
| 17 | [enroll in 3YC (inline)](DATA_LAYER.md#17-enroll-in-3yc-inline-enroll-customer-in-three-year-commit) | PATCH | `/api/customers?type=updateCustomer` | Invite a customer to enroll in a 3-year commit | STABLE |
| 18 | [fetchRenewalPreviewAndUpdateProducts](DATA_LAYER.md#18-fetchrenewalpreviewandupdateproducts-renewal-price-preview) | POST | `/api/orders?type=PreviewRenewal&fetch-price=true` | Renewal price preview | STABLE |
| 19 | [createRenewalSubscription](DATA_LAYER.md#19-createrenewalsubscription-add-a-new-product-to-renewal) | POST | `/api/subscriptions?type=createSubscription` | Add a new product to a customer's renewal | STABLE |
| 20 | [updateRenewalSubscription](DATA_LAYER.md#20-updaterenewalsubscription-update-subscription-auto-renewal) | PATCH | `/api/subscriptions` | Update an existing subscription's auto-renewal | STABLE |
| 21 | [createRenewalOrder](DATA_LAYER.md#21-createrenewalorder-submit-a-late-renewal-order) | POST | `/api/orders?type=RenewalOrder` | Submit a late-renewal order | STABLE |

### State We Own Client-Side

| Key | Storage | Owner | Contents | TTL |
|---|---|---|---|---|
| `bridge_cart` | `localStorage` | `CartContext` | Cart item quantities/details + selected customer info | Indefinite — cleared only by `clearCart()` |

(See `STATE_MANAGEMENT.md` for the two React Context stores; TanStack Query's in-memory cache is not persisted and is not listed here.)

## 4. Source Structure

| Layer | Path | Files |
|---|---|---|
| Pages | `pages/*.tsx` | 7 routes + `_app`/`_document` |
| Components | `components/**/*.tsx` | 43 |
| Contexts | `contexts/*.tsx` | 2 |
| Hooks | `hooks/*.ts` | 7 |
| Client utils | `utils/*.ts(x)` | 19 |
| Types | `types/*.ts` | 7 |
| Shared models (Zod, also used by backend) | `models/*.ts` | 11 |
| Styles (CSS Modules) | `styles/**/*.css` | 30 |

## 5. Module Index (Summary)

15 capabilities across 7 domains — see [`UI_MODULE_INDEX.md`](UI_MODULE_INDEX.md) for the full catalogue with Key Decisions and call graphs.

| Domain | Capability count |
|---|---|
| Navigation | 1 |
| Reseller Management | 1 |
| Customer Management | 2 |
| Purchase History | 1 |
| Subscriptions & Renewal | 3 |
| Three-Year Commit | 1 |
| Catalog & Cart | 3 |
| Checkout | 2 |
| Cross-cutting (placeholder) | 1 |

## Companion Files

| File | When to Load | Purpose |
|---|---|---|
| [`UI_MODULE_INDEX.md`](UI_MODULE_INDEX.md) | Impact analysis, LLD, code gen (load FIRST) | Capability catalogue, Key Decisions, file paths |
| [`ROUTES.md`](ROUTES.md) | LLD, routing work, code gen | Route registry, auth guard, navigation contracts |
| [`DATA_LAYER.md`](DATA_LAYER.md) | LLD, code gen | Fetch library, cache strategy, mutations, auth injection |
| [`STATE_MANAGEMENT.md`](STATE_MANAGEMENT.md) | LLD, code gen (any state change) | All stores/contexts: shapes, actions, init order, storage keys, cache strategy |
| [`UI_CODE_PATTERNS.md`](UI_CODE_PATTERNS.md) | Code gen | Naming, component patterns, design system, forms, tests |
| [`UI_PLATFORM.md`](UI_PLATFORM.md) | Impact analysis, deployment planning, code gen | Runtime, build scripts, key dependencies, env vars, a11y, monitoring |