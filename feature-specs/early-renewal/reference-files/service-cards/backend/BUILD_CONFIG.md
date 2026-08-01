# Build Config — Bridge

## 1. Runtime Specifications

| Field | Value |
|-------|-------|
| Build tool | npm (`package.json`) |
| Language | TypeScript 5.4.2 |
| Runtime | Node.js (version not pinned in package.json; no `.nvmrc` detected) |
| Framework | Next.js 15.5.15 |
| Server | Custom Node HTTP server (`server.js`) — wraps Next.js request handler |
| Port | 9000 |
| Test framework | Jest 30.1.3 + `@testing-library/react` 16.3.2 |

---

## 2. Key Dependencies

### Runtime

| Package | Version | Role |
|---------|---------|------|
| `next` | ^15.5.15 | Full-stack framework — routing, SSR, API routes |
| `react` | 18.2.0 | UI library |
| `react-dom` | 18.2.0 | DOM rendering |
| `zod` | ^3.22.4 | Runtime schema validation for all models |
| `axios` | ^1.6.8 | HTTP client (present in deps; controllers use native `fetch`) |
| `node-fetch` | ^3.3.2 | Fetch polyfill for Node (present in deps; controllers use native `fetch`) |
| `pino` | ^9.9.5 | Structured JSON logging |
| `pino-pretty` | ^13.1.1 | Human-readable log formatting for development |
| `uuid` | ^11.1.0 | UUIDs for `X-Correlation-Id` / `X-Request-Id` headers |
| `jsonwebtoken` | ^9.0.2 | JWT handling |
| `react-hook-form` | ^7.51.2 | Client-side form management |
| `@tanstack/react-query` | ^5.77.2 | Client-side data fetching and caching |
| `@react-spectrum/s2` | ^0.9.1 | Adobe Spectrum 2 UI component library |
| `dotenv` | ^16.5.0 | `.env` file loading |
| `js-cookie` | ^3.0.5 | Cookie access in the browser |
| `cookie` | ^0.6.0 | Server-side cookie parsing |

### Dev / Test

| Package | Version | Role |
|---------|---------|------|
| `typescript` | ^5.4.2 | TypeScript compiler |
| `jest` | ^30.1.3 | Test runner |
| `jest-environment-jsdom` | ^30.1.2 | DOM environment for component tests |
| `ts-jest` | ^29.4.1 | TypeScript support for Jest |
| `@testing-library/react` | ^16.3.2 | React component testing utilities |
| `@testing-library/jest-dom` | ^6.8.0 | Custom Jest matchers for DOM assertions |
| `eslint` | ^8.57.1 | Linting |
| `prettier` | ^3.6.2 | Code formatting |

---

## 3. Scripts

| Script | Command | Use |
|--------|---------|-----|
| `dev` | `next dev` | Standard Next.js dev server (port 3000) |
| `dev:local` | `NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js` | Local dev via custom server on port 9000 |
| `build` | `next build` | Production build |
| `start` | `next start` | Start production build |
| `lint` | `next lint` | ESLint |
| `test` | `jest` | All tests |
| `test:unit` | `jest tests/unit` | Unit tests only |
| `test:coverage` | `jest --coverage` | Coverage report |
| `format` | `prettier --write .` | Format all files |

---

## 4. Environment Variables

All env vars are read at runtime. No build-time injection except `NEXT_PUBLIC_*` vars (baked in at `next build`).

| Variable | Required | Description |
|----------|----------|-------------|
| `PARTNER_API_BASE_URL` | Yes | Base URL for the VIP Marketplace Partner API (e.g. `https://api.partners.adobe.com`) |
| `PARTNER_CLIENT_ID` | Yes | OAuth client ID — used as both `client_id` in token request and `x-api-key` header on all Partner API calls |
| `PARTNER_CLIENT_SECRET` | Yes | OAuth client secret for IMS token fetch |
| `IMS_TOKEN_URL` | No | IMS token endpoint — defaults to `https://ims-na1-stg1.adobelogin.com/ims/token/v2` |
| `IMS_SCOPES` | No | OAuth scopes — defaults to `openid,AdobeID,read_organizations` |
| `PARTNER_NAME` | No | Display name for the partner |
| `MARKET_SEGMENTS` | Yes | JSON array of market segment codes — e.g. `["COM","EDU","GOV"]` |
| `CURRENCIES` | Yes | JSON array of currency codes — e.g. `["USD"]` |
| `REGION` | Yes | Price region code — e.g. `NA` |
| `NODE_ENV` | No | `development` / `production` / `test` — controls Pino log format |
| `NEXT_PUBLIC_APP_ENV` | No | App environment label (e.g. `local`, `stage`, `prod`) — included in log base fields |
| `LOG_LEVEL` | No | Pino log level override — defaults to `debug` (dev) or `info` (prod) |

---

## 5. Secrets & Sensitive Data

| Secret | Env var | Notes |
|--------|---------|-------|
| OAuth client secret | `PARTNER_CLIENT_SECRET` | Never logged; passed directly to IMS token endpoint |
| OAuth client ID | `PARTNER_CLIENT_ID` | Used as `x-api-key` in API calls; treat as sensitive |
| IMS access token | In-memory only | Cached in `imsTokenService.ts` module scope; never written to disk or logs |

---

## 6. TypeScript Config

`tsconfig.json` — standard Next.js TypeScript config. Strict mode not explicitly enabled. `ts-jest` used for test compilation.
