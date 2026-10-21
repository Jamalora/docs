# Returns — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_return`, which keeps **your return sheets** in sync with ours.

A return sheet is a trip that takes parcels back to their merchant: a driver leaves one of
your depots with the parcels, hands them to the merchant, and the sheet is closed. You keep
running that trip in your own console; this endpoint tells us about each step, so the
orders show the right status, the merchant sees them come back, and the returned orders
are billed and can be settled like any other
(see [Merchant Settlement](partner-merchant-settlement.md)).

Each sheet is found by **your own `reference`**. You never store an id of ours.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. How a return works: three actions](#2-how-a-return-works-three-actions)
- [3. The request](#3-the-request)
  - [`create`](#create)
  - [`add`](#add)
  - [`close`](#close)
- [4. How to build `x-signature`](#4-how-to-build-x-signature)
- [5. Copy-paste examples](#5-copy-paste-examples)
- [6. The response](#6-the-response)
  - [What happens to each order](#what-happens-to-each-order)
  - [Webhooks](#webhooks)
- [7. Every error, and what to do about it](#7-every-error-and-what-to-do-about-it)
- [8. Retries: every call is safe to send again](#8-retries-every-call-is-safe-to-send-again)
- [9. Checklist before you go live](#9-checklist-before-you-go-live)
- [10. Common mistakes](#10-common-mistakes)

---

## 1. What you need before you start

Two secrets from us. Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent** — used to compute `x-signature` |

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 4](#4-how-to-build-x-signature).

These are the same key and secret you use for [Accept In Depot](partner-accept-in-depot.md)
and [Transfers Between Depots](partner-transfers.md). Your key must be enabled for partner
endpoints; if it isn't, every call returns `403 partner_scope_not_configured`.

You also need these ids from us (we give you the lists):

| Id | What it is |
| --- | --- |
| `merchantId` | The merchant the parcels go back to. It must have a contract with you. |
| `depotId` | One of **your** depots: where the driver leaves from. |
| `driverId` | One of **your** drivers. |
| `employeeId` | One of **your** employees: who did the step in your console. |

> **You do not send your partner id.** We read it from your API key. That is deliberate —
> you can only ever work with **your own** depots, drivers and orders.

## 2. How a return works: three actions

All three go to the same URL, with the same `reference`. The `action` field says which
step it is.

```
create ──► add ──► add ──► … ──► close
 (open)   (load parcels)          (back at the merchant)
```

1. **`create`** — open the return for one merchant, one depot and one driver. The return
   is **on the road at once**. You can already put orders on it in the same call.
2. **`add`** — put orders on it as the driver loads them. Each order goes `inTransit`
   with the driver.
3. **`close`** — end it **for good**, once the driver has been to the merchant. Every
   order on it is marked **returned**, except those you list as **not returned**: they
   go back to the depot, and can go on a new return later.

Rules that apply to every return:

- **One merchant per return.** Every order must belong to the return's merchant.
- **At most 300 orders per return**, and at most 100 order ids per call.
- **A closed return stays closed.** For the next trip, open a new return with a new
  `reference`.

## 3. The request

- **URL (production):** `https://livra.mofavo.com/partner_return`
- **URL (staging):** `https://staging.livra.mofavo.com/partner_return`
- **Method:** `POST`

Build and test against **staging** first. It works the same way, with its own data and its
own API key and secret (ask `ops@mofavo.com` for them). Your production key does not work
on staging, and your staging key does not work in production.

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

That's all. **There is no `Authorization` header and no bearer token on this endpoint.**

### Fields every action needs

| Field | Type | Rules |
| --- | --- | --- |
| `action` | string | Required. `"create"`, `"add"` or `"close"`. |
| `reference` | string | Required. **Your** id for the return sheet: 1 to 100 characters (spaces at the ends are trimmed), no control characters. The same on every call for that sheet. We show it as the return's name, prefixed `partner:` (e.g. `partner:RET-2026-000154`). |
| `employeeId` | number | Required. Positive whole number. **One of your employees** — the person who did this step. |

- Ids must be **numbers**, not strings. `"123456"` is wrong. `123456` is right.
- Extra fields you add are ignored. A known field with the wrong type is a `400`.

### `create`

```json
{
  "action": "create",
  "reference": "RET-2026-000154",
  "merchantId": 655,
  "depotId": 3,
  "driverId": 42,
  "employeeId": 7,
  "orderIds": [123456, 123457]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `merchantId` | number | Required. The merchant the parcels go back to. It must have a contract with you. |
| `depotId` | number | Required. One of your depots: where the driver leaves from. |
| `driverId` | number | Required. One of your drivers: who takes the parcels. |
| `orderIds` | number[] | Optional. 0 to 100 order ids, no duplicates. Exactly the same as an `add` right after. |

### `add`

```json
{
  "action": "add",
  "reference": "RET-2026-000154",
  "employeeId": 7,
  "orderIds": [123458]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. 1 to 100 order ids, no duplicates. |

Each order is handled **on its own**: the valid ones are put on the return, the others are
listed in the answer with the reason (see [section 6](#6-the-response)).

### `close`

```json
{
  "action": "close",
  "reference": "RET-2026-000154",
  "employeeId": 7,
  "notReturnedOrderIds": [123458]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `notReturnedOrderIds` | number[] | Optional. Orders of this return the merchant did **not** take. No duplicates. Leave it out (or send `[]`) when every parcel was handed over. |

A `close` is **all or nothing**: if one order can't be closed, nothing is written and the
return stays open.

## 4. How to build `x-signature`

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
signature fails with `401 invalid_signature`.

✅ Right:

```js
const body = JSON.stringify({ action: "add", reference: "RET-2026-000154", employeeId: 7, orderIds: [123458] }); // build ONCE
const signature = hmac(body);   // sign the string
fetch(url, { body });           // send the SAME string
```

❌ Wrong:

```js
const payload = { action: "add", reference: "RET-2026-000154", employeeId: 7, orderIds: [123458] };
const signature = hmac(JSON.stringify(payload));  // signs one string…
axios.post(url, payload);                         // …axios serializes a different one
```

## 5. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"add","reference":"RET-2026-000154","employeeId":7,"orderIds":[123458]}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://livra.mofavo.com/partner_return \
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

async function callReturn(payload) {
  const body = JSON.stringify(payload);
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://livra.mofavo.com/partner_return", {
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

// A whole trip:
await callReturn({ action: "create", reference: "RET-2026-000154", merchantId: 655, depotId: 3, driverId: 42, employeeId: 7 });
await callReturn({ action: "add", reference: "RET-2026-000154", employeeId: 7, orderIds: [123456, 123457, 123458] });
await callReturn({ action: "close", reference: "RET-2026-000154", employeeId: 7, notReturnedOrderIds: [123458] });
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"

def call_return(payload: dict):
    body = json.dumps(payload, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://livra.mofavo.com/partner_return",
        headers={
            "Content-Type": "application/json",
            "x-api-key": API_KEY,
            "x-signature": signature,
        },
        data=body,  # data=, NOT json= — json= would re-serialize and break the signature
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()

call_return({"action": "close", "reference": "RET-2026-000154", "employeeId": 7, "notReturnedOrderIds": []})
```

### PHP

```php
<?php
$apiKey    = getenv('API_KEY');
$apiSecret = getenv('API_SECRET');

$body = json_encode([
    'action'     => 'add',
    'reference'  => 'RET-2026-000154',
    'employeeId' => 7,
    'orderIds'   => [123458],
]);
$signature = hash_hmac('sha256', $body, $apiSecret);

$ch = curl_init('https://livra.mofavo.com/partner_return');
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

## 6. The response

There are three shapes. **Read `ok` first.**

**Done** — HTTP **200**:

```json
{ "ok": true }
```

Also the answer when the step was **already done** (see [section 8](#8-retries-every-call-is-safe-to-send-again)).

**Some orders failed** — HTTP **200**, `ok: false`, with an `errors` list:

```json
{
  "ok": false,
  "errors": [
    { "orderId": 123457, "error": "wrong_merchant", "message": "This order belongs to another merchant" }
  ]
}
```

- On `create` / `add`: **the other orders were applied.** Fix the listed ones and send
  them again — or send the whole call again: orders already on the return count as done.
- On `close`: **nothing was written** and the return is **still open**. Fix your list,
  then send the `close` again.

**The whole call was refused** — HTTP **4xx/5xx**, nothing applied:

```json
{ "ok": false, "error": "return_not_found", "message": "No return with this reference" }
```

The answer never carries an id or status of ours. Every response — success or failure —
also carries an **`x-request-id`** header. **Log it.** If you ever open a support ticket,
that id is the first thing we ask for.

### What happens to each order

| Step | The order | Its timeline |
| --- | --- | --- |
| `add` | `inTransit`, with the return's driver. If the order was still on a pick-up or a transfer, that trip is closed as failed. An order **out for delivery** is never touched: it is refused (`out_for_delivery`). | "mission created" and "picked up" |
| `close`, returned | `returned` — or `exchange-complete` for a completed exchange, `exchange-returned` for a refused one — with its return date (`returnedDate`). Billed as a return. | "dropped off" at the merchant |
| `close`, not returned | `inDepot` at the return's depot. Not billed. It can go on a new return later. | "dropped off" at the depot |

You can read any order back with [Get Order](partner-get-order.md) to see its `status`
and `returnedDate`.

### Webhooks

The status changes above are **not sent back to your webhook feed**: you made them with
this endpoint, so you already know. An `add` (order `inTransit`) or a `close` (order
`returned`, `exchange-complete`, `exchange-returned` or back `inDepot`) sends you no
`status.changed`.

Everything else still reaches you as before, including changes made on our side to the
same orders. The merchant's own order webhooks are not affected: the merchant is still
told about every change.

## 7. Every error, and what to do about it

Read the **`error` string**, not just the HTTP status.

### Authentication problems

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **401** | `missing_headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `invalid_api_key` | Key is wrong, or disabled. | Check the key. Contact us if it should work. |
| **401** | `invalid_signature` | Your HMAC doesn't match. **99% of the time you signed a different string than you sent.** | Re-read [section 4](#4-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Your key is valid, but it isn't enabled for partner endpoints yet. | Email `ops@mofavo.com`. Nothing you can fix in code. |

### Your request was wrong

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body couldn't be parsed. | Send valid JSON. |
| **400** | `invalid_payload` | A field is missing or wrong. `message` names it, e.g. `Invalid field: orderIds (duplicate id 123458)`. | Fix the request. |
| **400** | `employee_not_found` | `employeeId` isn't one of your employees. | Check your employee ids. |
| **400** | `merchant_not_found` | `create`: that merchant has no contract with you. | Check the merchant id. |
| **400** | `depot_not_found` | `create`: not one of your depots. | Check your depot ids. |
| **400** | `driver_not_found` | `create`: not one of your drivers. | Check your driver ids. |

### The return can't take this call

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **404** | `return_not_found` | `add` / `close`: no return with this `reference`. | Send the `create` first. |
| **409** | `reference_in_use` | `create`: this `reference` is already a return with another merchant, depot or driver. | Use a new reference. |
| **409** | `return_closed` | The return is closed: an `add`, a `create`, or a `close` with other not-returned orders. | **Don't retry.** Open a new return. |
| **409** | `return_too_large` | The return would hold more than 300 orders. An `add` can also get it part-way through, when another `add` on the same return filled it at the same moment: the orders before that point stay on the return. | Close it and open a new one for the rest. |
| **409** | `busy` | Another call on this return is in flight. | Retry after a short pause. |

### Our side

| Status | `error` | What to do |
| --- | --- | --- |
| **500** | `internal_error` | Nothing was applied. Retry once; if it fails again, send us the `x-request-id`. |

### Errors on one order (in `errors`)

| `error` | Step | What it means | What to do |
| --- | --- | --- | --- |
| `order_not_found` | create, add | No order with this id. | Check the id. |
| `unauthorized_partner` | create, add | The order belongs to another delivery partner. | Not your order. |
| `not_a_return` | create, add | The order is not going back to its merchant. | Check the order; only returns can go on a return. |
| `wrong_merchant` | create, add | The order belongs to another merchant than the return's. | Put it on a return for its own merchant. |
| `already_returned` | create, add | The order was already returned to the merchant. | Nothing to do. |
| `out_for_delivery` | create, add | The order is out for delivery. | Finish (or fail) the delivery first, then add it. |
| `cancelled_check_pending` | create, add | The order has an open cancellation check. | Close the check first. |
| `on_other_return` | create, add | The order is on another open return. | Close that return first. |
| `not_on_this_return` | close | An id in `notReturnedOrderIds` is not an open order of this return. | Remove it from the list, then send the `close` again. |
| `unsupported_return_type` | close | We can't mark this order returned. | Contact us with the `x-request-id`. |
| `busy` | create, add | Another call on this order is in flight. | Send it again. |
| `internal_error` | create, add | Our error on this order. | Send it again later. |

## 8. Retries: every call is safe to send again

Lost the answer? Timed out? **Send the same call again.** What is already done counts as
`ok` and is never written twice:

- a `create` with the same reference, merchant, depot and driver,
- an order already on this return,
- a `close` with the same not-returned orders.

A `close` is final. After it, an `add`, a `create`, or a `close` with **other**
not-returned orders is refused with `return_closed`.

Worth retrying automatically: `busy` (after a short pause), `internal_error` (once), and
network errors. Every other error gives the same answer the second time.

## 9. Checklist before you go live

- [ ] One full return tested on **staging**: `create` → `add` → `close`.
- [ ] The signature is computed over the **exact body string** you send.
- [ ] Ids are sent as **numbers**, not strings.
- [ ] Each return sheet has **one stable `reference`**, sent on every call for it, and a new one for each trip.
- [ ] You send `employeeId` on every call, and it is one of your employees.
- [ ] You read `ok` and, when it's `false`, the `errors` list — a `200` is not always a full success.
- [ ] After a refused `close`, you fix the list and close again (the return is still open).
- [ ] Parcels the merchant didn't take are in `notReturnedOrderIds`.
- [ ] Only `busy`, `internal_error` and network errors are retried automatically.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 10. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 invalid_signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python use `data=`, not `json=`. |
| `404 return_not_found` on `add` | The `create` wasn't sent, failed, or used a different `reference` (check spaces and case). |
| `409 reference_in_use` on `create` | The reference is reused for another trip, or the `create` was resent with another driver, depot or merchant. |
| `409 return_closed` on `add` | The return was already closed. Open a new one for the next trip. |
| `wrong_merchant` on some orders | One return is one merchant. Split the parcels by merchant. |
| `out_for_delivery` on an order | The order still has a delivery going on. Finish or fail it in your console first. |
| A parcel the merchant refused shows as returned | It wasn't in `notReturnedOrderIds` when you closed. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
