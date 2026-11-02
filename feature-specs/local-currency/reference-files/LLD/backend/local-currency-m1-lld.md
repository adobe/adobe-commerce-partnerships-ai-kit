# Local Currency Catalog - Backend LLD

**No backend changes are required.**

The existing `POST /api/pricelist` endpoint forwards `region`, `currency`, `marketSegment`, `priceListType`, and `priceListMonth` to Adobe's `POST /v3/pricelist`. The UI can send a country code such as `IN` or the partner's regional code such as `AP` using the same API.

Reuse `GET /api/partnerDetails` for configured market segments, currencies, and price region.

Country availability, selection, pricing month, and Catalog display behavior are UI responsibilities. Existing backend behavior remains unchanged; no fixes, refactoring, new endpoints, configuration changes, or DB changes are included.
