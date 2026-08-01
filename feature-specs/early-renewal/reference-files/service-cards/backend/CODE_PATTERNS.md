# Code Patterns — Bridge

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Controller files | `camelCase` + `Controller.ts` | `orderController.ts` |
| Controller functions | `camelCase` verb + noun | `createNewOrder`, `getCustomerSubscriptions` |
| API route files | `camelCase.ts` under `pages/api/` | `customers.ts`, `offer-switch-paths.ts` |
| Model files | `PascalCase.ts` | `Order.ts`, `CustomerDetails.ts` |
| Zod schema exports | `PascalCase` + `Schema` | `OrderSchema`, `NewOrderSchema` |
| Type exports | `PascalCase` | `Order`, `LineItem`, `BackendResult` |
| Utility files | `camelCase.ts` | `commonUtils.ts`, `imsTokenService.ts` |
| Constants | `SCREAMING_SNAKE_CASE` | `HTTP_METHOD.GET`, `ORDER_API_TYPE.NEW` |
| Constant objects | `SCREAMING_SNAKE_CASE` const object with `as const` | `export const HTTP_METHOD = { GET: 'GET' } as const` |

---

## Package Layout

```
bridge/
├── pages/api/          ← Next.js API route handlers (thin — delegate to controllers)
├── controllers/        ← Business logic: validation, API calls, response mapping
├── models/             ← Zod schemas + TypeScript type exports
├── types/              ← TypeScript-only types (no Zod) for client-side shapes
├── utils/              ← Shared utilities: auth, logging, error, constants, helpers
├── hooks/              ← Client-side React hooks (data fetching, state)
├── components/         ← React UI components
├── contexts/           ← React context providers (cart, partner)
├── pages/              ← Next.js UI pages (non-api)
└── tests/              ← Integration and unit tests
```

---

## API Route Handler Pattern

Every handler in `pages/api/` follows this structure:

```typescript
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  try {
    const { accessToken } = await handlePrerequisites(req);  // 1. Auth

    // 2. Route on method + type query param
    if (req.method === HTTP_METHOD.GET) {
      const { type, ...params } = req.query;
      if (type === SOME_API_TYPE.OPERATION) {
        result = await someControllerFn(params, accessToken);
        statusCode = 200;
        responseData = result.data;
      } else {
        throw new ApiError('Invalid type parameter', 400);
      }
    } else {
      return res.status(405).end(`Method ${req.method} Not Allowed`);
    }

    forwardRequestIdHeader(res, result?.requestId);  // 3. Forward X-Request-Id
    return res.status(statusCode).json(responseData);

  } catch (error: any) {
    if (error instanceof ApiError) {
      if (error.requestId) forwardRequestIdHeader(res, error.requestId);
      res.status(error.status).json({ error: error.message });
    } else {
      res.status(500).json({ error: 'Internal Server Error' });
    }
  }
}
```

Key rules:
- All operations are dispatched via the `type` query parameter
- `handlePrerequisites(req)` is always the first call
- Method-not-allowed returns 405 via `res.status(405).end(...)`
- `forwardRequestIdHeader` is called before every successful response

---

## Controller Pattern

```typescript
export async function createSomething(
  customerId: string,
  body: SomeType,
  accessToken: string
): Promise<BackendResult<SomeResponse>> {
  const logger = createControllerLogger('someController', 'createSomething');
  const apiKey = process.env.PARTNER_CLIENT_ID;

  // 1. Input validation
  try {
    SomeSchema.parse(body);
  } catch (err) {
    logger.error({ err }, 'Input validation failed');
    throw new ApiError('Input validation failed', 400);
  }

  logRequest(logger, body);

  // 2. Guard env vars
  if (!accessToken) throw new ApiError('Missing ACCESS_TOKEN', 500);
  if (!apiKey) throw new ApiError('Missing ADOBE_API_KEY', 500);

  // 3. Outbound fetch
  const result = await fetch(url, {
    method: HTTP_METHOD.POST,
    headers: {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      'X-Correlation-Id': uuidv4(),
      'X-Request-Id': uuidv4(),
    },
    body: JSON.stringify(body),
  });

  const text = await result.text();
  const requestId = extractRequestIdFromResponse(result);

  // 4. Parse response
  let data;
  try {
    data = JSON.parse(text);
  } catch (e) {
    logErrorResponse(logger, { error: 'Parse failed', text }, requestId, { status: result.status });
    throw new ApiError('Non-JSON response from Adobe API', result.status, requestId);
  }

  if (!result.ok) {
    logErrorResponse(logger, data, requestId, { status: result.status });
    throw new ApiError('Failed to create something', result.status, requestId);
  }

  logResponse(logger, data, requestId, {});

  // 5. Validate response
  try {
    return { data: SomeResponseSchema.parse(data), requestId };
  } catch (err: any) {
    logErrorResponse(logger, data, requestId, { validationError: String(err) });
    throw new ApiError('Response validation failed', 500, requestId);
  }
}
```

---

## Error Handling

**Error class:** `ApiError extends Error` (`utils/apiError.ts`)

```typescript
class ApiError extends Error {
  status: number;        // HTTP status code
  requestId?: string;   // X-Request-Id from upstream
  constructor(message: string, status: number = 500, requestId?: string)
}
```

**Throw pattern:**
- Input validation failure → `throw new ApiError('Input validation failed', 400)`
- Missing env var → `throw new ApiError('Missing PARTNER_CLIENT_ID', 500)`
- Non-OK upstream response → `throw new ApiError(message, result.status, requestId)`
- Non-JSON upstream response → `throw new ApiError('Non-JSON response from Adobe API', result.status, requestId)`
- Response schema validation failure → `throw new ApiError('...validation failed', 500, requestId)`

**Catch pattern in route handlers:**
```typescript
if (error instanceof ApiError) {
  if (error.requestId) forwardRequestIdHeader(res, error.requestId);
  res.status(error.status).json({ error: error.message });
} else {
  logger.error({ err: error }, 'Unexpected error');
  res.status(500).json({ error: 'Internal Server Error' });
}
```

---

## Validation

**Library:** Zod 3.22.4

**Schema location:** `models/` directory — one file per domain entity

**Invocation point:** Inside controller functions before the outbound API call:
```typescript
try {
  SomeSchema.parse(body);
} catch (err) {
  logger.error({ err }, 'Input validation failed');
  throw new ApiError('Input validation failed', 400);
}
```

**Response validation:** After successful API response, before returning to caller:
```typescript
try {
  return { data: SomeSchema.parse(data), requestId };
} catch (err) {
  throw new ApiError('Data validation failed', 500, requestId);
}
```

**`.passthrough()` usage:** Most response schemas use `.passthrough()` — unexpected fields from the Partner API are preserved and passed through to the caller.

---

## Constants

**File:** `utils/constants.ts`

Centralised constant objects using `as const`:
```typescript
export const HTTP_METHOD = { GET: 'GET', POST: 'POST', PATCH: 'PATCH', ... } as const;
export const ORDER_API_TYPE = { NEW: 'NEW', PREVIEW: 'Preview', RENEWAL_ORDER: 'RenewalOrder', ... } as const;
export const CUSTOMER_API_TYPE = { GET_CUSTOMER: 'getCustomer', ... } as const;
```

Route handlers import and use these for `req.method` and `type` query parameter comparisons.

---

## Logging

**Library:** Pino 9.9.5

**Logger factory pattern:**
```typescript
// In controllers:
const logger = createControllerLogger('controllerName', 'operationName');
// Produces: { component: 'controller', controller: 'controllerName', operation: 'operationName' }

// In API routes:
const logger = createAPILogger(req);
// Produces: { component: 'api', method, path, correlationId, userAgent }
```

**Structured log helpers:**
```typescript
logRequest(logger, requestBody, metaData?)    // level: info, msg: 'API Request from Bridge'
logResponse(logger, responseBody, requestId?, metaData?)  // level: info, msg: 'API Response to Bridge'
logErrorResponse(logger, errorBody, requestId?, metaData?)  // level: error, msg: 'API Error Response from Bridge'
```

**Environment behaviour:** Development → pino-pretty (colourised), Production → JSON, Test → error level only.

---

## Test Approach

**Framework:** Jest 30 + React Testing Library 16

**Test style:** Integration-style unit tests — tests directly import and call controller functions (not route handlers), mocking only the HTTP layer and logger.

**Mocking pattern:**
```typescript
// Mock logger (prevent noise):
jest.mock('../utils/logger', () => {
  const noopLogger = { info: jest.fn(), error: jest.fn(), warn: jest.fn() };
  return {
    createControllerLogger: () => noopLogger,
    logRequest: jest.fn(),
    logResponse: jest.fn(),
    logErrorResponse: jest.fn(),
    logger: { child: jest.fn().mockReturnValue(noopLogger) },
  };
});

// Mock fetch (global):
function mockFetchOk(body: unknown) {
  (global.fetch as jest.Mock).mockResolvedValueOnce({
    ok: true, status: 200,
    text: async () => JSON.stringify(body),
    headers: { get: () => null },
  });
}
```

**Environment isolation:** `savedEnv` pattern — save `process.env` in `beforeEach`, restore in `afterEach`:
```typescript
let savedEnv: NodeJS.ProcessEnv;
beforeEach(() => { savedEnv = { ...process.env }; process.env.PARTNER_API_BASE_URL = '...'; });
afterEach(() => { process.env = savedEnv; });
```

**Test file location:** `tests/` (integration-style), `tests/unit/` (pure unit tests)

**Naming:** `<operation>.<type>.test.ts` — e.g. `createorder.integration.test.ts`, `apiError.test.ts`

---

## Cross-cutting Concerns

### Auth prerequisite

`handlePrerequisites(req)` in `utils/commonUtils.ts` — called at the top of every API route handler. Calls `getAccessToken()` from `imsTokenService.ts`. Returns `{ accessToken: string }`. Throws `ApiError(500)` if token fetch fails.

### Request ID forwarding

`forwardRequestIdHeader(res, requestId?)` — sets `X-Request-Id` response header when a request ID is available from the upstream Partner API. Called before every successful response and inside `ApiError` catch blocks.

### Correlation IDs

Every outbound `fetch` call generates fresh `uuidv4()` values for `X-Correlation-Id` and `X-Request-Id` headers. These are not read from the inbound request — they are always newly generated per outbound call.
