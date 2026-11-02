# DB_SCHEMA.md — adobe-commerce-partnerships-ref-app

## 1. Overview

**This service owns no data store.** No ORM, query builder, or raw-SQL access was found (no
`prisma`, `sequelize`, `mongoose`, `typeorm`, `knex`, or DB driver package in `package.json`; no
`DATABASE_URL`/`DB_HOST`/connection-string env vars in `.env.sample`). adobe-commerce-partnerships-ref-app is a stateless
Next.js backend-for-frontend that proxies and re-shapes data from the Adobe Commerce Partner API
(see `CONNECTORS.md`) — it does not persist anything itself beyond process-lifetime, in-memory
caches (see `PLATFORM.md → ## 4. Caching`).

Sections 2–5 (Connection & Deployment, Entity → Table Mapping, Tables, Relationships) are omitted
per the no-data-store rule.