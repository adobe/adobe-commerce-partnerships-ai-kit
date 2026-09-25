# Partial Return

Partial quantity returns are supported for eligible `NEW` and `RENEWAL` orders within the return window. You can return part of a line item, and you can submit multiple partial returns against the same line item. Each request is validated against the line item’s current `remainingQuantity` value.

For example, if a line item starts with a quantity of 70 and you return 10, the `remainingQuantity` becomes 60. If you later return 5 more, the `remainingQuantity` becomes 55. Returning the full original quantity in a single request is still supported.

**Sample request**

```json
{
  "referenceOrderId": "0123456789",
  "orderType": "RETURN",
  "externalReferenceId": "759",
  "currencyCode": "USD",
  "lineItems": [
    {
      "extLineItemNumber": 4,
      "offerId": "80004567EA01A12",
      "quantity": 10,
      "currencyCode": "USD",
      "deploymentId": "12345"
    }
  ]
}
```

The return response is the same as a standard return. The credit is calculated using the pricing from the original order. To confirm the updated returnable quantity, call [Get order details](get-order.md) for the *original* order:

**Request:** `GET /v3/customers/9876543210/orders/0123456789`

**Response:**

```json
{
  "orderId": "0123456789",
  "orderType": "RENEWAL",
  "status": "1000",
  "lineItems": [
    {
      "extLineItemNumber": 4,
      "offerId": "80004567EA01A12",
      "quantity": 70,
      "subscriptionId": "a4f1c2d0-0001",
      "status": "1000",
      "remainingQuantity": 60
    }
  ],
  "links": {}
}
```

The `remainingQuantity` drops from 70 to 60. Each request is checked against the current value. Licenses covered by the returned quantity are deprovisioned immediately once the return is accepted.

#### **Validation and errors**

A RETURN request against a NEW or RENEWAL line item is rejected with HTTP 422, if it does not meet the required conditions. The original order remains unchanged.

| Scenario | Condition | Error code |
|---|---|---|
| Minimum Order Quantity offer | Offer is flagged as MOQ | **PARTIAL_CANCELLATION_NOT_ALLOWED_FOR_SCP_OR_CONSUMABLE** |
| Switch Plan order | Referenced order is a **SWITCH** order. Use **REVERT_SWITCH** instead. | **SWITCH_ORDER_CANCELLATION_NOT_ALLOWED** |
| Three-Year Commit minimum | Return would reduce the committed product-family quantity below the minimum commit quantity | **THREE_YEAR_COMMIT** |
