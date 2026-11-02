# Access Country Specific Price Lists

Partners can now retrieve country-specific price lists for accurate quoting. Depending on the `country`, the price list uses USD or a local currency. See [Local currency support](#local-currency-support) for pricing, subscription, and renewal behavior.

## Fetch Price List API

| Endpoint      | Method |
|---------------|--------|
| /v3/pricelist | POST   |

**Sample request URL:** `POST <env root url>/v3/pricelist?offset=0&limit=10`

The `POST /v3/pricelist` API retrieves the price list information for the partners.

### Integration Guide

Partners will receive email notifications whenever updates are made to the Price List. Upon receiving the notification, partners can initiate a request to the Fetch Price List API by following the steps below:

1. Initiate API request

   Use the `POST /v3/pricelist` API. The request must include required parameters, with optional filters available to refine the response. See [Request](#request-body) for the complete list of supported parameters.

2. Handle API errors

   If the API returns an error, refer to the [Error Handling](#http-status-codes) section for guidance on interpreting and resolving error codes.

3. Parse the response

   The API returns a JSON response. Partners should parse this response and process the data outlined in the [Response](#response-body) section.

4. Pagination support

   The API supports pagination. Refer to the pagination guidelines to understand how to navigate through paged results efficiently.

5. Store Price List data

   Partners are expected to store the Price List data in their internal systems. Key attributes to store include:

    - `offerId` – Unique identifier for the offer; required when placing orders through [Order](../order-management/create-order.md) APIs.
    - `productFamily` – Helps categorize and group offers that belong to the same product family.

   Additional attribute details are available in the [Response](#response-body)  section.

6. Track Price List changes

   To identify changes to the Price List, review the following fields:

    - `acdIndicator`: Indicates whether the offer is new, changed, or deleted.
    - `acdEffectiveDate`: The date on which the acdIndicator status was set.

## Request

### Request header

| Parameter        | Description                                                                                                                                                                                                                      |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| X-Request-Id     | A unique identifier for the call. The value should be reset for every single request. If this is not provided, then a request ID will be automatically generated. Using a duplicate request ID may return an error.              |
| X-Correlation-Id | **Required**. A unique identifier for the call. This is to ensure idempotency. In the case of a timeout, the retry call could include the same value. Upon receiving some response, the value should be reset for the next call. |
| Accept           | **Required**. Specifies the response type. Must be "application/json" for proper usage.                                                                                                                                          |
| Content-Type     | **Required**. Specifies the request type. Must be "application/json" for proper usage.                                                                                                                                           |
| Authorization    | **Required**. Authorization token in the form `Bearer <token>`                                                                                                                                                                   |
| X-Api-Key        | **Required**. The API Key for your integration                                                                                                                                                                                   |

### Query parameters

You can use the query parameters such as `limit` and `offset` in the request URL. For example: `POST <env root url>/v3/pricelist?offset=0&limit=10`

| **Field**                        | **Required** | **Type**           | **Description** |
|----------------------------------|--------------|--------------------|-----------------|
| `limit`            | No           | Integer            | Specifies the number of items to be returned per page, enabling pagination. The default value is 20.|
| `offset`       | No           | Integer            |  Specifies the starting point for retrieving items in a paginated list. It determines how many items to skip before beginning to return results. If not specified, the default value is 0, meaning the response will start from the first item. |

### Request body

A sample request is as follows:

```json
{
    "region": "NA",
    "marketSegment": "GOV",
    "priceListType": "3YC",
    "currency": "USD",
    "priceListMonth": "202601",
    "filters": {
        "offerId": "30001475CC01A12"
    },
    "includeOfferAttributes": [
        "productFamily",
        "productType",
        "productTypeDetail",
        "operatingSystem",
        "language",
        "version",
        "users",
        "metric",
        "bridge",
        "partNumber",
        "acdIndicator",
        "acdEffectiveDate",
        "levelDetails",
        "firstOrderDate",
        "lastOrderDate",
        "partnerPrice",
        "estimatedStreetPrice",
        "discountCode",
        "estimatedShipDate",
        "publicAnnounceDate",
        "rmaRequestDeadline",
        "acdDescription",
        "pool",
        "duration",
        "additionalDetail"
    ]
}
```

#### Request parameters

| **Field**                        | **Required** | **Type**           | **Description** |
|----------------------------------|--------------|--------------------|-----------------|
| `region` | Yes | String (Enum) | Specifies the region or country for which the Price List should be returned. The value must be supported for the Partner. \<br /\>- Supported regional codes: `AP`, `WE`, `LA`, `EE`, `PA`, `NA`. \<br /\> - Supported country codes: `JP`, `CN`, `MX`, `KR`, `TR`, `BR`, `IN`, `CA`. See [Local currency support](#local-currency-support) for country-specific pricing and currencies. |
| `marketSegment`                  | Yes          | String (Enum)      | Specifies the market segment. Must be one of the supported segments, such as: \<br /\> - COM \<br /\> - EDU \<br /\> - GOV. |
| `priceListType`                   | No           | String (Enum)      | Indicates the price list type. Possible values: \<br /\> - STANDARD \<br /\> - 3YC \<br /\> - STD-LGA (Standard Price List for Large Government Agencies)|
| `currency`                       | Yes          | String (Enum)      | Specifies the currency. Must be one of the supported currencies for the Partner and applicable price list. \<br /\>Possible Values: AUD, CAD, EUR, GBP, JPY, MXN, USD. |
| `priceListMonth`                | Yes          | String             | Specifies the month you want the Price List for, in `YYYYMM` format. Example: `202410`. |
| `filters`                        | No           | Object             | Filters to narrow down the Price List. All filters use AND logic. For example, if `offerId` and `productFamily` filters are included in the request, then the API response will include results that match `offerId` AND `productFamily` |
| `filters → offerId`             | No           | String             | Returns only offers matching the specified `offerId`. |
| `filters → productFamily`       | No           | String             | Returns only offers matching the exact, case-sensitive `productFamily`. |
| `filters → firstOrderDate`      | No           | Date               | Returns only those offers whose first order date is later than the specified date. |
| `filters → lastOrderDate`       | No           | Date               | Returns only those offers whose last order date is on or after the specified date. |
| `filters → discountCode`        | No           | String             | Returns only offers matching the specified discount code. |
| `includeOfferAttributes`        | No           | List  | List of additional offer attributes to include in the response. Unknown attributes will result in an error. Read more about how to [define offer data to be displayed in the response](#define-offer-data-to-be-displayed-in-the-response). |

### Filter

You can refine the data returned in the response by using the filters property in the request body. This allows you to narrow down the price lists based on specific criteria. All filter fields are combined using AND logic, meaning that only results matching all specified conditions will be returned if multiple filters are applied. Filters with null or empty values will be ignored. You can filter the data using the following parameters:

- `offerId`
- `productFamily`
- `firstOrderDate`
- `lastOrderDate`
- `discountCode`

### Define offer data to be displayed in the response

To customize the offer details returned in the response, use the `includeOfferAttributes` parameter in the request body. This allows you to specify which offer attributes should be included in the response.  For example:

- `productType`
- `productTypeDetail`
- `language`

**Notes:**

- The response will include only the attributes listed in `includeOfferAttributes`, plus these default attributes:

    - `totalCount`
    - `limit`
    - `offset`
    - `offers`
    - `offer → offerId`
    - `offer → discountCode`
    - `offer → productFamily`
    - `offer → firstOrderDate`
    - `offer → lastOrderDate`
    - `offer → partnerPrice`
- Any request parameter that has no value is omitted from the response.

## Response body

**Disclaimer – Offer Attributes in Response**

- When using the `includeOfferAttributes` parameter in the request, the API will attempt to return the specified attributes for each offer. However, it is not guaranteed that every offer in the response will contain all the requested attributes.
- The availability of each attribute depends on the underlying data applicability and validity for the specific offer. If an attribute is not relevant or does not have a valid (non-null) value for a given offer, it may be omitted from that offer’s attribute section in the response.
- Requesting all available `includeOfferAttributes` does not ensure that all attributes will be present for every offer. Consumers of the API should design their integrations to handle missing or null attribute values gracefully.

A sample response is as follows:

```json
{
    "priceListMonth": "202601",
    "marketSegment": "GOV",
    "region": "NA",
    "currency": "USD",
    "priceListType": "3YC",
    "totalCount": 2,
    "count": 2,
    "limit": 100,
    "offset": 0,
    "offers": [
        {
            "offerId": "30001475CC01A12",
            "productFamily": "Creative Cloud For Enterprise All Apps",
            "productType": "Subscription New",
            "productTypeDetail": "Annual",
            "additionalDetail": "GOV | First Order Date and Last Order Date=3-year commit accept. RMA Request Deadline=Last eligible order date.",
            "operatingSystem": "Multiple Platforms",
            "language": "Multi NorthAmerican Language",
            "version": "ALL",
            "users": "1 User",
            "levelDetails": "Level 1 1 - 9",
            "firstOrderDate": "2022-05-03T00:00:00.000+00:00",
            "lastOrderDate": "2023-11-01T00:00:00.000+00:00",
            "partnerPrice": "1206.84",
            "estimatedStreetPrice": "1283.88",
            "pool": "Application",
            "duration": "12 Months"
        },
        {
            "offerId": "30001475CC01A12",
            "productFamily": "Creative Cloud For Enterprise All Apps",
            "productType": "Subscription New",
            "productTypeDetail": "Annual",
            "additionalDetail": "GOV | First Order Date and Last Order Date=3-year commit accept. RMA Request Deadline=Last eligible order date.",
            "operatingSystem": "Multiple Platforms",
            "language": "Multi NorthAmerican Language",
            "version": "ALL",
            "users": "1 User",
            "levelDetails": "Level 1 1 - 9",
            "firstOrderDate": "2023-11-02T00:00:00.000+00:00",
            "lastOrderDate": "2024-08-01T00:00:00.000+00:00",
            "partnerPrice": "1263.24",
            "estimatedStreetPrice": "1343.88",
            "pool": "Application",
            "duration": "12 Months"
        }
    ]
}
```

### Response parameters

| **Field** | **Not Null** | **Type** | **Description** |
|-----------|--------------|----------|-----------------|
| `priceListMonth` | Yes | String | Price List published month |
| `marketSegment` | Yes | String (Enum) | Market segment applicable for the offers included in the response. Possible values: \<br /\> - COM \<br /\> - EDU \<br /\> - GOV |
| `region` | Yes | String (Enum) | For regional price list requests, the region where the offer is sold. \<br /\> Possible values: \<br /\> - NA \<br /\> - PA \<br /\> - EE \<br /\> - MX \<br /\> - LA \<br /\> - WE \<br /\> - JP \<br /\> - AP |
| `currency` | Yes | String (Enum) | Currency applicable for the market segment and region or country. Must be supported for the partner and applicable price list. \<br /\> Possible Values: \<br /\> - AUD \<br /\> - CAD \<br /\> - EUR \<br /\> - GBP \<br /\> - JPY \<br /\> - MXN \<br /\> - USD |
| `priceListType` | Yes | String (Enum) | The price list type. \<br /\> Possible values:\<br /\> - STANDARD \<br /\> - 3YC \<br /\> - STD-LGA (Standard Price List for Large Government Agencies) |
| `totalCount` | Yes | Integer | Indicates the total items matched as per the search criteria. |
| `count` | Yes | Integer | Indicates the total number of offers returned in the response. |
| `limit` | Yes | Integer | The number of offers limited in the response. |
| `offset` | Yes | Integer | Specifies the starting point for retrieving items in a paginated list. It determines how many items to skip before beginning to return results. If not specified, the default value is 0, meaning the response will start from the first item. |
| `offers` | Yes | JSON List | List of Offers \<br /\> List will be empty if no Offers are available to return. |
| `offer → offerId` | Yes | String | Unique identifier of the Offer. Used as Offer ID in Order APIs |
| `offer → productFamily` | Yes | String | Indicates the grouping of products based on product code, product config, and so on. |
| `offer → productType` | Yes | String (Enum) | Possible values: \<br /\> - Enterprise Consumption (consumable) \<br /\> - Enterprise Hosted Subscription New \<br /\> - Feature Restricted Licensing Subscription New \<br /\> - Hosted Subscription \<br /\> - Subscription New \<br /\> - Team Consumption (consumable) \<br /\> - Transaction New (consumable) |
| `offer → productTypeDetail` | Yes | String (Enum) | Indicates whether the product is term base and if price is fixed or prorated. \<br /\> Possible values: \<br /\> - 12-month, no proration (fixed price) \<br /\> - Annual (proration based on duration) \<br /\> - No Proration (fixed price, no term) |
| `offer → additionalDetail` | No | String | Additional details for the product offer, such as the number of credits in a pack, the number of license bundles in a subscription, and so on. |
| `offer → operatingSystem` | Yes | String (Enum) | Possible values: \<br /\> - Multiple Platforms \<br /\> - Other \<br /\> - Windows |
| `offer → language` | Yes | String (Enum) | Language offered by Offer \<br /\> The following are the possible values: \<br /\> - EU English \<br /\> - Japanese \<br /\> - Multi Asian Languages \<br /\> - Multi European Languages \<br /\> - Multi Language Australia \<br /\> - Multi Latin American Languages \<br /\> - Multi NorthAmerican Language |
| `offer → version` | Yes | String | Possible values: \<br /\> - ALL |
| `offer → users` | Yes | String (Enum) | License type \<br /\> The Following are the possible values: \<br /\> - 1 User \<br /\> - Named \<br /\> - Per Credit Pack \<br /\> - Per Server \<br /\> - Per Transaction \<br /\> - Per Workstation \<br /\> - Subscription |
| `offer → levelDetails` | Yes | String | Level Description of Min and Max range for the price point. |
| `offer → firstOrderDate` | Yes | Date | The first date for the item's sale. For a country-specific price list, the date the offer's country-specific price applies to orders. For 3YC pricing, use the customer's applicable partner price lock date. |
| `offer → lastOrderDate` | Yes | Date | The last date for the item's sale. |
| `offer → partnerPrice` | Yes | String | List Price for Partner \<br /\> |
| `offer → estimatedStreetPrice` | Yes | String | Estimated retail price \<br /\>  |
| `offer → pool` | Yes | String (Enum) | The category or grouping to which the offer belongs. This field helps in identifying the broader classification of the offer. Possible values are: \<br /\> - Application \<br /\> - Discount1 \<br /\> - Discount2 \<br /\> - Pricing Version 23 \<br /\> - Pricing Version 24 \<br /\> - Pricing Version 25 \<br /\>|
| `offer → duration` | Yes | String | The time period for which the offer is valid or applicable. This field specifies the length of time the offer is effective. |

## HTTP status codes

| Status code | Description                                       |
| ----------- | ------------------------------------------------- |
| 200         | Successfully retrieved the price list.            |
| 400         | Bad request                                       |
| 401         | Invalid Authorization token                       |
| 403         | Invalid API Key                                   |
| 404         | Price list not found for the given request/offset |

## Error codes specific to Fetch Price List API

The following table lists the error codes that are specific to the Fetch Price List API:

| Code | Message | Applicable API calls | HTTP Code |
|--|--|--|--|
|1159 |Inactive Partner Contract |Fetch Price Lists |400 |
|1160 |Invalid Market Segment for Partner |Fetch Price Lists |400 |
|1161 |Invalid Currency/Region for Partner |Fetch Price Lists |400 |
|1162 |Price List not found |Fetch Price Lists |404 |
|5122 |Invalid Request |Fetch Price Lists |400 |

For the complete set of error codes, see [Status codes and error handling](../references/error-handling.md).

## Local currency support

Local currency support provides country-specific prices and, where applicable, a change in subscription currency. A country's cutover is the date on which its country-specific pricing and applicable currency take effect.

| Country | Currency after cutover | What changes |
|---|---|---|
| South Korea (`KR`), Türkiye (`TR`), Brazil (`BR`), India (`IN`) | USD | Country-specific prices replace regional USD prices. The currency does not change. |
| Canada (`CA`) | CAD | Country-specific CAD prices apply, with subscriptions moving from USD to CAD as described below. |
| Mexico (`MX`) | MXN | Country-specific MXN prices apply, with subscriptions moving from USD to MXN as described below. |

**Important:**

- Local currency support will be rolled out in phases. The first phase, planned for November 2026, introduces country-specific pricing in USD. The second phase, planned for December 2026, introduces CAD for Canada and MXN for Mexico.
- The December 1, 2026 cutover date applies to South Korea, Türkiye, Brazil, and India only; it is not a CAD or MXN cutover date. Use the applicable country's cutover when planning orders and renewals.

### Request country-specific prices

Use `POST /v3/pricelist` to retrieve country-specific prices. Use `USD` for `KR`, `TR`, `BR`, and `IN`, `CAD` for `CA`, and `MXN` for `MX`. See [Request parameters](#request-parameters) for supported request fields and region and country codes.

Country-specific prices can differ from regional prices for the same offer, currency, and price list month. After cutover, use country-specific pricing, subject to existing 3YC partner price locks.

Each offer's `firstOrderDate` indicates when its country-specific price applies. The following example retrieves a price for `Türkiye`. The month and offer ID are illustrative.

```json
{
    "region": "TR",
    "marketSegment": "COM",
    "priceListType": "STANDARD",
    "currency": "USD",
    "priceListMonth": "202610",
    "filters": {
        "offerId": "65324779CA01A12"
    }
}
```

To retrieve regional pricing, including historical pricing for a 3YC commitment accepted before cutover, specify the applicable regional code in `region`, such as `NA` for North America.

### Quote and place orders at cutover

| Scenario | Pricing behavior |
|---|---|
| New orders, renewals, early renewals, mid-term seat additions, and `SWITCH_PLAN` on or after cutover | Use country-specific pricing and currency after cutover, subject to commitment exceptions. If the currency changes, create a new-currency subscription instead of adding seats to the old-currency subscription. |
| 3YC commitment accepted before cutover | Retain the regional price list from the partner price lock date and the original currency until the commitment ends. |
| 3YC commitment accepted on or after cutover | Lock to the country-specific price list at the commitment acceptance date. |
| Early renewal placed before cutover | Retain regional pricing. |
| Returns and switch reverts | Use the original order price and currency. Existing eligibility rules still apply. |

For order requests, send the applicable `currencyCode` on each line item as described in [Create order](../order-management/create-order.md).

### Track subscriptions when the currency changes

Subscriptions are uniquely identified by SKU and currency. For Canada and Mexico, subscriptions transition from USD to the applicable local currency, CAD and MXN, as follows:

- New orders after cutover create subscriptions in CAD or MXN.
- At the next eligible anniversary renewal, the subscription renews in the new currency. If a subscription in that currency does not exist, a new one is created.
- Auto-renewal is disabled for the old-currency subscription, which remains active until it expires.
- Active 3YC and High Growth Offer, or HGO, commitments remain in the old currency until the commitment ends.
- Scheduled subscriptions migrate with the active subscription.

During the transition, two active subscriptions can exist for the same SKU. When tracking these subscriptions:

- Retain both subscription IDs and their currencyCode values.
- Do not treat the new subscription as a duplicate.
- Do not overwrite the history of the old subscription.

Currency changes also apply to global customers. Renewal consolidation applies only to subscriptions with the same deployment and SKU, not to all subscriptions for that SKU across countries. Continue to use `deploymentId` for deployment-specific orders. See[Manage deployments](../deployment-management/index.md).

Use [Get details of a customer's subscriptions](../subscription-management/get-details-for-customers.md) to track the separate records and [Manage auto-renewals](../renewals/auto-renewals.md#renewals-when-the-country-currency-changes) to calculate the combined renewal quantity.

At the anniversary date, subscriptions for the same deployment and SKU are consolidated into one renewal order in the new currency. Preview Renewal shows the combined renewal quantity. The renewal quantity is calculated as follows:

- For each subscription, use renewalQuantity as the target quantity. If renewalQuantity is null, use currentQuantity.
- Subtract any quantity already early renewed from each subscription's target quantity.
- Add the remaining quantities from the old-currency and new-currency subscriptions to determine the final renewal quantity.
- If part of a subscription has been renewed early, the early-renewed quantity remains in the existing subscription until the next commitment. At the anniversary date, the remaining quantity is migrated to the new-currency subscription.

For example, if the USD subscription has a target of 100 licenses and 40 were already early renewed, and the CAD subscription has a target of 20 with no early renewals, the anniversary renewal is for 80 licenses in CAD:

```json
(100 - 40) + (20 - 0) = 80
```

When the subscriptions are consolidated:

- `renewalQuantity` is combined across both subscriptions. Either subscription can be updated before the anniversary date.
- The `autoRenewable` setting from the new-currency subscription applies.
- `flexDiscountCodes` are carried over.
- The Renewal / MOQ code is carried over.

### Handle old-currency order rejections

After a country changes currency, new orders, order previews, manual renewals, early renewals, and seat additions in the old currency are rejected with `CURRENCY_NOT_ACTIVE_FOR_COUNTRY` (HTTP 422), subject to the commitment exceptions.

For eligible customers:

- Place a new order in the new currency instead of adding licenses to the old-currency subscription.
- Do not attempt to migrate an old subscription by changing the currency in a request that still references that subscription.
- For early renewals, specify the new currency without the old `subscriptionId`.
- Do not reference the old USD subscription, even if the request specifies `CAD` or `MXN`.
- The new-currency subscription runs until the next anniversary date.

See [Early renewal after a currency change](../renewals/manual-renewals.md#early-renewal-after-a-currency-change) for the request rules.

### Preserve commitment pricing and currency

Existing commitments have the following exceptions:

- **3YC accepted before cutover:** Retains the existing price list and currency until the commitment ends. Orders cannot be placed in the new currency during the commitment.
- **3YC early renewal:** Active 3YC customers can continue to early renew in USD, subject to existing eligibility rules.
- **Early currency transition:** To move a customer to the new currency before the 3YC commitment ends, contact support to terminate the 3YC first.
- **HGO commitments:** Remain in the old currency until the commitment ends.
- **HGO subscriptions:** Remain ineligible for `SWITCH_PLAN`.

### Switch plans and process returns

For switch plans, returns, and reverts:

- An eligible `SWITCH_PLAN` across currencies uses the destination currency for the entire transaction.
- Returns retain the original order price and currency.
- Switch reverts retain the original order price and currency.
- Existing product, commitment, return-window, and switch-eligibility restrictions continue to apply.

See [Mid-term upgrades](../mid-term/index.md).
