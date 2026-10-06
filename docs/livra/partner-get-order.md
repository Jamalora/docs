# Get Order — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_get_order`, which reads back **one of your own orders** as it is stored
right now.

You get the order's content (recipient, products, amount, …), where it is in its journey
(`status`, `finalDestination`, `deliveryDate`, `returnedDate`), its exchange link, and the
current slip `hash` for printing the QR code.

It only reads. Calling it never changes the order, so you can call it as often as you need.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The request](#2-the-request)
- [3. How to build `x-signature`](#3-how-to-build-x-signature)
- [4. Copy-paste examples](#4-copy-paste-examples)
- [5. The success response](#5-the-success-response)
  - [Every field, explained](#every-field-explained)
  - [Exchange orders](#exchange-orders)
  - [Printing the slip — `hash`](#printing-the-slip--hash)
- [6. Every error, and what to do about it](#6-every-error-and-what-to-do-about-it)
- [7. Checklist before you go live](#7-checklist-before-you-go-live)
- [8. Common mistakes](#8-common-mistakes)

---

## 1. What you need before you start

Two secrets from us. Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent** — used to compute `x-signature` |

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 3](#3-how-to-build-x-signature).

These are the same key and secret you use for [Accept In Depot](partner-accept-in-depot.md)
and [Transfers Between Depots](partner-transfers.md). Your key must be enabled for partner
endpoints; if it isn't, every call returns `403 partner_scope_not_configured`.

> **You do not send your partner id.** We read it from your API key. That is deliberate —
> it means you can only ever read **your own** orders, and nobody can read yours.

## 2. The request

- **URL:** `https://livra.mofavo.com/partner_get_order`
- **Method:** `POST`

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 3>
```

That's all. **There is no `Authorization` header and no bearer token on this endpoint.**

### Body

```json
{
  "orderId": 1234
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderId` | number | Required. Positive whole number, at most `2147483647`. |

- The id must be a **number**, not a string. `"1234"` is wrong. `1234` is right.
- Extra fields you add are ignored, they don't break anything.
- One order per call.

## 3. How to build `x-signature`

`x-signature` is an HMAC-SHA256 of the **request body**, keyed with your **API secret**,
written as lowercase hex.

```
x-signature = HMAC_SHA256( raw request body , apiSecret )  →  lowercase hex
```

`sha256=<hex>` is also accepted, if your HTTP library adds that prefix.

**The one rule that trips everyone up:** sign the **exact bytes you send**.

Build the JSON string **once**, sign *that string*, and send *that same string* as the
body. Do **not** build an object, sign a serialization of it, and then let your HTTP
library serialize the object again — the two strings can differ by one space and the
signature fails with `401 Invalid signature`.

✅ Right:

```js
const body = JSON.stringify({ orderId: 1234 }); // build ONCE
const signature = hmac(body);                   // sign the string
fetch(url, { body });                           // send the SAME string
```

❌ Wrong:

```js
const payload = { orderId: 1234 };
const signature = hmac(JSON.stringify(payload));  // signs one string…
axios.post(url, payload);                         // …axios serializes a different one
```

## 4. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"orderId":1234}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://livra.mofavo.com/partner_get_order \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

### Node.js

```js
import crypto from "node:crypto";

const API_KEY = process.env.API_KEY;
const API_SECRET = process.env.API_SECRET;

export async function getOrder(orderId) {
  const body = JSON.stringify({ orderId });
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://livra.mofavo.com/partner_get_order", {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-api-key": API_KEY,
      "x-signature": signature,
    },
    body, // the SAME string that was signed
  });

  const data = await res.json();
  return { status: res.status, requestId: res.headers.get("x-request-id"), data };
}
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"

def get_order(order_id: int):
    body = json.dumps({"orderId": order_id}, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://livra.mofavo.com/partner_get_order",
        headers={
            "Content-Type": "application/json",
            "x-api-key": API_KEY,
            "x-signature": signature,
        },
        data=body,  # data=, NOT json= — json= would re-serialize and break the signature
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()
```

### PHP

```php
<?php
$apiKey    = getenv('API_KEY');
$apiSecret = getenv('API_SECRET');

$body = json_encode(['orderId' => 1234]);
$signature = hash_hmac('sha256', $body, $apiSecret);

$ch = curl_init('https://livra.mofavo.com/partner_get_order');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER => [
        'Content-Type: application/json',
        'x-api-key: ' . $apiKey,
        'x-signature: ' . $signature,
    ],
    CURLOPT_POSTFIELDS => $body,
]);
$response = curl_exec($ch);
$status   = curl_getinfo($ch, CURLINFO_HTTP_CODE);
curl_close($ch);
```

## 5. The success response

**HTTP 200**

```json
{
  "products": [
    { "name": "T-shirt", "quantity": 1, "price": 40 },
    { "name": "Socks", "quantity": 2 }
  ],
  "productsToRetrieve": [],
  "merchantId": 5,
  "deliveryPartnerId": 7,
  "status": "inDepot",
  "finalDestination": "primaryRecipient",
  "primaryName": "Ali Ben Salah",
  "primaryPhone": "51111111",
  "primaryPhone2": "",
  "primaryStreet": "12 Rue de Marseille",
  "primaryZone": "",
  "primaryCity": "Tunis",
  "primaryState": "Tunis",
  "primaryZipcode": "1000",
  "oldOrderId": null,
  "deliveryInstructions": "",
  "deliveryDate": null,
  "returnedDate": null,
  "amount": 79.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false,
  "hash": "Xk3pQw"
}
```

**Every field is always present.** A value that isn't set yet is `null` (or `""` for the
optional text fields) — it is never left out. The one exception is `products[].price`,
which is left out when the product has no price.

Every response — success or failure — also carries an **`x-request-id`** header.
**Log it.** If you ever open a support ticket, that id is the first thing we ask for.

### Every field, explained

**The order's content** — the same fields you'd send to create an order:

| Field | Type | Meaning |
| --- | --- | --- |
| `products` | array | What is delivered to the customer: `name`, `quantity`, and `price` when there is one. |
| `productsToRetrieve` | array | What is collected from the customer on an exchange: `name`, `quantity`. Always `[]` for a normal order. |
| `merchantId` | number | The merchant the order belongs to. |
| `deliveryPartnerId` | number | Your partner id. Always yours — see [section 6](#6-every-error-and-what-to-do-about-it). |
| `primaryName` | string | Recipient's name. |
| `primaryPhone` | string | Recipient's phone. |
| `primaryPhone2` | string | Second phone, or `""`. |
| `primaryStreet`, `primaryZone` | string | Address lines, or `""`. |
| `primaryCity`, `primaryState` | string | City and governorate. |
| `primaryZipcode` | string | Postal code, or `""`. |
| `deliveryInstructions` | string | Free-text instructions, or `""`. |
| `amount` | number | Cash to collect on delivery. |
| `allowOpen` | boolean | Whether the customer may open the parcel before paying. |
| `isFragile` | boolean | Handle with care. |

**Where the order is now:**

| Field | Type | Meaning |
| --- | --- | --- |
| `status` | string | The order's current status, e.g. `readyForPickUp`, `inDepot`, `inTransit`. Treat it as text and don't fail on a value you don't recognise — new ones can appear. |
| `finalDestination` | string \| `null` | Where the parcel is heading: `primaryRecipient` (the customer) or `merchant` (on its way back). `null` until set. |
| `deliveryDate` | string \| `null` | When the parcel was delivered to the customer. `null` until then. |
| `returnedDate` | string \| `null` | When the parcel's return to the merchant was completed. `null` until then. |

Both dates are ISO-8601 in UTC, e.g. `"2026-10-02T09:30:00.000Z"`. Convert to local time
(Tunisia is UTC+1) before showing them to an agent.

**Exchange and slip:**

| Field | Type | Meaning |
| --- | --- | --- |
| `isExchange` | boolean | `true` when the order is an exchange. |
| `oldOrderId` | number \| `null` | On an exchange, the id of the return leg (the item coming back). `null` for a normal order. |
| `hash` | string | The current slip hash — see [below](#printing-the-slip--hash). |

### Exchange orders

An exchange delivers new products **and** collects old ones in the same visit.

- `isExchange` is `true`, and `oldOrderId` is set. These two always agree.
- `products` is what you hand over; `productsToRetrieve` is what you take back.
- `deliveryDate` is set **at the swap**, when the customer receives the new products.
- After the swap, `finalDestination` becomes `merchant` while the old item travels back.
- `returnedDate` is set **later**, once the old item is back with the merchant.

So an exchange with `deliveryDate` set and `returnedDate` still `null` is **not finished**:
the old item is still on its way back.

### Printing the slip — `hash`

A slip's QR code must encode:

```
<orderId>#<hash>
```

For example `1234#Xk3pQw`. Depot scanners check that value; a slip with the wrong hash
is **rejected at the depot**.

The hash is computed from the order's content: recipient name, phones, address, products,
amount, and a few more. **When any of those change, the hash changes**, and every slip
printed with the old hash has to be reprinted.

So:

- **Call this endpoint right before you print** (or reprint) a slip, and use the `hash`
  it returns. Don't cache hashes.
- Changes to `status`, the dates, or `finalDestination` never change the hash, since they
  aren't part of the slip.

## 6. Every error, and what to do about it

Failures always look like this:

```json
{ "ok": false, "error": "<code>" }
```

Read the **`error` string**, not just the HTTP status. The table tells you exactly what
to do for each one. **None of these errors ever include order data.**

### Authentication problems

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **401** | `Missing authentication headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `Invalid api key` | Key is wrong, or disabled. | Check the key. Contact us if it should work. |
| **401** | `Invalid signature` | Your HMAC doesn't match. **99% of the time you signed a different string than you sent.** | Re-read [section 3](#3-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Your key is valid, but it isn't enabled for partner endpoints yet. | Email `ops@mofavo.com`. Nothing you can fix in code. |

### Your request was wrong

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **400** | `Missing or invalid field: orderId (must be a positive integer)` | `orderId` missing, a string, zero, negative, or too large. | Send a positive number. |
| **400** | `Invalid payload` | The body wasn't a JSON object. | Send `{"orderId":…}`. |
| **400** | `body is not valid JSON` | The body couldn't be parsed. | Send valid JSON. |

### The order can't be read

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `orderNotFound` | No order with that id. | Check the id. **Don't retry.** |
| **403** | `unauthorizedPartnerForRequest` | That order belongs to a **different** delivery partner. | **Don't retry.** Not your order. |

### Our side

| Status | `error` | What to do |
| --- | --- | --- |
| **500** | `internal_error` | Retry once. If it fails again, send us the `x-request-id`. |

Because this endpoint only reads, **retrying is always safe**, but only a `500` is worth
retrying. Every other error will give the same answer the second time.

## 7. Checklist before you go live

- [ ] The signature is computed over the **exact body string** you send.
- [ ] `orderId` is sent as a **number**, not a string.
- [ ] You are **not** sending a partner id in the body (we ignore it; it comes from your key).
- [ ] You fetch a **fresh `hash`** right before printing each slip, and never reuse an old one.
- [ ] Your code accepts `null` for `finalDestination`, `deliveryDate`, `returnedDate` and `oldOrderId`.
- [ ] Your code doesn't break on a `status` value it hasn't seen before.
- [ ] Your code accepts a product with no `price`.
- [ ] Only `500` is retried automatically.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 8. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 Invalid signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python use `data=`, not `json=`. |
| `403 partner_scope_not_configured` | Nothing wrong with your code. Your key isn't switched on for partner routes yet — email us. |
| `403 unauthorizedPartnerForRequest` | The order is another partner's, or you typed the wrong id. |
| Printed slips are rejected at the depot | The slip has an old `hash`. The order changed after you printed it. Fetch the order again and reprint. |
| Your code crashes on some orders | A field you assumed is always set was `null` — usually `deliveryDate`, `returnedDate`, `finalDestination`, or a product's missing `price`. |
| An exchange looks "done" but the merchant hasn't got the item back | `deliveryDate` only marks the swap. Wait for `returnedDate`. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
