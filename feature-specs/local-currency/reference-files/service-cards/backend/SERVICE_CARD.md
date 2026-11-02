# SERVICE_CARD.md — adobe-commerce-partnerships-ref-app

## 1. Identity

| Field | Value |
|---|---|
| Service name | adobe-commerce-partnerships-ref-app (repo: `adobe-commerce-partnerships-ref-app`) |
| What it owns | Nothing persistent. adobe-commerce-partnerships-ref-app is a stateless Next.js backend-for-frontend that validates, shapes, and proxies requests for customers, resellers, orders, subscriptions, price lists, recommendations, and partner configuration between its own UI and the Adobe Commerce Partner API. |
| System of record | Adobe Commerce Partner API for all business data (customers, resellers, orders, subscriptions, price lists); local environment variables for static partner configuration (see `MODULE_INDEX.md → Get Partner Details`). |
| Source branch scanned | `develop` @ `4f8625ebc132191b88f492b75d05df06dcd11b8e` |

## 2. External Surface at a Glance

- 9 physical API routes under `pages/api/`, several multiplexed by a `type` query parameter into ~24 logical capabilities (see `MODULE_INDEX.md`).
- No inbound authentication on the API routes themselves — every route acquires a cached, service-level Adobe IMS OAuth token to authenticate *outbound* to the Partner API.
- Two outbound dependencies total: the Adobe Commerce Partner API (business data) and the Adobe IMS token endpoint (auth) — see `CONNECTORS.md`.
- No async messaging, no database.

## 3. Contracts

### APIs We Expose

| Endpoint | Method | Purpose | Version | Stability |
|---|---|---|---|---|
| [`/api/customers?type=getCustomer`](CONTRACTS.md#get-apicustomers-fetch-a-single-customer-typegetcustomer) | GET | Fetch a single customer | v3 | STABLE |
| [`/api/customers?type=getAllCustomers`](CONTRACTS.md#get-apicustomers-list-customers-for-a-reseller-typegetallcustomers) | GET | List customers for a reseller | v3 | STABLE |
| [`/api/customers?type=createCustomer`](CONTRACTS.md#post-apicustomers-create-a-customer-typecreatecustomer) | POST | Create a customer | v3 | STABLE |
| [`/api/customers?type=updateCustomer`](CONTRACTS.md#patch-apicustomers-update-a-customer-typeupdatecustomer) | PATCH | Update a customer | v3 | STABLE |
| [`/api/orders?type=getOrders`](CONTRACTS.md#get-apiorders-fetch-a-single-order-typegetorders) | GET | Fetch a single order | v3 | STABLE |
| [`/api/orders?type=getOrdersHistory`](CONTRACTS.md#get-apiorders-fetch-order-history-for-a-customer-typegetordershistory) | GET | Fetch order history for a customer | v3 | STABLE |
| [`/api/orders?type=NEW`](CONTRACTS.md#post-apiorders-create-a-new-order-typenew) | POST | Create a new order | v3 | STABLE |
| [`/api/orders?type=Return`](CONTRACTS.md#post-apiorders-returncancel-an-order-typereturn) | POST | Return/cancel an order | v3 | STABLE |
| [`/api/orders?type=Preview`](CONTRACTS.md#post-apiorders-preview-an-order-typepreview) | POST | Preview an order | v3 | STABLE |
| [`/api/orders?type=PreviewRenewal`](CONTRACTS.md#post-apiorders-preview-a-renewal-order-typepreviewrenewal) | POST | Preview a renewal order | v3 | STABLE |
| [`/api/orders?type=RenewalOrder`](CONTRACTS.md#post-apiorders-create-a-renewal-order-typerenewalorder) | POST | Create a renewal order | v3 | STABLE |
| [`/api/search?type=reseller`](CONTRACTS.md#get-apisearch-search-resellers-typereseller) | GET | Search resellers | v3 | STABLE |
| [`/api/search?type=customer`](CONTRACTS.md#get-apisearch-search-customers-typecustomer) | GET | Search customers | v3 | STABLE |
| [`/api/pricelist`](CONTRACTS.md#post-apipricelist-fetch-a-price-list) | POST | Fetch a price list | v3 | STABLE |
| [`/api/recommendation`](CONTRACTS.md#get-apirecommendation-fetch-product-recommendations) | GET | Fetch product recommendations | v3 | IN FLUX — see `MODULE_INDEX.md → Get Recommendations` |
| [`/api/partnerDetails`](CONTRACTS.md#get-apipartnerdetails-fetch-partner-contract-details) | GET | Fetch partner contract details | n/a (env-sourced) | STABLE |
| [`/api/partnerDetails?type=marketSegments`](CONTRACTS.md#get-apipartnerdetailstypemarketsegments-fetch-distinct-market-segments) | GET | Fetch distinct market segments | n/a | STABLE |
| [`/api/partnerDetails?type=currencies`](CONTRACTS.md#get-apipartnerdetailstypecurrencies-fetch-supported-currencies) | GET | Fetch supported currencies | n/a | STABLE |
| [`/api/resellers?type=getResellerDetails`](CONTRACTS.md#get-apiresellers-fetch-reseller-details-typegetresellerdetails) | GET | Fetch reseller details | v3 | STABLE |
| [`/api/resellers?type=getAllResellers`](CONTRACTS.md#get-apiresellers-list-all-resellers-typegetallresellers) | GET | List all resellers | v3 | STABLE |
| [`/api/resellers`](CONTRACTS.md#post-apiresellers-create-a-reseller-typecreatereseller-or-omitted-for-backward-compatibility) | POST | Create a reseller | v3 | IN FLUX — no response validation, see `MODULE_INDEX.md → Create Reseller` |
| [`/api/subscriptions?type=getSubscriptionDetails`](CONTRACTS.md#get-apisubscriptions-fetch-subscription-details-typegetsubscriptiondetails) | GET | Fetch subscription details | v3 | STABLE |
| [`/api/subscriptions?type=getCustomerSubscriptions`](CONTRACTS.md#get-apisubscriptions-list-a-customers-subscriptions-typegetcustomersubscriptions) | GET | List a customer's subscriptions | v3 | STABLE |
| [`/api/subscriptions`](CONTRACTS.md#post-apisubscriptions-create-a-subscription-typecreatesubscription-or-omitted) | POST | Create a subscription | v3 | IN FLUX — no response validation |
| [`/api/subscriptions`](CONTRACTS.md#patch-apisubscriptions-update-auto-renewal-on-a-subscription) | PATCH | Update auto-renewal on a subscription | v3 | IN FLUX — no response validation |
| [`/api/env`](CONTRACTS.md#get-apienv-runtime-public-environment-variables) | GET | Runtime public environment variables | n/a | IN FLUX — allowlist currently empty |

### APIs We Consume

| Service | Connector Interface | Connector Impl | Auth Type | Purpose |
|---|---|---|---|---|
| Adobe Commerce Partner API | n/a — no dedicated connector module; native `fetch()` inlined per controller | `customerController.ts`, `orderController.ts`, `resellerController.ts`, `subscriptionController.ts`, `priceController.ts`, `recommendationController.ts` | Bearer token (IMS) + `x-api-key` | [Customers, resellers, orders, subscriptions, price lists, recommendations](CONNECTORS.md#1-adobe-commerce-partner-api-external-partner-platform-customers-resellers-orders-subscriptions-price-lists-recommendations) |
| Adobe IMS OAuth Token Service | n/a — no dedicated connector module | `utils/imsTokenService.ts::getAccessToken` | OAuth2 client_credentials | [Issues the Bearer token used against the Partner API](CONNECTORS.md#2-adobe-ims-oauth-token-service-client-credentials-token-issuance) |

### Events We Consume

None — omitted (no async consumers).

### Events We Publish

None — omitted (no async publishers).


## Companion Files

| File | When to Load | Purpose |
|---|---|---|
| [`MODULE_INDEX.md`](MODULE_INDEX.md) | Impact analysis, feature planning, code generation (load FIRST) | Capability → implementation mapping: which classes, connectors, events, and flags implement each business capability, with Key Decisions per capability. |
| [`CONTRACTS.md`](CONTRACTS.md) | Feature planning, code generation | The service's full API surface: HTTP operations it exposes and async events it consumes and publishes, with request/response shapes and auth. |
| [`CONNECTORS.md`](CONNECTORS.md) | Feature planning, code generation | Every outbound dependency: auth, transport config, resilience settings, and error-handling posture. |
| [`BUILD_CONFIG.md`](BUILD_CONFIG.md) | Code generation | Runtime stack, dependencies, environment variables, and secrets. |
| [`CODE_PATTERNS.md`](CODE_PATTERNS.md) | Code generation | Team coding conventions: naming, package layout, error handling, validation, test approach. |
| [`PLATFORM.md`](PLATFORM.md) | Impact analysis, deployment planning | Deployment topology, monitoring, feature flags, caching, database, active migrations, runbooks. |

> `DB_SCHEMA.md` is omitted from this table — this service owns no data store (see `DB_SCHEMA.md` for the explicit no-data-store banner).