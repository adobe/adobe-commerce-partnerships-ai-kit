# Experience Card: Local Currency (Country-Specific Catalog)

- Catalog Browsing and Filters
- Country-Specific Pricing and Currency Selection

figma-url:

---

## Overview

Partners can browse **country-specific catalogs** to see the offers and partner prices available for a particular country, instead of relying only on a regional price list. A **Country/Region** picker in the Catalog lets partners switch between their regional catalog and eligible country-specific catalogs.

A country-specific price list can have different prices even when the currency stays the same. This release introduces country-specific **USD** pricing for India, South Korea, Türkiye, and Brazil.

This experience covers the Catalog page as a whole: opening the regional catalog, selecting filters, viewing offers and partner prices, and loading more results. Country-specific browsing extends that foundation with a Country/Region selection. Partners can use this experience to introduce a Catalog page or extend one they already have.

**Countries supported in this release:**

| Partner price region | Eligible countries | Country-specific pricing currency |
|---|---|---|
| Asia (`AP`) | India (`IN`), South Korea (`KR`) | `USD` |
| Eastern Europe (`EE`) | Türkiye (`TR`) | `USD` |
| Latin America (`LA`) | Brazil (`BR`) | `USD` |

Currency options come from partner configuration for every region. For **AP, EE, and LA**, the configured currency in this release is **USD**, so the filter shows a single USD option for regional and country-specific browsing. Regional pricing is a Country/Region selection, not a separate currency option.

Country options are limited to the countries listed for the partner's region above. Other regions retain regional browsing with no country options. Canada (`CA`)/CAD and Mexico (`MX`)/MXN are later-phase support and are not enabled in this release.

**Scope:** Catalog browsing, filters, country selection, price list retrieval, and display of the returned offers and prices. Where purchase actions already exist, preserve their behavior; building a cart, checkout, order submission, renewals, and returns are outside this experience.

---

## Goals

- Partners can discover which country-specific catalogs are available within their region.
- Partners can tell which country or region the displayed offers and prices apply to.
- Partners see the prices returned for their selected country.
- Partners can switch between country-specific and regional pricing.
- Partners can view either the current month's or next month's price list for the same country or region.

---

## Surface 1 - Catalog Page

### Entry point

The partner opens **Adobe catalog** from the navigation.

### Screen layout

The page contains:

- **Page heading** - Adobe catalog.
- **Filter panel** - Country/Region when available, Market segment, Currency, and Price list month.
- **Product grid** - one card per displayed offer, showing product details and partner pricing as described in Surface 3.
- **Results status** - indicates when more offers are loading and whether more results are available.


### Available markets and currencies

The partner's configured price region determines the regional catalog and eligible country options. Both market-segment and currency options come from partner configuration for all regions.

Examples of configured currencies:

| Partner price region | Configured currencies |
|---|---|
| Asia (`AP`), Eastern Europe (`EE`), Latin America (`LA`) | `USD` |
| Western Europe (`WE`) | `GBP`, `EUR` |
| North America (`NA`) | `USD` |

For example, a partner configured for Asia, Commercial, and USD can browse the Asia regional catalog or select India or South Korea, with Commercial and USD as the available filter options.

### Opening the Catalog

On entry:

1. Determine the partner's price region, supported market segments, and currencies from the partner configuration.
2. Default to the regional catalog, with no country selected. Use **Commercial** when supported, otherwise the first supported market segment.
3. Use the first configured currency when no valid selection already exists. A partner with only USD configured sees USD selected as the sole option.
4. Default **Price list month** to **Current month** and retrieve that month's standard price list for the selected region, market segment, and currency.
5. Show a loading state until the offers are ready, then display the product grid.

Do not request prices unless partner configuration supplies a region and non-empty lists of market segments and currencies. If required settings are missing or invalid, show the configuration error described in [Loading, Empty, and Error States](#loading-empty-and-error-states), rather than guessing defaults.

### Filters

| Control | Options and behavior |
|---|---|
| Market segment | Commercial (`COM`), Education (`EDU`), or Government (`GOV`), limited to the partner's configured market segments. Changing the selection retrieves that segment's price list. |
| Currency | A radio group separate from Country/Region, showing only the partner's configured currencies for every region. If only USD is configured, show only **USD**, selected by default. Changing currency retrieves that currency's price list while preserving Country/Region, market segment, and price list month. |
| Price list month | **Current month** (default) or **Next month**, each showing its month and year. Changing the selection retrieves that month's price list. |

### Price list month

Partners can switch between **Current month** and **Next month** for both regional and country-specific catalogs. Include the month and year in each option, for example **Current month (December 2026)** and **Next month (January 2027)**, so the selected pricing period is clear.

Changing the month retrieves its offers and prices from the first page, preserving Country/Region, market segment, and currency. Changing other filters preserves the selected month. Next month means the next calendar month, including the year change after December.

If next month's price list is not yet available, show a clear message that the selected month's price list is unavailable. Keep **Next month** selected; do not silently show current-month prices instead.

### Browsing more offers

Retrieve offers in pages using the price list API's `offset` and `limit`. As the partner approaches the bottom of the grid, load the next page and append offers without removing those already displayed.

Keep the selected filters applied to each additional page. If a page has no entry-level offers to display but more offers are available, continue loading until displayable offers are found or the price list is exhausted. Show **No offers found** only when there are no displayable offers and no more pages to check.

Changing the country or region, market segment, currency, or price list month starts browsing the corresponding price list from its first page. Do not mix offers from the previous selection into the new results.

---

## Surface 2 - Country/Region Selection

### Entry point

**Country/Region** appears as the first filter on the Catalog page when the partner's region has eligible countries.

### Availability
| Condition | Experience |
|---|---|
| Eligible countries configured for the partner region | The picker shows the partner region followed by eligible countries. |
| No eligible countries configured for the partner region | The picker is not shown. Regional Catalog browsing remains available. |

### Country/Region options

Derive country availability from the configured mapping in the [Overview](#overview).

The configuration defines each country's code, display name, associated partner price region, and country-specific pricing currency using the **Countries supported in this release** table in the Overview: `AP` maps to `IN` and `KR`, `EE` maps to `TR`, and `LA` maps to `BR`. All four country-specific price lists use `USD`. No other countries are enabled in this release.

Only countries associated with the partner's price region appear in the picker, sorted alphabetically by display name.

The picker contains:

- **Partner region** - the default selection, displayed using its readable name, such as **North America** for `NA` or **Asia** for `AP`. An unmapped region displays its code.
- **Eligible countries** - displayed by country name, with the country code used when requesting prices.


### Selecting a country

When the partner selects a country:

1. The picker displays the selected country.
2. The Catalog requests that country's price list, using its country code in the request's `region` field.
3. The product grid uses the results for the selected pricing context, starting from the first page.

The selected currency, market segment, and price list month are preserved. For partners configured with USD only, currency remains **USD** when switching between a regional catalog and its eligible countries.

### Returning to regional pricing

Selecting the partner region again removes the country override and requests the regional price list. The other filters remain unchanged.

A manually selected country is cleared when the active organization or partner region changes. A selection that is no longer eligible under the hardcoded configuration is also cleared. The previous organization's or region's country selection must not carry into the new pricing context.

---

## Surface 3 - Offers and Partner Pricing

### Price list request

The Catalog retrieves the selected price list through Adobe's `POST /v3/pricelist`. The pricing context is:

| Field | Value |
|---|---|
| `region` | Selected country code, or the partner region when regional pricing is selected |
| `currency` | Selected configured currency |
| `marketSegment` | Selected market segment as `COM`, `EDU`, or `GOV` |
| `priceListType` | `STANDARD` |
| `priceListMonth` | Selected current or next calendar month in `YYYYMM` format |

For example, selecting Türkiye with USD and Commercial requests `region: "TR"`, `currency: "USD"`, and `marketSegment: "COM"`. Returning to regional pricing replaces `"TR"` with the partner's regional code.

Prices shown must always belong to the selected country or region, currency, market segment, and price list month. Loading more offers or switching market segments must retain the selected country and month. Cached or prefetched results must not cause another country's, region's, or month's prices to appear under the current selection.

### Product prices

Each product card shows:

- Product name, icon, offer ID, and additional details where available.
- **Level 1 partner price:** the offer's `partnerPrice`, formatted in the selected currency, per year and per license.
- Where purchase actions are available, **Buy**, or the quantity already in the cart.

Show the entry-level offers appropriate to this Catalog's **Level 1 partner price** presentation: discount-level suffix `01` for licenses or `T1` for transactions. Do not present every volume-discount tier as a separate product card.

Prices are returned by the selected price list; the UI does not convert an amount from one currency to another. Infinite scrolling retains the selected pricing context.

Display a pricing disclaimer above the results:

*"Prices below are PARTNER PRICING based on a level 1 annual license term. Final PARTNER PRICES determined by customer's discount level."*

---
## Loading, Empty, and Error States

| State | Experience |
|---|---|
| Partner configuration missing or invalid | Show *"Catalog configuration is missing or invalid. Check the region, market segments, and currencies in partner configuration."* Do not request a price list until the configuration is corrected. |
| Price list loading | Show product-card skeletons. Additional-page loading shows more skeletons below existing results. |
| No displayable offers | Show *"No offers found"* after all available pages have been checked for the selected pricing context. |
| Price list failure | Show *"Error loading catalog"* and the returned error message, or *"Failed to load products. Please try again."* if none is available. |
| Selected month's price list unavailable | Show *"Price list for {month year} is not available."* Keep the selected month and do not substitute another month's offers. |

Price list failures, including invalid country/currency combinations or unavailable price lists, use the Catalog error presentation above. Do not silently retry with another currency or region, or display the previous selection's offers as if they belonged to the failed selection.

Country options are resolved locally from the hardcoded configuration. There is no country-configuration API loading state, failure alert, or retry action.

---

## Acceptance Criteria

1. Partners can browse regional and eligible country-specific catalogs using configured market segments and currencies, switch between current and next month's price lists, and see the correct offers and partner prices throughout filtering and pagination, with clear loading, empty, and error states.

---