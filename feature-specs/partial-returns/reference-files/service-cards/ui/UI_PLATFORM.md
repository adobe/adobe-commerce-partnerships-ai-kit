# UI_PLATFORM.md — adobe-commerce-partnerships-ref-app (UI)

## §1 Runtime

| Field | Value |
|---|---|
| Framework | Next.js 15.5.x — Pages Router |
| Router | Next.js file-based routing (`pages/*.tsx`) — no `react-router-dom` |
| Rendering strategy | Pure CSR — no `getServerSideProps`/`getStaticProps`/`getInitialProps` found in any page; every data fetch happens client-side after mount, gated behind `useQuery`/`useEffect` |
| Build tool | npm + Next.js's own webpack build (`next build`); `next.config.js` adds one custom webpack plugin (`unplugin-parcel-macros`) for `@react-spectrum/s2` support and transpiles `@adobe/react-spectrum`/`@react-spectrum/*`/`@spectrum-icons/*` packages |
| Language | TypeScript 5.4.x (`strict: true`, target `ES6`) |
| Runtime | Node 20 (same Next.js app/runtime as the backend — see backend `BUILD_CONFIG.md`) |
| State library | React Context (`CartContext`, `PartnerContext`) — no Zustand/Redux/Jotai/Recoil |
| Data fetch library | TanStack Query v5 (`@tanstack/react-query` + `-devtools`, mounted via `<ReactQueryDevtools initialIsOpen={false} />` in `pages/_app.tsx`) |
| Component library | Adobe `@react-spectrum/s2` (`Provider colorScheme="light"`) |
| Form library | `react-hook-form` is declared but **unused** — see `UI_CODE_PATTERNS.md → B.6` |

## §2 Build

**Scripts** (`package.json`):
```json
"dev": "next dev",
"dev:local": "NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js",
"build": "next build",
"start": "next start",
"lint": "next lint",
"test": "jest",
"format": "prettier --write ."
```

**Key frontend dependencies:**

| Category | Library | Version |
|---|---|---|
| Framework | `next` | ^15.5.15 |
| UI runtime | `react`, `react-dom` | 18.2.0 |
| Component library | `@react-spectrum/s2` | ^0.9.1 |
| Data fetching | `@tanstack/react-query`, `@tanstack/react-query-devtools` | ^5.77.2 / ^5.90.2 |
| Forms (declared, unused) | `react-hook-form` | ^7.51.2 |
| Client-side ID generation | `uuid` | ^11.1.0 |
| Cookies (client) | `js-cookie` | ^3.0.5 |
| Logging | `pino` (browser build) | ^9.9.5 |
| Font | `next/font/google` (`Source_Sans_3`, built into Next.js — no separate package) | — |

**Build plugins:** `unplugin-parcel-macros` (webpack plugin registered in `next.config.js`, required by `@react-spectrum/s2`'s macro-based styling); `transpilePackages` globs all `@adobe/react-spectrum`, `@react-spectrum/*`, `@spectrum-icons/*` packages found under `node_modules/`.

**Bundle analysis:** none configured — no `@next/bundle-analyzer` or equivalent found.

**Security headers:** `next.config.js → headers()` sets `X-Frame-Options: DENY` and `X-Content-Type-Options: nosniff` on every route.

## §3 Environment Variables

| Name | Browser-accessible | Purpose | Required | Example |
|---|---|---|---|---|
| `NEXT_PUBLIC_APP_ENV` | Yes | Environment label (`dev`/`stage`/`prod`) used by `types/environment.ts` helpers (`isDevelopment`/`isStaging`/`isProduction`) and embedded in every log line (`utils/logger.ts`) | No (defaults to `dev` via `AppEnvironment.DEV` fallback) | `local` (per `dev:local` script) |
| `NODE_ENV` | No (server-only, but affects client bundle via Next.js's own dev/prod split) | Standard Next.js dev/production/test switch | Yes | `development` |
| `LOG_LEVEL` | No | Pino log level override, read server-side but also referenced in the isomorphic `utils/logger.ts` | No | `debug` |

All other environment variables (`PARTNER_API_BASE_URL`, `IMS_TOKEN_URL`, `PARTNER_CLIENT_ID`, `PARTNER_CLIENT_SECRET`, `PARTNER_NAME`, `MARKET_SEGMENTS`, `CURRENCIES`, `REGION`) are backend-only — see the backend `BUILD_CONFIG.md`. None use the `NEXT_PUBLIC_` prefix, so none are bundled into client JavaScript; the UI only ever learns partner configuration indirectly, via the `/api/partnerDetails` response (see `DATA_LAYER.md` unit 1).

`pages/api/env.ts` exists as a scaffold for serving additional runtime-injected public env vars but its allowlist object is currently empty (see backend `SERVICE_CARD.md` fragile areas) — no UI code currently calls `/api/env`.

## §4 Local Development

From `README.md`:
```bash
npm install
cp .env.sample .env   # then fill in PARTNER_API_BASE_URL, IMS_TOKEN_URL, etc.
npm run dev:local     # NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js — port 9000
npm run dev           # standard `next dev` — port 3000
```
No HTTPS setup is configured for local development (`docker-compose.yml` mounts a `./certs` volume into the container but nothing in `server.js` or `next.config.js` reads certs or starts a TLS listener — `http.createServer` only). No auth exists in dev (or anywhere) — see `ROUTES.md → Auth Guard Pattern`.

## §5 Routing Table (Quick Reference)

`ROUTES.md` is authoritative. Summary:

| URL | File | Auth required |
|---|---|---|
| `/` | `pages/index.tsx` | No |
| `/resellers` | `pages/resellers.tsx` | No |
| `/customers` | `pages/customers.tsx` | No |
| `/customerdetails` | `pages/customerdetails.tsx` | No |
| `/catalog` | `pages/catalog.tsx` | No |
| `/checkout` | `pages/checkout.tsx` | No |
| `/orderConfirmation` | `pages/orderConfirmation.tsx` | No |

## §6 Feature Flags

None found — no feature-flag SDK (LaunchDarkly, Unleash, Split, GrowthBook, etc.) in `package.json`, and no custom flag/toggle mechanism in `utils/`, `hooks/`, or `contexts/`. The closest analog is server-authoritative partner configuration (`PartnerContext` — market segments, currencies, region), which gates *what data is fetched*, not *which UI code path renders* — see `STATE_MANAGEMENT.md → 2. PartnerContext`.

## §7 Browser Storage (Quick Reference)

`STATE_MANAGEMENT.md → Storage Key Registry` is authoritative.

| Key | Type | Owner | Contents | TTL |
|---|---|---|---|---|
| `bridge_cart` | `localStorage` | `CartContext` | cart item maps + `customerInfoInCart` | indefinite (cleared only by `clearCart()`) |

## §8 Accessibility Compliance

| Field | Value |
|---|---|
| WCAG target | NOT CAPTURED — no explicit WCAG level target found in README, CI config, or code comments |
| Automated tooling | None dedicated (no `axe-core`, `jest-axe`, `eslint-plugin-jsx-a11y` as a direct dependency). `eslint-config-next`'s `next/core-web-vitals` preset does transitively include `jsx-a11y` recommended rules, so basic a11y linting runs as part of `npm run lint`, but this is inherited, not explicitly configured. |
| Known exceptions | Several custom interactive elements are hand-rolled without a component library's built-in a11y (see `UI_CODE_PATTERNS.md → B.5`): the reseller/customer card grid items (`role="button"` + manual `onKeyDown` for Enter/Space) and `NavigationPanel`'s breadcrumbs (plain styled `<span>`s, not a semantic breadcrumb/nav element) |
| CI enforcement | No CI config found (no `.github/workflows/`, `Jenkinsfile`, `.gitlab-ci.yml`) — lint/test are npm scripts only, not gated by any pipeline visible in this repo |

## §9 Monitoring & Error Tracking

None found — no error-tracking SDK (Sentry, Bugsnag, Rollbar) or RUM/APM tool (Datadog RUM, New
Relic Browser) in `package.json`. The only observability mechanism is `utils/logger.ts`'s Pino
instance running in browser mode (`browser: { asObject: true }`), which routes through
`console.*` — there is no log shipping from the browser to any backend or third-party service.

## §11 Links & Runbooks

| Resource | Location |
|---|---|
| Setup / configuration / testing / Docker instructions | `README.md` (repo root) |
| Architecture notes (MVC layering) | `ARCHITECTURE.md` (repo root) — backend-focused, does not describe the UI layer specifically |
| Runbooks | NOT CAPTURED — none found in this repo |