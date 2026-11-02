# BUILD_CONFIG.md — adobe-commerce-partnerships-ref-app

## 1. Runtime Specifications

| Field | Value | Status |
|---|---|---|
| Build tool | npm (`package.json`, `package-lock.json`) | CONFIRMED |
| Language | TypeScript 5.4.x (`tsconfig.json` target `ES6`, module `esnext`) | CONFIRMED |
| Runtime version | Node 20 (`Dockerfile:1` — `FROM node:20-alpine`) | CONFIRMED (no `.nvmrc`/`.node-version`) |
| Web framework | Next.js 15.5.x — Pages Router, API Routes under `pages/api/` | CONFIRMED |
| ORM / persistence | None — this service owns no data store; all state lives in the upstream Adobe Commerce Partner API | CONFIRMED |
| Test framework | Jest 30.x (via `next/jest`), `ts-jest`, `@testing-library/react` + `@testing-library/jest-dom` | CONFIRMED |

## 2. Package Identity

| Field | Value |
|---|---|
| Package name (`package.json` `name` field, historical — this service is referred to as `adobe-commerce-partnerships-ref-app` throughout these cards) | `Bridge` (`package.json:2`) |
| Version | `1.0.0` |

## 3. Key Dependencies

| Category | Library | Version |
|---|---|---|
| Framework | `next` | ^15.5.15 |
| UI | `react`, `react-dom` | 18.2.0 |
| UI component library | `@react-spectrum/s2` | ^0.9.1 |
| Data fetching / cache (client) | `@tanstack/react-query` (+ `-devtools`) | ^5.77.2 |
| Schema validation | `zod` | ^3.22.4 |
| Forms | `react-hook-form` | ^7.51.2 |
| Logging | `pino`, `pino-pretty` | ^9.9.5 / ^13.1.1 |
| HTTP (unused server-side; native `fetch` used instead) | `axios`, `node-fetch` | ^1.6.8 / ^3.3.2 |
| Auth token (declared, no usage found in source) | `jsonwebtoken` | ^9.0.2 |
| ID generation | `uuid` | ^11.1.0 |
| Cookies | `cookie`, `js-cookie` | ^0.6.0 / ^3.0.5 |
| Env loading | `dotenv` | ^16.5.0 |
| Test | `jest`, `ts-jest`, `jest-environment-jsdom` | ^30.1.3 / ^29.4.1 / ^30.1.2 |
| Lint/format | `eslint` (+ `eslint-config-next`), `prettier` | ^8.57.1 / ^3.6.2 |

## 4. Environment Variables

Sourced from `.env.sample` and grep of `process.env.*` usage.

| Variable | Required | Used by | Purpose |
|---|---|---|---|
| `NODE_ENV` | Yes | `next`, `server.js`, `utils/logger.ts` | dev/production/test mode switch |
| `PARTNER_API_BASE_URL` | Yes | all controllers (see `CONNECTORS.md#1`) | Adobe Commerce Partner API base URL |
| `IMS_TOKEN_URL` | Yes | `utils/imsTokenService.ts` | Adobe IMS token endpoint base |
| `PARTNER_CLIENT_ID` | Yes | `imsTokenService.ts`, all controllers (`x-api-key` header) | IMS client ID / API key |
| `PARTNER_CLIENT_SECRET` | Yes (secret) | `imsTokenService.ts` | IMS client secret |
| `IMS_SCOPES` | No (defaults to `openid,AdobeID,read_organizations`) | `imsTokenService.ts` | OAuth scopes requested |
| `PARTNER_NAME` | Yes | `controllers/partnerDetailsController.ts` | static partner display name |
| `MARKET_SEGMENTS` | Yes | `partnerDetailsController.ts` | JSON array string, e.g. `["COM","EDU","GOV"]` |
| `CURRENCIES` | Yes | `partnerDetailsController.ts` | JSON array string, e.g. `["USD"]` |
| `REGION` | Yes | `partnerDetailsController.ts` | price region code, e.g. `NA` |
| `LOG_LEVEL` | No (defaults to `debug` dev / `info` prod) | `utils/logger.ts` | pino log level override |
| `NEXT_PUBLIC_APP_ENV` | No | `utils/logger.ts` (base log field), `server.js` | environment label surfaced in logs |
| `HOSTNAME` | No (defaults to `localhost`) | `utils/logger.ts` | server log field |

## 5. Secrets & Sensitive Data

| Secret | Source | Notes |
|---|---|---|
| `PARTNER_CLIENT_SECRET` | `.env` (local), container env at runtime (`docker-compose.yml` `env_file: .env`) | never logged directly; not referenced by `utils/logger.ts` redaction (no redaction config present — treat all log call sites passing raw request/response bodies as a leakage risk if `PARTNER_CLIENT_SECRET` were ever included in a payload, which it is not today) |
| `PARTNER_CLIENT_ID` | `.env` | doubles as both IMS client ID and `x-api-key` header value — not fully secret (sent as a request header) but treated as a credential |

No secrets manager / vault integration found — configuration is environment-variable only, loaded via `dotenv` locally and container env injection in deployment.

## 6. Build & Run Scripts (`package.json`)

| Script | Command | Purpose |
|---|---|---|
| `dev` | `next dev` | local dev server |
| `dev:local` | `NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js` | local dev via custom HTTP server (`server.js`) |
| `build` | `next build` | production build |
| `start` | `next start` | production server (Next.js default) |
| `lint` | `next lint` | ESLint |
| `test` / `test:unit` | `jest` / `jest tests/unit` | run tests |
| `format` | `prettier --write .` | formatting |

## 7. Container

- `Dockerfile`: `node:20-alpine` base → `npm install` → `npm run build` → `CMD ["node", "server.js"]`, exposes ports `9000` and `8080`.
- `server.js`: custom HTTP server wrapping Next.js's request handler; listens on port `9000` only (despite `EXPOSE 8080` in the Dockerfile, no listener is bound to 8080 in `server.js`); adds a `/ping` health-check route returning plain-text `OK`.