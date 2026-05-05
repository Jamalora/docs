# Livra Integration Guide (Production)

This document describes how to call Livra integration endpoints from your app.

## Contents

- [Shared authentication headers](#shared-authentication-headers)
- [Create Merchant](#create-merchant)
- [Create Order](#create-order)
- [Update Order](#update-order)
- [Change Request](#change-request)
- [Order status webhooks](#order-status-webhooks)

[← Back to documentation index](../README.md)

## Shared authentication headers

Use these headers for all Livra integration endpoints:

- `Content-Type: application/json`
- `x-api-key: <apiKey>`
- `x-signature: <hexHmac>`

`x-signature` must be `HMAC-SHA256(rawRequestBody, apiSecret)` encoded as lowercase hex (optionally prefixed with `sha256=`).

## Create Merchant

- **URL:** `https://livra.mofavo.com/create_merchant`
- **Method:** `POST`

### Request body

```json
{
  "merchant": {
    "name": "Example Merchant LLC",
    "state": "Dubai",
    "city": "Dubai",
    "street": "Example Street 1",
    "phoneNumber": "+971500000000",
    "zipcode": "00000",
    "TRN": "100000000000003",
    "CIN": 12345678
  },
  "sender": {
    "name": "Example Sender LLC",
    "state": "Dubai",
    "city": "Dubai",
    "street": "Business Bay",
    "phoneNumber": "+971511111111"
  },
  "contract": {
    "deliveryPartnerId": 10,
    "deliveryFee": 12.5,
    "exchangeFee": 4.25,
    "cancellationFee": 3.0
  }
}
```

### Rules

- `merchant.CIN` must be an 8-digit integer.
- All merchant/sender string fields must be non-empty.
- `contract.deliveryPartnerId` must be a positive integer.
- Contract fees must be non-negative numbers.

### Success

- **201**: `{ "merchantId": <number> }`

### Errors

- **400** invalid payload
- **401** missing/invalid auth headers/signature
- **500** internal error

## Create Order

- **URL:** `https://livra.mofavo.com/create_order`
- **Method:** `POST`

### Request body

```json
{
  "products": [
    { "name": "string", "quantity": 1, "price": 12.5 },
    { "name": "string", "quantity": 2 }
  ],
  "productsToRetrieve": [
    { "name": "string", "quantity": 1 },
    { "name": "", "quantity": 0 }
  ],
  "merchantId": 1,
  "deliveryPartnerId": 1,
  "primaryName": "string",
  "primaryPhone": "string",
  "primaryPhone2": "",
  "primaryStreet": "",
  "primaryZone": "",
  "primaryCity": "string",
  "primaryState": "string",
  "primaryZipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false,
  "callback_link": "https://your-app.example.com/livra/webhook"
}
```

`callback_link` is optional. Omit it or leave off to disable webhooks for that order.

### Rules

- `products` is required and must be a non-empty array.
- Each `products[]` item needs non-empty `name`, positive integer `quantity`, optional non-negative `price`.
- `merchantId` and `deliveryPartnerId` must be positive integers.
- `primaryName`, `primaryPhone`, `primaryCity`, `primaryState` must be non-empty.
- `amount` must be non-negative.
- `allowOpen`, `isExchange`, `isFragile` must be booleans.
- If `isExchange=true`, `productsToRetrieve` is expected. Invalid/missing entries still proceed with fallback name `"unknown"`.
- **`callback_link` (optional):** when present, Livra sends a signed `POST` to this URL whenever the order undergoes a meaningful status change (see [Order status webhooks](#order-status-webhooks)). Must be a valid absolute HTTP or HTTPS URL that accepts JSON `POST`s from Livra’s infrastructure.

### Success

- **201**: `{ "orderId": <number> }`

### Errors

- **400** one of:
  - `merchant_not_found`
  - `no_active_contract`
  - `sender_not_found`
  - validation errors
- **401** missing/invalid auth headers/signature
- **500** internal error

## Update Order

- **URL:** `https://livra.mofavo.com/update_order`
- **Method:** `POST`

### Request body

Patch-style payload. Only `orderId` is required; all other fields are optional.

```json
{
  "orderId": 1234,
  "products": [{ "name": "string", "quantity": 1 }],
  "productsToRetrieve": [{ "name": "string", "quantity": 1 }],
  "primaryName": "string",
  "primaryPhone": "string",
  "primaryPhone2": "",
  "primaryStreet": "",
  "primaryZone": "",
  "primaryCity": "string",
  "primaryState": "string",
  "primaryZipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false
}
```

### Constraints

- `orderId` must exist.
- Existing order status must still be `readyForPickUp`; otherwise update is rejected.
- Missing fields keep their current DB value.
- If a provided field has the same value, it is ignored (no rewrite).

### Success

- **200**: `{ "orderId": <number> }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_update_not_permitted`
  - plus all create-order 400 errors
- **401** missing/invalid auth headers/signature
- **500** internal error

## Change Request

- **URL:** `https://livra.mofavo.com/change_request`
- **Method:** `POST`
- **Auth:** same `x-api-key` + `x-signature` flow as other endpoints

### Request body

```json
{
  "orderId": 1234,
  "changes": [
    {
      "type": "PHONE_CHANGE",
      "oldValue": "+971500000000",
      "newValue": "+971511111111"
    }
  ],
  "comment": "Customer requested phone correction",
  "makeRegular": false
}
```

### Rules

- `orderId` must be a positive integer.
- `changes` must be a non-empty array.
- Each change requires `type` and must be one of:
  - `PHONE_CHANGE`
  - `PHONE2_CHANGE`
  - `AMOUNT_CHANGE`
  - `ADDRESS_CHANGE`
  - `ALLOW_OPEN_CHANGE`
  - `DELIVERY_DATE_CHANGE`
- `oldValue` and `newValue` are optional and can be any JSON value.
- `comment` is optional string.
- `makeRegular` is optional boolean.
- Order must currently be in `inDepot` or `inTransit` status.
- If the order is exchange-linked and `deliveryDate` is set (exchange already happened), request creation is blocked.

### Success

- **201**: `{ "ok": true }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_status_not_eligible_for_change_request`
  - `exchange_already_completed_change_request_not_allowed`
  - `no_changes_provided`
  - validation errors
- **401** missing/invalid auth headers/signature
- **500** internal error

## Order status webhooks

When you include **`callback_link`** on [create order](#create-order), Livra calls that URL with an outbound webhook on every meaningful change to the order.

### Request format

```
POST <your-callback-url>
Content-Type: application/json
X-Webhook-ID: <delivery-uuid>
X-Webhook-Signature: <hmac-hex>
```

### Payload

```json
{
  "ok": true,
  "orders": [
    {
      "id": 1234,
      "deliveryStatus": "pending",
      "orderStatus": "inTransitToCustomer"
    }
  ]
}
```

### What `deliveryStatus` and `orderStatus` mean

Together they answer two different questions: whether the **delivery outcome** for this order is still open, completed, or void, and **where the shipment is** in its journey right now (depot, on the road, and which direction when in transit).

#### `deliveryStatus`

High-level outcome from a delivery perspective:

| Value | Meaning |
| --- | --- |
| `pending` | The order is still in play — not finally delivered to the end recipient in the sense we track here, and not written off as a customer decline / return-to-merchant flow. |
| `delivered` | That outcome is satisfied in our model (for example, the exchange with the customer has already happened and the parcel leg you care about is treated as delivered, even if another leg — such as back to the merchant — is still moving). |
| `cancelled` | The outcome is no longer a normal forward delivery (for example, the customer refused delivery and the parcel is being handled as a return or stop). |

#### `orderStatus`

Where the order sits in the **operational pipeline** — in a depot, on the road, and which direction it is heading when in transit (toward the customer vs toward the merchant):

| Value | Meaning |
| --- | --- |
| `readyForPickUp` | The order has been created and is waiting to be collected by a courier. |
| `inDepot` | The parcel is held at a depot (before first-mile pickup, between legs, or after a customer decline). |
| `inTransitToCustomer` | The parcel is on the road heading toward the customer. |
| `inTransitToMerchant` | The parcel is on the road heading back to the merchant (return or post-exchange leg). |
| `delivered` | The order has been delivered to the customer. |
| `returned` | The order has been returned to the merchant. |
| `exchange-returned` | The exchange was declined by the customer, and the parcel sent for the exchange has returned to the merchant. |
| `exchange-completed` | The exchange happened with the customer, and the parcel collected from the customer has returned to the merchant. |
| `cancelled` | The order has been voided (e.g. a dummy order created as part of an exchange flow). |
| _(other values)_ | New or internal statuses — treat unknown values gracefully. |

#### How the two combine

Examples of valid combinations:

- **Exchange already completed with the customer, parcel now traveling back to the merchant:** `deliveryStatus` can be `delivered` while `orderStatus` reflects the current leg, e.g. `inTransitToMerchant`.
- **Parcel waiting in a depot before final delivery to the customer:** `deliveryStatus` `pending`, `orderStatus` `inDepot`.
- **Customer declined delivery and the parcel is held in a depot:** `deliveryStatus` `cancelled`, `orderStatus` may still be `inDepot` — location/stage in the pipeline can differ even when the delivery outcome is cancelled.

### Verifying signatures

Every request includes an `X-Webhook-Signature` header containing an **HMAC-SHA256** of the **raw request body**, hex-encoded, using your **Livra API secret** (the same secret you use to sign requests to Livra).

Always verify this header before processing the payload.

#### Examples

**Node.js**

```js
const crypto = require('crypto');

function verifySignature(secret, rawBody, signature) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}

// Express example
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-webhook-signature'];
  if (!verifySignature(process.env.WEBHOOK_SECRET, req.body, sig)) {
    return res.status(401).send('Invalid signature');
  }
  const event = JSON.parse(req.body);
  // process event...
  res.sendStatus(200);
});
```

**Python**

```python
import hmac, hashlib

def verify_signature(secret: str, raw_body: bytes, signature: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Go**

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func verifySignature(secret, signature string, body []byte) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}
```

> **Important:** always read the raw request body for signature verification. Parsing the JSON first and re-serialising it may produce a different byte sequence and cause verification to fail.

### Responding to events

Reply with any **2xx status code** to acknowledge successful delivery. The response body is ignored.

If your endpoint returns a non-2xx status or does not respond within **10 seconds**, the delivery is retried automatically.

### Retry schedule

Failed deliveries are retried with exponential backoff:

| Attempt | Delay before retry |
| --- | --- |
| 1 | 30 seconds |
| 2 | 5 minutes |
| 3 | 30 minutes |
| 4 | 2 hours |
| 5 | 8 hours |

After 5 failed attempts the delivery is marked permanently failed and no further retries are made. The platform team can manually re-queue a delivery on request.

### Identifying deliveries

Each delivery has a unique UUID sent in the `X-Webhook-ID` header. Use this to deduplicate events if your endpoint receives the same delivery more than once.

### Testing your endpoint

A quick way to simulate a webhook delivery locally:

```bash
SECRET="your-secret"
BODY='{"ok":true,"orders":[{"id":1234,"deliveryStatus":"pending","orderStatus":"inTransitToCustomer"}]}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://your-endpoint.example.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Webhook-ID: test-$(uuidgen)" \
  -H "X-Webhook-Signature: $SIG" \
  -d "$BODY"
```
