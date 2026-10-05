# Livra Integration Guide (Production)

This document describes how to call Livra integration endpoints from your app.

## Contents

- [Shared authentication headers](#shared-authentication-headers)
- [Create Merchant](#create-merchant)
- [Create Order](#create-order)
- [Update Order](#update-order)
- [Order Hash](#order-hash)
- [Change Request](#change-request)
- [Order status webhooks](#order-status-webhooks)
- [Accept In Depot (delivery partners)](partner-accept-in-depot.md)
- [Transfers Between Depots (delivery partners)](partner-transfers.md)
- [Get Order (delivery partners)](partner-get-order.md)

[← Back to documentation index](../README.md)

> **Are you a delivery partner working with your own depots?**
> Those endpoints have their own pages:
> [Accept In Depot](partner-accept-in-depot.md) to check a parcel in when a truck
> arrives, [Transfers Between Depots](partner-transfers.md) to send parcels on to
> the next depot, and [Get Order](partner-get-order.md) to read one of your orders back
> (status, dates, and the slip hash for printing).

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

- **201**: `{ "orderId": <number>, "hash": "<string>" }`
  - `hash` is the order-slip QR hash. A slip's QR code must encode `"<orderId>#<hash>"` to be accepted by depot scanners. The hash covers the order's content (recipient, products, amount, …), so after any order update the previous hash is stale — always use the latest one returned.

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

One endpoint, two actions on an order that is still waiting for its pickup (`readyForPickUp`):

| Action | Body | What it does |
|---|---|---|
| **Update** | no `action` | Patch-style edit of the order's fields. |
| **Cancel** | `"action": "cancel"` | Cancels the order before the driver collects it. |

If your API key is bound to a delivery partner, both actions reach only that partner's orders; any other order is answered with `order_not_found`, the same as one that doesn't exist.

Every response carries an `x-request-id` header. Quote it when you report a problem: it finds the one log line for your request.

### Update: request body

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

### Update: constraints

- `orderId` must exist.
- Existing order status must still be `readyForPickUp`; otherwise update is rejected.
- Missing fields keep their current DB value.
- If a provided field has the same value, it is ignored (no rewrite).

### Update: success

- **200**: `{ "orderId": <number>, "hash": "<string>" }`
  - `hash` is the recomputed order-slip QR hash after the update (see Create Order). Any slip printed with an older hash must be reprinted.

### Update: errors

- **400** one of:
  - `order_not_found`
  - `order_update_not_permitted`
  - plus all create-order 400 errors
- **401** missing/invalid auth headers/signature
- **500** internal error

### Cancel: request body

Use it when the merchant cancels an order **before the driver has collected it**.

**Which orders can be cancelled:**
- Orders created through these APIs: Create Order, or External Create Order. Orders made in the merchant app are refused: their stock goes back to the merchant only through the app's own cancel.
- Orders still waiting for the pickup (`readyForPickUp`), or whose pickup was declined (`pickUp-declined`).
- Orders that haven't been handed over to another delivery partner (outsourced).
- The API key must be a delivery partner's key: Livra's. Other keys get a **403**.

```json
{
  "orderId": 1234,
  "action": "cancel",
  "reason": "Customer changed their mind",
  "comment": "Called the customer on 05/10"
}
```

| Field | Required | Rules |
|---|---|---|
| `orderId` | yes | Positive integer. |
| `action` | yes | Exactly `"cancel"`. |
| `reason` | no | String, up to 255 characters. The cancellation reason, shown with the order. |
| `comment` | no | String, up to 1000 characters. Free text, shown with the order. |

- A cancel takes **only** these four fields. Any other field (e.g. `amount`) is a **400**: cancel and edit are never mixed in one request.
- `reason` and `comment` may be left out, sent as `null`, or sent blank: all three mean "not given". Longer text is cut to the limit.

### Cancel: what happens

- The order becomes **`cancelled`**, and its pending pickup is called off, so no driver is sent for it.
- A cancel is **not a return**. A returned order is one whose delivery was attempted, failed, and came back to the merchant (`orderStatus: "returned"`). A cancelled order never had a delivery attempt.
- A cancel before pickup is **free**: no cancellation fee is charged.
- The order's history shows an **order cancelled** event with the reason and comment.
- The Status API reports it as `orderStatus: "cancelled"`, `deliveryStatus: "cancelled"`.
- A cancelled order can't be edited or un-cancelled through the API.

### Cancel: success

- **200**:

  ```json
  {
    "ok": true,
    "orderId": 1234,
    "status": "cancelled",
    "cancelledAt": "2026-10-05T09:12:44.120Z",
    "alreadyCancelled": false
  }
  ```

- **Safe to retry.** Cancelling an order that is already cancelled is also a **200**, with `alreadyCancelled: true` and the original `cancelledAt` (`null` for an order cancelled before the cancellation date was recorded). Nothing changes. After a timeout or a network error, just send the cancel again.

### Cancel: errors

| Status | `error` | Meaning | What to do |
|---|---|---|---|
| **400** | validation message | Missing/invalid `orderId`, an `action` other than `"cancel"`, a non-string `reason`/`comment`, or extra fields. | Fix the request. |
| **400** | `order_not_found` | No such order, or it isn't yours. | Check the `orderId`. |
| **409** | `order_already_picked_up` | The driver has already collected the parcel. The body also has the order's current `orderStatus` (e.g. `"inTransit"`). | It can't be cancelled here any more. To change an order in the network, use Change Request. |
| **409** | `order_not_cancellable` | Not collected, but in a status this API can't cancel (e.g. `"readyForPackaging"`). The body has `orderStatus`. | Cancel it in the merchant app. |
| **409** | `order_managed_in_merchant_app` | The order was made in the merchant app, not through these APIs. | Cancel it in the merchant app. |
| **409** | `order_outsourced` | The order has been handed over to another delivery partner. | Contact operations. |
| **409** | `order_busy` | The order was being changed at the same moment (e.g. the driver was scanning it). | Retry once after a second. The answer will then be `200` or `order_already_picked_up`. |
| **401** | `Missing authentication headers` / `Invalid api key` / `Invalid signature` | Authentication failed. | Check `x-api-key` and how you sign the body. |
| **403** | `partner_scope_not_configured` | This API key isn't a delivery partner's key, so it can't cancel. | Use the delivery partner's key (ask `ops@mofavo.com`). |
| **500** | `internal_error` | Our error. | Retry later; send us the `x-request-id` if it persists. |

## Order Hash

Use this to fetch the current **order-slip QR hash** for an order — for example right before printing (or reprinting) a delivery slip. The QR must encode `"<orderId>#<hash>"`. The hash covers the order's content, so it changes whenever the order changes; a slip carrying an outdated hash is rejected at scan time.

Create Order and Update Order already return the same `hash` in their responses; this endpoint is for when you need it again later for an existing order.

- **URL:** `https://livra.mofavo.com/order_hash`
- **Method:** `POST`

### Request body

```json
{
  "orderId": 1234
}
```

### Rules

- `orderId` is required and must be a positive integer.

### Success

- **200**: `{ "orderId": <number>, "hash": "<string>" }`

### Errors

- **400** `order_not_found` or validation errors
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
      "oldValue": "50000000",
      "newValue": "51111111"
    },
    {
      "type": "AMOUNT_CHANGE",
      "oldValue": 110,
      "newValue": 125.5
    }
  ],
  "comment": "Customer requested phone and amount correction",
  "makeRegular": false
}
```

### Rules

- `orderId` must be a positive integer.
- `changes` must be a non-empty array. One request can carry one change or several of
  different types (e.g. a phone and an amount together, as in the example above).
- Each change requires `type`, one of the types below, and a `newValue` of the type shown.
  Values are checked strictly: `"79"` is not an amount, and `"true"` is not a boolean.

  | `type` | `newValue` | Example |
  |---|---|---|
  | `AMOUNT_CHANGE` | number, `>= 0` | `79`, `125.5` |
  | `ALLOW_OPEN_CHANGE` | boolean | `true` |
  | `PHONE_CHANGE` | string: optional `+`, then 8 to 15 digits. Spaces and dashes are allowed and removed before saving (`"51 111 111"` and `"51-111-111"` are saved as `"51111111"`). Dots, brackets, tabs and other characters are rejected. | `"51111111"` |
  | `PHONE2_CHANGE` | same as `PHONE_CHANGE`, or `""` to remove the second phone. `null` is rejected: send `""` instead. | `"51111111"`, `""` |
  | `DELIVERY_DATE_CHANGE` | the planned delivery day, as a `"YYYY-MM-DD"` date that exists, today or later (Tunisia time). A past date is rejected. `null` clears the planned date. | `"2026-10-02"`, `null` |
  | `ADDRESS_CHANGE` | an object with any of `street`, `zone`, `city`, `state`, `zipcode`. Each value is a string or `null`; strings are trimmed before saving and may be at most 255 characters after trimming. A value of only spaces is rejected. At least one field must be non-empty. Any other key is rejected. | `{ "street": "12 Rue X", "zipcode": "2036" }` |

- In `ADDRESS_CHANGE`, a `null` or `""` field is **ignored** when the request is approved, not
  cleared: the order keeps its current value. Send only the fields that change.
- `oldValue` is optional, is not checked, and is only shown to whoever reviews the request.
  Use the same type as `newValue` so it displays well.
- Each `type` may appear only once per request. To change several fields, put all of them in
  the **same** request rather than sending them one by one: a new request for the order replaces
  its pending one, so sending them separately would keep only the last.
- `comment` is optional string.
- `makeRegular` is optional boolean.
- If your API key is linked to a delivery partner, the order must belong to that partner. Any
  other order is answered with `order_not_found`, exactly as if it didn't exist.
- Order must currently be in `inDepot` or `inTransit` status.
- If the order is exchange-linked and `deliveryDate` is set (exchange already happened), request creation is blocked.

### Success

- **201**: `{ "ok": true }`

### Errors

- **400** one of:
  - `order_not_found` (also for an order that belongs to another delivery partner)
  - `order_status_not_eligible_for_change_request`
  - `exchange_already_completed_change_request_not_allowed`
  - `no_changes_provided`
  - `body is not valid JSON`
  - validation errors. Each names the change and says what to send, for example:
    - `changes[0].newValue must be a number, not a string: send 79, not "79"`
    - `changes[0].newValue must be true or false without quotes: send true, not "true"`
    - `changes[0].newValue must be a phone number like "51111111"`
    - `changes[0].newValue is in the past (today is 2026-10-02)`
    - `changes[0].newValue.primaryStreet is not an address field (use street, zone, city, state, zipcode)`
    - `changes[1] repeats AMOUNT_CHANGE from changes[0]`
- **401** missing/invalid auth headers/signature
- **500** internal error

## Order status webhooks

When you include **`callback_link`** on [create order](#create-order), Livra calls that URL with an outbound webhook on every meaningful change to the order.

Every request carries two headers that identify exactly what you are receiving:

```
X-Webhook-Type: advanced
X-Webhook-Version: 1
```

Use `X-Webhook-Version` to guard your parsing logic against future changes.

---

### Advanced webhook

**Current version: 1**

The raw event payload as recorded by the platform. Each delivery represents one discrete change, with an explicit event name, a full snapshot of the current field values, and their previous values for comparison. Driver outcomes are separate `driver.*` events rather than a comment on an order event.

Full documentation: [Livra Webhooks — Advanced](webhooks-preview.md)

---

### Shared delivery mechanics

The following applies to all Livra webhooks.

#### Verifying signatures

Every request includes an `X-Webhook-Signature` header containing an **HMAC-SHA256** of the **raw request body**, hex-encoded, using your **Livra API secret** (the same secret you use to sign requests to Livra).

Always verify this header before processing the payload.

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

#### Responding to events

Reply with any **2xx status code** to acknowledge successful delivery. The response body is ignored.

If your endpoint returns a non-2xx status or does not respond within **10 seconds**, the delivery is retried automatically.

#### Retry schedule

| Attempt | Delay before retry |
| --- | --- |
| 1 | 30 seconds |
| 2 | 5 minutes |
| 3 | 30 minutes |
| 4 | 2 hours |
| 5 | 8 hours |

After 5 failed attempts the delivery is marked permanently failed and no further retries are made. The platform team can manually re-queue a delivery on request.

#### Identifying deliveries

Each delivery has a unique UUID in the `X-Webhook-ID` header. Use it to deduplicate events if your endpoint receives the same delivery more than once.
