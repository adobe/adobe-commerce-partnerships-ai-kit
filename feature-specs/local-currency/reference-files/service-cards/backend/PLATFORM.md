# PLATFORM.md — adobe-commerce-partnerships-ref-app

## 1. Deployment Topology

NOT CAPTURED — requires the actual deployment manifests (Kubernetes/Helm/Terraform/CDK or CI/CD
pipeline config). No `deploy/`, `infra/`, `ops/`, `k8s/`, or `.github/workflows/` directory exists
in this repo. What *is* captured, from the files present:

| Artifact | Detail |
|---|---|
| `Dockerfile` | `node:20-alpine` base; `npm install` → `npm run build` → `CMD ["node", "server.js"]`; exposes `9000` and `8080` (only `9000` is actually bound by `server.js`) |
| `docker-compose.yml` | Single `app` service, built from the local `Dockerfile`, env loaded from `.env`, ports `9000:9000` and `8080:8080` mapped, bind-mounts the repo and `./certs`, `restart: unless-stopped` |
| Entry point | `server.js` — custom Node HTTP server wrapping Next.js's request handler; binds `0.0.0.0:9000`; adds a plaintext `/ping` health-check route returning `200 OK` |
| Ownership / on-call team | NOT CAPTURED — requires org/team metadata not present in this repo |

## 2. Monitoring & Observability

| Aspect | Detail |
|---|---|
| Logging | Pino (`utils/logger.ts`) — structured JSON in production, pretty-printed in development. Every API-route and controller call is logged via `createAPILogger`/`createControllerLogger` plus `logRequest`/`logResponse`/`logErrorResponse`. No log shipping/aggregation config (e.g. Datadog, CloudWatch, ELK) found in this repo — assumed to be handled by the deployment platform, not application code. |
| Health check | `GET /ping` (plain text `OK`, handled directly in `server.js`, bypassing Next.js routing) |
| Metrics / APM | NOT CAPTURED — no metrics library (Prometheus client, StatsD, OpenTelemetry) found in dependencies |
| Tracing / correlation | Manual, not automatic: each outbound call generates a fresh `X-Correlation-Id`/`X-Request-Id` (see `CONNECTORS.md`); the upstream's response `X-Request-Id` is read and forwarded back to the adobe-commerce-partnerships-ref-app API caller via the `x-request-id` response header (`utils/commonUtils.ts::forwardRequestIdHeader`) |

## 3. Feature Flags

None found — no feature-flag library (LaunchDarkly, Unleash, Split, etc.) in `package.json`, and
no custom flag/toggle mechanism in `utils/` or `controllers/`.

## 4. Caching

| Cache | Scope | TTL | Notes |
|---|---|---|---|
| IMS access token | In-memory module-level variable (`utils/imsTokenService.ts`) | Until 5 minutes before token `expires_in` elapses | Not shared across process instances/replicas — each running instance re-authenticates independently on cold start |
| Public env vars | In-memory module-level variable (`pages/api/env.ts`) | 60 seconds | Serves `GET /api/env`; sets `Cache-Control: public, max-age=60, must-revalidate` and an `X-Cache: HIT`/`MISS` response header |

No distributed cache (Redis/Memcached) or CDN-level caching config found.

## 5. Database

None — see `DB_SCHEMA.md`. This service is stateless; all persisted business data lives in the
upstream Adobe Commerce Partner API.

## 6. Active Migrations

None — no data store, so no schema migrations. No in-code feature-migration markers (e.g.
deprecated-endpoint flags) found either.

## 7. Runbooks

NOT CAPTURED — no runbook documents found in this repo (no `RUNBOOK.md`, `docs/runbook*`, or
equivalent). Operational guidance is limited to `README.md`'s Configuration/Development/Docker
sections.