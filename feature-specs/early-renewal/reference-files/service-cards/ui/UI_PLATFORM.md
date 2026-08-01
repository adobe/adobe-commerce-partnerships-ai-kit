# UI Platform — Bridge

## 1. Runtime Specifications

| Field | Value |
|-------|-------|
| Framework | Next.js 15.5.15 |
| Router | Pages router (`pages/` directory) |
| Rendering strategy | CSR (Client-Side Rendering) — no `getServerSideProps`, no `output: 'export'`; API routes are SSR via BFF pattern |
| Language | TypeScript 5.4.2 |
| Runtime | Node.js (custom server via `server.js`) |
| Port | 9000 |
| React | 18.2.0 |
| Component library | `@react-spectrum/s2` ^0.9.1 (Adobe Spectrum 2) |
| Data fetching | `@tanstack/react-query` ^5.77.2 |
| State management | React Context (CartContext + PartnerContext) — no Redux/Zustand |
| Forms | Plain React `useState` + manual validation (react-hook-form present in deps but not used in UI components) |
| Logging (client) | `createClientLogger` from `utils/logger.ts` (Pino-based, console-targeted in browser) |
| Test framework | Jest 30.1.3 + @testing-library/react 16.3.2 |

---

## 2. Build Scripts

| Script | Command | Use |
|--------|---------|-----|
| `dev:local` | `NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js` | Local dev on port 9000 |
| `dev` | `next dev` | Standard dev server (port 3000) |
| `build` | `next build` | Production build |
| `start` | `next start` | Serve production build |
| `lint` | `next lint` | ESLint |
| `test` | `jest` | All tests |
| `test:unit` | `jest tests/unit` | Unit tests only |
| `test:coverage` | `jest --coverage` | Coverage report |
| `format` | `prettier --write .` | Format all files |

---

## 3. Environment Variables

Browser-accessible prefix: `NEXT_PUBLIC_` (baked in at `next build`).

| Variable | Required | Client or Server | Description |
|----------|----------|-----------------|-------------|
| `PARTNER_API_BASE_URL` | Yes | Server only | VIP MP Partner API base URL |
| `PARTNER_CLIENT_ID` | Yes | Server only | OAuth client ID / `x-api-key` |
| `PARTNER_CLIENT_SECRET` | Yes | Server only | OAuth client secret |
| `IMS_TOKEN_URL` | No | Server only | IMS token endpoint |
| `IMS_SCOPES` | No | Server only | OAuth scopes |
| `PARTNER_NAME` | No | Server + client via `/api/partnerDetails` | Partner display name |
| `MARKET_SEGMENTS` | Yes | Server + client via `/api/partnerDetails` | JSON array of segment codes |
| `CURRENCIES` | Yes | Server + client via `/api/partnerDetails` | JSON array of currency codes |
| `REGION` | Yes | Server + client via `/api/partnerDetails` | Price region code |
| `NODE_ENV` | No | Both | Controls Pino log format |
| `NEXT_PUBLIC_APP_ENV` | No | Client | Environment label (`local`, `stage`, `prod`) |
| `LOG_LEVEL` | No | Server | Pino log level override |

---

## 4. Key Dependencies

| Package | Version | Role |
|---------|---------|------|
| `next` | ^15.5.15 | Framework — routing, SSR API routes, static optimization |
| `react` | 18.2.0 | UI library |
| `@tanstack/react-query` | ^5.77.2 | Data fetching, caching, pagination |
| `@react-spectrum/s2` | ^0.9.1 | Adobe Spectrum 2 UI components |
| `react-hook-form` | ^7.51.2 | In deps (not used in component code) |
| `zod` | ^3.22.4 | Runtime validation (models layer) |
| `pino` | ^9.9.5 | Structured logging |
| `uuid` | ^11.1.0 | Correlation IDs |
| `js-cookie` | ^3.0.5 | Browser cookie access |

---

## 5. Next.js Config Highlights (`next.config.js`)

- Spectrum S2 package transpiled via `transpilePackages: ['@react-spectrum/s2', '@react-spectrum/...']`
- Custom security headers configured (X-Frame-Options, X-Content-Type-Options, Referrer-Policy)
- No custom rewrites or redirects

---

## 6. Accessibility (a11y)

| Aspect | Detail |
|--------|--------|
| Component-level a11y | Handled automatically by `@react-spectrum/s2` (ARIA roles, keyboard nav, focus ring) |
| Manual additions required | `aria-label` on icon-only buttons; `role="button"` + `tabIndex=0` + `onKeyDown` on `<div>` cards |
| Focus management | Spectrum manages focus within dialogs automatically |
| Screen reader support | Spectrum components tested against NVDA/JAWS by Adobe |

---

## 7. Browser Storage

| Key | Type | Contents | Cleared by |
|-----|------|----------|-----------|
| `bridge_cart` | `localStorage` | Full cart state (items, customer info, modal state) | `CartContext.clearCart()` |

No `sessionStorage` or cookie-based client-side storage.

---

## 8. Monitoring & Logging (Client)

| Aspect | Detail |
|--------|--------|
| Client logger | `createClientLogger(component)` from `utils/logger.ts` — Pino targeted to console |
| Log level | `debug` in dev, `info` in prod (via `NEXT_PUBLIC_APP_ENV`) |
| No error tracking / APM | No Sentry, Datadog RUM, or similar detected |
| No analytics | No Amplitude, Mixpanel, or GA detected |

---

## 9. Feature Flags

None detected — no feature flag framework in use.
