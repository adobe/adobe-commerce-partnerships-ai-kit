# DB Schema — Bridge

## 1. Overview

Bridge owns no data store. It is a stateless proxy — all persistent data lives in the VIP Marketplace Partner API. There are no databases, no migrations, and no ORM configurations in this service.

Data shapes are documented as Zod schemas in `models/` for validation purposes only; they do not map to any owned tables.
