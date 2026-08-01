# Platform — Bridge

## 1. Deployment Topology

| Field | Value |
|-------|-------|
| Runtime | Node.js HTTP server (custom `server.js` wrapping Next.js) |
| Port | 9000 |
| Container | Docker (`Dockerfile` + `docker-compose.yml` present) |
| Start command (local) | `npm run dev:local` → `NODE_ENV=development NEXT_PUBLIC_APP_ENV=local node server.js` |
| Start command (production) | `npm run build && npm start` (standard Next.js) or via Docker |
| Health check endpoint | `GET /ping` → `200 OK` (`text/plain: OK`) |

---

## 2. Docker

`Dockerfile` and `docker-compose.yml` are present in the repo root.

- Container exposes port 9000
- Environment variables injected at container runtime via Docker env or compose `environment` section
- No persistent volumes (stateless service)

---

## 3. Monitoring & Logging

| Aspect | Detail |
|--------|--------|
| Log library | Pino 9.9.5 |
| Log format (dev) | pino-pretty — colourised, human-readable to stdout |
| Log format (prod) | Structured JSON to stdout |
| Log level (dev) | `debug` |
| Log level (prod) | `info` (or `LOG_LEVEL` env override) |
| Log base fields | `service: bridge-app`, `version`, `environment`, `pid`, `hostname` |
| Correlation ID | `X-Correlation-Id` header forwarded from inbound request to Pino child logger fields |
| Request ID | `X-Request-Id` extracted from Partner API responses and forwarded to callers |
| Error logging | `logger.error({ err })` in all catch blocks |

---

## 4. Feature Flags

None detected — no feature flag framework in use.

---

## 5. Caching

| Cache | Mechanism | TTL |
|-------|-----------|-----|
| IMS access token | Module-level variables in `imsTokenService.ts` | `expires_in - 5 min` |
| `/api/env` response | In-memory object (`envCache`) | 60 seconds |

No distributed cache (Redis, Memcached, etc.) — all caches are in-process and cleared on restart.

---

## 6. Database

Bridge owns no data store. See `DB_SCHEMA.md`.
