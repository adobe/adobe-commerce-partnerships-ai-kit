# Order creation scenarios

This topic provides details of the following scenarios related to renewal order creation and management:

- [Preview renewal orders](#preview-renewal-orders)
- [Renewal of order](#renewal-orders)

## Preview renewal orders

The `Create Order` API with `orderType` set to PREVIEW_RENEWAL allows partners to simulate a renewal order before the actual renewal is processed. This helps validate renewal eligibility, pricing, and offer availability in advance.

| Endpoint | Method |
|----------|--------|
| `/v3/customers/<customer-id>/orders`         |   `POST`     |

A few of the benefits of previewing a renewal order include:

- Validates renewal eligibility for existing subscriptions based on auto-renewal preferences.
- Returns the best available offer IDs for renewal, including High Growth Offers.
- Provides accurate pricing details for renewal scenarios, including discounts and proration logic.

### Usage instructions

- No `orderId`, `status`, and `links` in the request.
- In case of no `lineItems` in the request, the response indicates what would be in the RENEWAL order based on the auto-renewal preferences (`autoRenewal.enabled` and `autoRenewal.renewalQuantity`) on the customer's subscriptions.
- In case of `lineItems` in the request, the response indicates the RENEWAL order initiated for the selected line items, supporting both Early Renewal (before anniversary date) and Late Renewal (after anniversary date).
- If the customer does not have any subscriptions with autoRenewal enabled, then an error is returned.
- Returns the best available offer IDs for the renewal order.
- The `eligibleOffers` section lists the High Growth Offers available for the customer. Read more about the [High Growth Offers](../customer-account/high-growth.md).
- The `discountCode` indicates the discount code applicable to the HVD customers migrating from VIP to VIP Marketplace. This parameter does not apply to non-HVD customers.
- The `lineItems []  → status` parameter in the response adheres to standard status codes, for example, 1000 = Active, 1004 = Inactive, and so on. However, its interpretation is specific to the context of Preview Renewal Orders. For example:

    - **1000** indicates that the subscription is either active or scheduled and is expected to renew successfully.
    - **1004** indicates that the subscription is active or scheduled, but the associated product has expired, so the renewal will not proceed.
- Include the query parameter `fetch-price=true` to retrieve pricing details.
- Pricing details are not available in Preview Order and Preview Renewal scenarios for global sales involving multiple currencies.
- `proratedDays` in the response indicates the number of days for which the order will be invoiced. This applies in the case of mid-term purchases.

### Sample request

**Request URL:** `<ENV>/v3/customers/{customer-id}/orders?fetch-price=true`

**Request sample:**

```json
{
  "orderType" : "PREVIEW_RENEWAL"
}
```

### Response

```json
{
  "referenceOrderId": "",
  "externalReferenceId": "",
  "orderId": "",
  "customerId": "1006370655",
  "currencyCode": "USD",
  "orderType": "PREVIEW_RENEWAL",
  "creationDate": "2025-05-02T22:49:54Z",
  "status": "",
  "lineItems": [
    {
      "extLineItemNumber": 1,
      "offerId": "11083117CA03A12",
      "quantity": 10,
      "subscriptionId": "3d0630693446f8bdff9cbd08f4b68bNA",
      "status": "1000",
      "currencyCode": "USD",
      "proratedDays": 365,
      "pricing": {
        "partnerPrice": 350.50,
        "discountedPartnerPrice": 350.50,
        "netPartnerPrice": 350.50,
        "lineItemPartnerPrice": 3505.00
      }
    }
  ],
  "pricingSummary": [
    {
      "totalLineItemPartnerPrice": 3505.00,
      "currencyCode": "USD"
    }
  ],
}
```

### Pricing details in lineitems (lineItems[].pricing)

| Field                       | Description                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| partnerPrice                | Non-prorated full-term unit price for the given offer, including any applicable volume discounts, but before applying flexible discounts and taxes.|
| discountedPartnerPrice     | Unit price after applying discount. \<br /\> |
| netPartnerPrice                 | Prorated unit price after discount. |
| lineItemPartnerPrice      | Prorated price of item after discount and before tax. This is the price partner needs to pay to Adobe for this item.  |

**Note:** The `proratedDays` parameter in the response specifies the number of days for which the order will be invoiced. This parameter appears only when the `fetch-price` parameter is set to `true` in the request. It is relevant for mid-term purchases.

### Pricing Summary (pricingSummary[])

| Field                       | Description                                                                 |
|----------------------------|-----------------------------------------------------------------------------|
| totalLineItemPartnerPrice               | Sum of all line item prices in the order.                 |
| currencyCode                 | Currency used for pricing. This is specified in ISO 4217 currency code. Example, USD and EUR.                                    |

For the complete set of request and response parameter descriptions, refer to [Order resource](../references/resources.md#order-top-level-resource).

#### Sample request and response for HVD customers

**Request:**

```json
{
  "orderType": "PREVIEW_RENEWAL"
}
```

OR

```json
{
  "orderType": "PREVIEW_RENEWAL",
  "lineItems": [
    {
      "extLineItemNumber": 1,
      "offerId": "80004567EA01A12",
      "discountCode": "HVD_L18_PRE",
      "subscriptionId": "e0b170437c4e96ac5428364f674dffNA"
    }
  ]
}
```

**Response:**

**Note:** Pricing details are included in the response as the query parameter `fetch-price` was set to `true` in the request URL.

```json
{
  "referenceOrderId": "",
  "orderType": "PREVIEW_RENEWAL",
  "externalReferenceId": "759",
  "orderId": "",
  "customerId": "9876543210",
  "currencyCode": "USD",
  "creationDate": "2019-05-02T22:49:54Z",
  "status": "",
  "lineItems": [
    {
      "extLineItemNumber": 4,
      "offerId": "80004567EA01A12",
      "quantity": 1,
      "subscriptionId": " e0b170437c4e96ac5428364f674dffNA",
      "discountCode": "HVD_L18_PRE",
      "status": "1000",
      "currencyCode": "USD",
      "deploymentId": "12345",
      "proratedDays": 365,
      "pricing": {
        "partnerPrice": 299.99,
        "discountedPartnerPrice": 299.99,
        "proratedPartnerPrice": 299.99,
        "lineItemPartnerPrice": 299.99
      }
    },
    {
      "extLineItemNumber": 1,
      "offerId": "65322447CA01A12",
      "quantity": 25,
      "subscriptionId": "4392d721a543929afb871a4c140435NA",
      "discountCode": "HVD_L18_PRE",
      "status": "1004",
      "currencyCode": "USD",
      "deploymentId": "12345",
      "proratedDays": 365,
      "pricing": {
        "partnerPrice": 299.99,
        "discountedPartnerPrice": 299.99,
        "proratedPartnerPrice": 299.99,
        "lineItemPartnerPrice": 7499.75
      }
    }
  ],
  "pricingSummary": [
    {
      "totalLineItemPartnerPrice": 7799.74,
      "currencyCode": "USD"
    }
  ],
  "eligibleOffers": [
    {
      "offerId": "65324918CA14X12",
      "renewalCode": "MOQ_100",
      "eligibilityCriteria": {
        "minQuantity": 100,
        "additionalCriteria": [
          "THREE_YEAR_COMMIT"
        ],
        "deploymentId": "1450043516"
      }
    },
    {
      "offerId": "65324918CA14Y12",
      "renewalCode": "MOQ_250",
      "eligibilityCriteria": {
        "minQuantity": 250,
        "additionalCriteria": [
          "THREE_YEAR_COMMIT"
        ],
        "deploymentId": "1450043516"
      }
    },
    {
      "offerId": "65324918CA14Z12",
      "renewalCode": "MOQ_500",
      "eligibilityCriteria": {
        "minQuantity": 500,
        "additionalCriteria": [
          "THREE_YEAR_COMMIT"
        ]
      }
    }
  ]
}
```

## Renewal orders

**Notes:**

- RENEWAL order type is used for late renewals, which are initiated after the anniversary date.
- Subscription ID is required in the request line item.
- The license quantities must be less than or equal to the customer's current subscription quantities.
- Order ID is created by this service and returned synchronously.
- Partner Marketplaces are expected to check the status of the order for success.
- You can select the expired subscriptions for manual renewal by using the [Get All Subscriptions for a Customer](../subscription-management/get-details-for-customers.md) API. Subscriptions that can be selected for manual renewal are indicated by the `allowedActions`": `["MANUAL_RENEWAL"]` parameter of the Get All Subscriptions of a Customer API response.

### Sample request

```json
{
  "orderType": "RENEWAL",
  "externalReferenceId": "759",
  "currencyCode": "USD",
  "lineItems": [
    {
      "extLineItemNumber": 1,
      "offerId": "80004567EA01A12",
      "subscriptionId": " e0b170437c4e96ac5428364f674dffNA ",
      "quantity": 1
    }
  ]
}
```

### Sample response

```json
{
  "referenceOrderId": "",
  "orderType": "RENEWAL",
  "externalReferenceId": "759",
  "customerId": "9876543210",
  "orderId": "5120008001",
  "currencyCode": "USD",
  "creationDate": "2019-05-02T22:49:54Z",
  "status": "1002",
  "lineItems": [
    {
      "extLineItemNumber": 1,
      "offerId": "80004567EA01A12",
      "quantity": 1,
      "status": "1002",
      "subscriptionId": " e0b170437c4e96ac5428364f674dffNA "
    }
  ],
  "links": { ... }
}
```