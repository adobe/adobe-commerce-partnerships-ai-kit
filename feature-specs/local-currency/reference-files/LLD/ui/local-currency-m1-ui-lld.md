# Local Currency Catalog - UI LLD

## 1. Summary

Add two additional filters to the existing `/catalog` page: **Country/Region** and **Price list month** with current/next month options. Wire their selected values into the existing price-list request, query key, and prefetch paths. All other Catalog behavior is already in place and remains unchanged.

**No Figma URL — LLD derived from experience card description only.** New controls reuse existing Spectrum filter sections and CSS Modules; visual details are approximate.

Reuse `GET /api/partnerDetails` and `POST /api/pricelist` as described in `docs/ai-kit/LLD/backend/local-currency-lld.md`. **No backend changes are required.**

## 2. Data Flow

### Select Country/Region

```text
User selects a country or the partner region
  -> CatalogFiltersComponent updates the controlled country selection
  -> CatalogPage derives effectiveRegion = selectedCountry ?? partnerRegion
  -> existing price-list query uses a key containing effectiveRegion and selected month
  -> POST /api/pricelist with region: effectiveRegion
  <- existing offers/prices response
  <- existing Catalog grid renders the selected context
```

Default to the partner region. Preserve the selected market segment, currency, and month. Country/region changes start browsing from the first page.

### Select current or next month

```text
User selects Price list month
  -> CatalogFiltersComponent updates the controlled YYYYMM value
  -> existing price-list query uses a key containing the selected month and effectiveRegion
  -> POST /api/pricelist with priceListMonth: selectedMonth
  <- existing offers/prices response, or selected-month failure
  <- existing grid/error presentation renders for that selection
```

Default to current month. Preserve Country/Region, market segment, and currency. Month changes start browsing from the first page. Additional-page requests and background prefetch retain both new selections.

## 3. Change Summary

| File                            | Action | Layer      | Reason                                                                        |
| ------------------------------- | ------ | ---------- | ----------------------------------------------------------------------------- |
| `models/Catalog.ts`             | MODIFY | State      | Add country and pricing-month fields to existing UI filter state.             |
| `utils/catalogUtils.ts`         | CREATE | Util       | Define country options, region labels, and current/next month options.        |
| `components/CatalogFilters.tsx` | MODIFY | Component  | Render and update the two new controls.                                       |
| `pages/catalog.tsx`             | MODIFY | Route      | Initialize selections and pass them to existing query/filter integrations.    |
| `utils/prefetchProducts.ts`     | MODIFY | Data Fetch | Include selected region/month in existing requests, keys, and prefetch calls. |

### `models/Catalog.ts` - MODIFY

Extend `CatalogFiltersSchema`, retaining the existing inferred `CatalogFilters` type:

```ts
country: z.string().nullable().default(null),
priceListMonth: z.string(),
```

- `country: null` means the configured partner region; otherwise store the selected country code.
- `priceListMonth` is the actual selected `YYYYMM`, not the visible label.
- Preserve all existing fields and their behavior. Do not modify the product schema, backend price-list schemas, or cart types.

### `utils/catalogUtils.ts` - CREATE

Keep the two controls' shared option logic in one Catalog utility:

```ts
export interface CatalogFilterOption {
  value: string;
  label: string;
}

export const COUNTRY_CATALOGS = [
  { code: 'IN', label: 'India', partnerRegion: 'AP' },
  { code: 'KR', label: 'South Korea', partnerRegion: 'AP' },
  { code: 'TR', label: 'Türkiye', partnerRegion: 'EE' },
  { code: 'BR', label: 'Brazil', partnerRegion: 'LA' },
] as const;

export function getCountryOptions(partnerRegion: string): CatalogFilterOption[];
export function getRegionDisplayName(partnerRegion: string): string;
export function getPriceListMonthOptions(currentMonth: string): CatalogFilterOption[];
```

- `getCountryOptions` filters the mapping by partner region and sorts countries alphabetically by display name. No Canada or Mexico option.
- Region labels include Asia (`AP`), Eastern Europe (`EE`), Latin America (`LA`), North America (`NA`), and Western Europe (`WE`). Fall back to the region code when unmapped.
- Month options use the current `YYYYMM` supplied by existing `getCurrentPriceMonth()`. Derive the following calendar month with correct year rollover, using a first-of-month date.
- Return current month first, with labels such as `Current month (December 2026)` and `Next month (January 2027)` and values `202612` and `202701`.
- Do not derive or change currencies from country selection. No network request or persistence is added.

### `components/CatalogFilters.tsx` - MODIFY

Retain the existing `filters` and `onFiltersChange` props, existing partner-context reads, and current controls. Add only the option inputs needed for the new controls:

```ts
countryRegionOptions: CatalogFilterOption[];
monthOptions: CatalogFilterOption[];
```

- **Visual specification — approximate:** reuse existing `FilterSection`, Spectrum `RadioGroup`/`Radio`, and CSS Module. Insert Country/Region as the first section and Price list month after Currency; preserve the other sections and their behavior.
- Country/Region options contain the partner-region code/readable name first, followed by eligible country codes/names. Hide the section entirely when there are no eligible countries.
- The controlled radio value is `filters.country ?? countryRegionOptions[0].value`. Selecting the region stores `null`; selecting a country stores its code.
- Price list month uses `filters.priceListMonth` and the supplied current/next options.
- Add explicit scalar cases for `country` and `priceListMonth` to the existing change handler. Spread existing filters so changing either preserves all other values.
- Keep accessible group labels **Country/Region** and **Price list month** and Spectrum's existing keyboard behavior.
- Do not change market-segment/currency options, defaults, validation, or existing category/type controls.

### `pages/catalog.tsx` - MODIFY

- Initialize `filters.country` to `null` and `filters.priceListMonth` to current `YYYYMM` using existing `getCurrentPriceMonth()`. Derive the month options from that same current-month anchor so labels and request values agree.
- Read the existing configured partner region and derive region-first country options with the new utility.
- Derive `effectiveRegion` from an eligible selected country or the partner region. Clear the stored country when partner region changes or the country is no longer eligible. Ensure old-region country state cannot issue a request for the new region before reset.
- If an existing host organization-change/reset boundary is available, clear country selection there as well. Do not introduce an organization identity model or switching UI.
- Pass the two option lists and existing controlled filter props to `CatalogFiltersComponent`.
- Extend the existing combined request filters with `region: effectiveRegion` and selected `priceListMonth`. Update active-query and prefetch call sites to use those same values.
- Changing country/region or month starts at the first page, including when revisiting a cached selection. Reset the target infinite-query pages as needed using the shared key helper; never display previous-selection placeholder offers under the new controls.
- Existing market-segment/currency changes preserve the new country/month fields. Additional-page requests retain them.
- Retain the selected month on failure. For an initial-page HTTP 404, show `Price list for {month year} is not available.` using the existing error presentation. Other failures, including additional-page failures, retain existing error handling; do not silently substitute current month or another pricing context.
- Preserve the existing query readiness checks, pagination algorithm, offer filtering, skeletons, disclaimer, product cards, status display, search, cart, and purchase handlers.
- **Visual specification — approximate:** no page layout or stylesheet changes; only the two additional filter sections.

### `utils/prefetchProducts.ts` - MODIFY

Extend the existing fetch helper and its filters rather than creating another fetch unit:

- Add required `priceListMonth` to its request-filter type through the updated `CatalogFilters`. Continue supplying effective `region` from the page.
- In the existing `POST /api/pricelist` body, use the passed `filters.region` and `filters.priceListMonth` instead of computing the current month inside the fetcher.
- Keep `marketSegment`, `currency`, `priceListType: 'STANDARD'`, `offset`, `limit`, `includeOfferAttributes`, same-origin headers, and response handling otherwise unchanged. Do not send `country` as a separate backend field.
- Extend the existing key helper with region and month while preserving its original prefix and currency position:

  ```ts
  getPricelistQueryKey(
    marketSegment: string,
    currency: string,
    region: string,
    priceListMonth: string
  )

  // Result:
  ['pricelist-infinite', segmentCode, currency, region, priceListMonth]
  ```

- Update active query, cache checks, and all existing prefetch helper calls to use that same key and selected region/month. Prefetching a different market segment must retain country/region, currency, and month.
- Preserve existing prefetch scheduling, cache freshness/GC times, retry policy, currency invalidation, page-index/offset calculation, and continuation behavior. No pagination or prefetch refactoring.
- Retain HTTP status and requested page on fetch errors using a typed error so the page can distinguish initial-page 404 for month availability. Keep the existing error message behavior; no new API contract or error envelope.

## 4. Design Decisions

| Decision                                             | Why                                                                           | Trade-off                                                  |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------- |
| Extend existing filters and fetch helper             | Only two controls and their wiring are missing.                               | Existing Catalog behavior stays unchanged.                 |
| Keep configured currencies unchanged                 | Country selects a price list, not a new currency option.                      | Same-currency catalogs still need separate cache entries.  |
| Add region/month to active and prefetched query keys | Prevent another country's or month's prices from appearing for the selection. | Cache entries are separated by the two added dimensions.   |
| Reuse existing radio sections/styles                 | No Figma URL is supplied.                                                     | Visual details are approximate rather than Figma-verified. |

Organization switching is not modeled in the current partner context. Do not create that capability as part of these filters; use an existing host reset boundary if one becomes available.

Implementation checks should focus only on country eligibility/default/reset, current/next month labels and year rollover, preservation of other filters, selected values in requests/prefetch/cache keys, first-page reset, and retained month selection on an unavailable-month response.

## 5. Acceptance Criteria Coverage

| Goal (from experience card)                                                                                                                                                                                                                                                                               | Covered by                                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Partners can browse regional and eligible country-specific catalogs using configured market segments and currencies, switch between current and next month's price lists, and see the correct offers and partner prices throughout filtering and pagination, with clear loading, empty, and error states. | New country/month controls and their request/cache wiring are covered by the five files in §3. Other Catalog behavior is already implemented and reused unchanged, per the clarified scope. |

## 6. Out of Scope

All changes other than the two filters and their directly required request/cache/prefetch wiring: backend changes, product-card labels/rendering, offer-tier filtering, price formatting, disclaimer/status copy, pagination fixes, general loading/error/configuration improvements, market-segment/currency behavior, cart/checkout changes, and organization-switching infrastructure.
