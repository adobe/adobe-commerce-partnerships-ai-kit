# Service Card — Bridge

**Service:** bridge  
**Branch:** develop  
**Commit:** 8d0291b55fc79c5bc048e2a8e165f337c95b1358  
**Runtime:** Node.js / Next.js 15 (TypeScript)  
**Port:** 9000 (custom Node HTTP server via `server.js`)

---

## 1. What This Service Owns

Bridge is a full-stack Next.js ordering interface for Adobe VIP Marketplace direct partners. It owns no data of its own — it is a stateless proxy and orchestration layer that:

- Authenticates with Adobe's IMS service and forwards requests to the VIP Marketplace Partner API
- Provides a React UI for partner workflows: customer management, ordering, renewals, subscriptions, and pricing
- Exposes Next.js API routes (`pages/api/`) as the BFF (Backend-For-Frontend) layer between the UI and the external Partner API

---

## 2. Source Structure

| Layer | Path | File count |
|-------|------|-----------|
| API route handlers | `pages/api/` | 10 |
| UI pages | `pages/` (non-api) | 8 |
| Controllers (business logic) | `controllers/` | 8 |
| Zod models / schemas | `models/` | 12 |
| TypeScript types | `types/` | 6 |
| Utilities | `utils/` | 17 |
| Client-side hooks | `hooks/` | 8 |
| React components | `components/` | ~30 |
| Tests | `tests/` | 15 (integration: 14, unit dir: 1) |

---

## 3. Contracts

### APIs We Expose

| Endpoint | Method | Purpose | Stability |
|----------|--------|---------|-----------|
| [`/api/customers`](#get-apicustomers--get-customer--get-all-customers) | GET | Fetch single customer or paginated customer list for a reseller | STABLE |
| [`/api/customers`](#post-apicustomers--create-customer) | POST | Create a new customer | STABLE |
| [`/api/customers`](#patch-apicustomers--update-customer) | PATCH | Update customer (e.g. 3YC benefits, company profile) | STABLE |
| [`/api/orders`](#get-apiorders--get-order--get-orders-history) | GET | Fetch single order or paginated order history | STABLE |
| [`/api/orders`](#post-apiorders--create-order) | POST | Create order (NEW / RETURN / PREVIEW / PREVIEW_RENEWAL / RENEWAL / SWITCH / PREVIEW_SWITCH) | STABLE |
| [`/api/subscriptions`](#get-apisubscriptions--get-subscriptions) | GET | Fetch customer subscriptions or single subscription detail | STABLE |
| [`/api/subscriptions`](#post-apisubscriptions--create-subscription) | POST | Create a subscription | STABLE |
| [`/api/subscriptions`](#patch-apisubscriptions--update-subscription) | PATCH | Update subscription auto-renewal settings / flex discount codes | STABLE |
| [`/api/resellers`](#get-apiresellers--get-resellers) | GET | Fetch reseller list or single reseller details | STABLE |
| [`/api/resellers`](#post-apiresellers--create-reseller) | POST | Create a reseller | STABLE |
| [`/api/pricelist`](#post-apipricelist--get-price-list) | POST | Fetch paginated VIP MP pricelist | STABLE |
| [`/api/offer-switch-paths`](#get-apioffer-switch-paths--get-switch-paths) | GET | Fetch valid upgrade/switch paths for an offer | STABLE |
| [`/api/partnerDetails`](#get-apipartnerdetails--get-partner-details) | GET | Fetch partner config (name, market segments, currencies) from env | STABLE |
| [`/api/search`](#get-apisearch--search-resellers-or-customers) | GET | Search resellers or customers by name or ID | STABLE |
| [`/api/recommendation`](#get-apirecommendation--get-recommendations) | GET | Fetch product recommendations for a customer | STABLE |
| [`/api/env`](#get-apienv--get-public-env-vars) | GET | Serve public runtime environment variables (cached 60s) | STABLE |
| `/ping` | GET | Health check — returns `OK` | STABLE |

### APIs We Consume

| Service | Connector Interface | Connector Impl | Auth Type | Purpose |
|---------|--------------------|--------------------|-----------|---------|
| [Adobe IMS](#1-imstoken--oauth-token-endpoint) | `getAccessToken()` | `utils/imsTokenService.ts` | OAuth 2.0 client_credentials | Obtain bearer token for all Partner API calls |
| [VIP MP Partner API — Customers](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/customerController.ts` | Bearer token | Customer CRUD, search |
| [VIP MP Partner API — Orders](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/orderController.ts` | Bearer token | New, return, preview, renewal, switch orders |
| [VIP MP Partner API — Subscriptions](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/subscriptionController.ts` | Bearer token | Subscription CRUD, auto-renewal management |
| [VIP MP Partner API — Resellers](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/resellerController.ts` | Bearer token | Reseller CRUD, search |
| [VIP MP Partner API — Pricelist](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/priceController.ts` | Bearer token | VIP MP pricelist fetch |
| [VIP MP Partner API — Switch Paths](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/switchOrderController.ts` | Bearer token | Offer switch / upgrade paths |
| [VIP MP Partner API — Recommendations](#2-partnerapi--vip-marketplace-partner-api) | controller functions | `controllers/recommendationController.ts` | Bearer token | Product recommendations |

---

## Companion Files

| File | When to Load | Purpose |
|------|-------------|---------|
| `MODULE_INDEX.md` | Impact analysis, feature planning, code generation (load FIRST) | Capability → implementation mapping with Key Decisions per capability |
| `CONTRACTS.md` | Feature planning, code generation | Full API surface: all exposed endpoints and consumed APIs with request/response shapes |
| `CONNECTORS.md` | Feature planning, code generation | Outbound connectors: auth, transport, error posture |
| `BUILD_CONFIG.md` | Code generation | Runtime stack, dependencies, environment variables |
| `CODE_PATTERNS.md` | Code generation | Naming, error handling, validation, logging, test conventions |
| `PLATFORM.md` | Impact analysis, deployment planning | Deployment topology, server config, Docker setup |
| `DB_SCHEMA.md` | n/a | Service owns no data store — see file for details |
