# CODE_PATTERNS.md — adobe-commerce-partnerships-ref-app

## 1. Package Layout (MVC, adapted for Next.js — see `ARCHITECTURE.md`)

| Layer | Path | Responsibility |
|---|---|---|
| API Routes | `pages/api/*.ts` | Thin entry points: parse `req`, call `handlePrerequisites` for the IMS token, delegate to a controller, map the result/error to an HTTP response. No business logic. |
| Controllers | `controllers/*.ts` | All business logic and orchestration: input validation, upstream `fetch` calls, response validation, error wrapping. One file per resource (`customerController.ts`, `orderController.ts`, etc.). |
| Models | `models/*.ts` | Zod schemas + inferred TypeScript types. Both request and response shapes for a resource live in the same file. |
| Utilities | `utils/*.ts` | Cross-resource helpers: `apiError.ts` (error class), `logger.ts` (Pino setup), `commonUtils.ts` (misc + `handlePrerequisites`), `imsTokenService.ts` (token acquisition), `constants.ts` (enums/constants). |
| Frontend (out of scope for this card set) | `pages/*.tsx`, `components/`, `hooks/`, `contexts/` | UI — see the project's UI service cards, if generated. |

Naming: one controller file per resource, named `<resource>Controller.ts`; one model file per
resource, named to match the domain noun (`Customer.ts`, `CustomerDetails.ts`, `Order.ts`, …);
functions are verb-first (`getCustomer`, `createOrder`, `findResellerByName`).

## 2. Domain / DTO Pattern

- Every schema is a Zod object; the TypeScript type is always derived via `z.infer<typeof XSchema>` — never hand-written separately from the schema.
- **Minimal vs. full schemas:** list-view endpoints return a `*MinimalSchema` (e.g. `CustomerMinimalSchema`, `ResellerMinimalSchema` — just `id` + `companyProfile.companyName`) while detail-view endpoints return the full `*DetailsSchema`/`*Schema`. Keep this split when adding new list endpoints.
- **`.passthrough()`** is used on schemas that mirror an upstream response adobe-commerce-partnerships-ref-app doesn't fully control (`OrderSchema`, `LineItemSchema`, `PriceListResponseSchema`, `PartnerDetailsSchema`, `SubscriptionSchema`) — unknown upstream fields survive parsing instead of being stripped. Schemas for data adobe-commerce-partnerships-ref-app itself constructs (e.g. request bodies) do not use `.passthrough()`.
- Pagination responses follow a consistent shape: `{ <items>: T[], totalCount, count, offset, limit, hasMore }`, with `hasMore` computed client-side as `offset + limit < totalCount`.
- `BackendResult<T>` (`models/Result.ts`) — `{ data: T; requestId?: string }` — is the standard controller return wrapper so the upstream's `X-Request-Id` can be threaded back through to the API response. Not every controller function uses it consistently (some return the bare payload); follow `BackendResult<T>` for new functions.

## 3. Error Handling

- Single custom error type: `ApiError extends Error` (`utils/apiError.ts`) — carries `status` (HTTP status to return) and optional `requestId` (upstream correlation ID to forward).
- Controllers never let a raw `Error` propagate for an expected failure — they catch and re-throw as `ApiError` with an explicit status: `400` for input-validation failures, `500` for missing env config, and the **upstream's own status code** for upstream failures (not remapped).
- Every `pages/api/*.ts` handler follows the same catch block shape:
  ```ts
  catch (error: any) {
    if (error instanceof ApiError) {
      if (error.requestId) forwardRequestIdHeader(res, error.requestId);
      res.status(error.status).json({ error: error.message });
    } else {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  }
  ```
  `pages/api/orders.ts` and `pages/api/resellers.ts`/`subscriptions.ts` extend this with extra
  branches (JSON-parsing the backend error body; catching `ZodError` explicitly at 400).
- `processSettledResults()` (`utils/apiError.ts:17`) is the shared helper for summarizing `Promise.allSettled` results (successful/failed counts + error messages) when a capability fans out to multiple upstream calls.

## 4. Validation

- **Library:** Zod, exclusively — no other validation library (Joi, Yup, class-validator) present.
- **Where invoked:** in the controller layer, immediately after logging "Request Received" and before any upstream call — never in the API-route layer. Pattern: `try { XSchema.parse(body); } catch (err) { logger.error(...); throw new ApiError('... validation failed', 400); }`.
- **Response validation:** upstream responses are also parsed through the corresponding Zod schema before being returned to the caller; a validation failure here throws `ApiError('<Resource> data validation failed', 500, requestId)` — treated as a bug in this service's own contract handling (contract drift with the upstream), not a client error.
- **Guards beyond schema:** some fields need validation Zod's built-ins can't express cleanly — e.g. `priceController.ts:39` explicitly checks `currency.trim() === ''` after Zod parsing, because a whitespace-only string passes `.min(1)` but is still not a usable currency code.
- List-parsing that tolerates partial failure: `getAllCustomers`/`findCustomerByName` map each upstream item through its Minimal schema inside a `try/catch`, logging and dropping (`filter(Boolean)`) any item that fails validation rather than failing the whole request.

## 5. Logging

- Pino (`utils/logger.ts`), wrapped by four factory functions: `createAPILogger(req)` (route layer, includes method/path/correlationId), `createControllerLogger(name, operation)` (controller layer), `createClientLogger(module)` (browser-side), `createChildLogger(context)` (generic).
- Three standardized log helpers, always called with a consistent field set: `logRequest(logger, body, meta?)`, `logResponse(logger, data, requestId?, meta?)`, `logErrorResponse(logger, errorData, requestId?, meta?)` — all stringify the body/response/error payload into a single field (`requestBody`/`responseBody`/`errorBody`) rather than nesting it, matching `LOG_MESSAGES` constants.
- Log level: `debug` in development, `info` in production, `error`-only in test; pretty-printed via `pino-pretty` in development, structured JSON in production.

## 6. Test Patterns

- **Framework:** Jest 30 via `next/jest`, executed under `jest-environment-jsdom`, transformed with `ts-jest`.
- **File placement:** integration-style controller tests live at the repo-root `tests/` directory (not co-located with source), named `<action><resource>.integration.test.ts` (e.g. `createorder.integration.test.ts`, `getCustomerSubscriptions.integration.test.ts`); narrower unit tests live under `tests/unit/<mirrors-source-path>` (e.g. `tests/unit/utils/apiError.test.ts`).
- **Mocking upstream calls:** `global.fetch` is mocked globally in `jest.setup.js` (`global.fetch = jest.fn()`) and stubbed per-test with `(global.fetch as jest.Mock).mockResolvedValueOnce({ ok, status, text: async () => ..., headers: { get: () => ... } })` — no HTTP-mocking library (nock, msw) is used.
- **Mocking the logger:** every controller test calls `jest.mock('../utils/logger', () => ({ ... }))` with a no-op logger object matching the real module's exports (`createControllerLogger`, `logRequest`, `logResponse`, `logErrorResponse`, `logger`, `default`) — copy this shape exactly or the mock will silently miss calls.
- **Env isolation:** tests snapshot `process.env` in `beforeEach` (`savedEnv = { ...process.env }`), set the specific vars the controller under test needs (`PARTNER_API_BASE_URL`, `PARTNER_CLIENT_ID`), and restore in `afterEach` (`process.env = savedEnv`).
- **Assertion style:** tests assert on the thrown `ApiError`'s `status`/`message` (`err = await fn(...).catch(e => e); expect(err).toBeInstanceOf(ApiError)`), on `global.fetch` call count/args (URL contains ID, method, headers), and on returned `data` via `toMatchObject`.
- **Coverage scope:** `jest.config.js` collects coverage from `models/`, `controllers/`, `utils/`, `components/` — the backend surface plus shared frontend components.