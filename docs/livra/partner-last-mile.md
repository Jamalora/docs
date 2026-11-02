# Last Mile — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_last_mile`, which keeps **your delivery sheets** in sync with ours.

A delivery sheet is a driver's round for one day: you put the parcels for that driver on
it in one of your depots, the driver collects them, and they go out for delivery. You keep
running that round in your own console. This endpoint tells us about each step, so the
orders show the right status and the merchant sees them go out.

Each sheet is found by **your own `reference`**. You never store an id of ours.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. How a delivery sheet works: four actions](#2-how-a-delivery-sheet-works-four-actions)
- [3. The request](#3-the-request)
  - [`create`](#create)
  - [`add`](#add)
  - [`remove`](#remove)
  - [`collect`](#collect)
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

You need two secrets from us. Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent**: it's only used to compute `x-signature` |

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 4](#4-how-to-build-x-signature).

These are the same key and secret you use for [Accept In Depot](partner-accept-in-depot.md),
[Transfers Between Depots](partner-transfers.md) and [Returns](partner-return.md). Your key
must be enabled for partner endpoints. If it isn't, every call returns
`403 partner_scope_not_configured`.

You also need these ids from us (we give you the lists):

| Id | What it is |
| --- | --- |
| `depotId` | One of **your** depots: where the driver collects the parcels. |
| `driverId` | One of **your** drivers. |
| `employeeId` | One of **your** employees: who did the step in your console. |

> **You do not send your partner id.** We read it from your API key, on purpose: you can
> only ever work with **your own** depots, drivers and orders.

## 2. How a delivery sheet works: four actions

All four go to the same URL, with the same `reference`. The `action` field says which
step it is.

```
create ──► add ──► add ──► (remove) ──► collect
 (open)   (load parcels in the depot)     (driver leaves: out for delivery)
```

1. **`create`** opens the sheet for one depot, one driver and one day (today or
   tomorrow). You can already put orders on it in the same call.
2. **`add`** puts orders on it as you prepare the round. They wait in the depot.
3. **`remove`** takes orders off it before the driver leaves. They stay in the depot.
4. **`collect`**: the driver takes the parcels. Every order on the sheet goes
   **out for delivery** (`inTransit`) with the driver. You send the **full list** of the
   orders the driver takes, and it must match ours exactly.

Rules that apply to every sheet:

- **One sheet per driver that hasn't been collected yet.** Collect (or empty) the driver's
  current sheet before you open the next one.
- **At most 300 orders per sheet**, and at most 100 order ids per `create`, `add` or `remove`.
- **An order goes on a sheet only when it's ready to leave from that depot.** It must be
  in that depot, going to the customer, not cancelled, delivered or returned, not on
  another sheet, and not scheduled for a later day. We don't fix anything for you: each
  refusal tells you what to do first, for example [Accept In Depot](partner-accept-in-depot.md)
  or a [transfer](partner-transfers.md).

## 3. The request

- **URL (production):** `https://livra.mofavo.com/partner_last_mile`
- **URL (staging):** `https://staging.livra.mofavo.com/partner_last_mile`
- **Method:** `POST`

Build and test against **staging** first. It works the same way, with its own data and its
own API key and secret (ask `ops@mofavo.com` for them). Your production key doesn't work
on staging, and your staging key doesn't work in production.

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
| `action` | string | Required. `"create"`, `"add"`, `"remove"` or `"collect"`. |
| `reference` | string | Required. **Your** id for the delivery sheet: 1 to 100 characters, letters, digits and `-` `_` `.` `/` only. The same on every call for that sheet. We show it as the sheet's name, prefixed `partner:` (e.g. `partner:LM-2026-000981`). |
| `employeeId` | number | Required. Positive whole number. **One of your employees**: the person who did this step. |

- Ids must be **numbers**, not strings. `"123456"` is wrong. `123456` is right.
- **Unknown fields are refused** with `400 invalid_payload`, and `details.field` names the
  field. A typo never goes unnoticed.

### `create`

```json
{
  "action": "create",
  "reference": "LM-2026-000981",
  "depotId": 3,
  "driverId": 42,
  "date": "2026-11-03",
  "employeeId": 7,
  "orderIds": [123456, 123457]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `depotId` | number | Required. One of your depots: where the parcels leave from. |
| `driverId` | number | Required. One of your drivers. |
| `date` | string | Required. The delivery day, `YYYY-MM-DD`: **today or tomorrow**, Tunis time. |
| `orderIds` | number[] | Optional. 0 to 100 order ids, no duplicates. It does exactly what an `add` right after would do. |

### `add`

```json
{
  "action": "add",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orderIds": [123458]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. 1 to 100 order ids, no duplicates. |

Each order is handled **on its own**. The valid ones go on the sheet, and the others are
listed in the answer with the reason (see [section 6](#6-the-response)). You can still
`add` after `collect` (a late load): that order goes out for delivery at once.

### `remove`

```json
{
  "action": "remove",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orderIds": [123458]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. 1 to 100 order ids, no duplicates. |

Only **before `collect`**. After it, the parcels are with the driver: a parcel that comes
back goes through [Accept In Depot](partner-accept-in-depot.md).

### `collect`

```json
{
  "action": "collect",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orderIds": [123456, 123457]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. **Every** order the driver takes: 1 to 300 ids, no duplicates. It must be exactly the list on the sheet. |

A `collect` is **all or nothing**:
- If your list differs from ours, nothing happens and the answer tells you the difference.
- If one order can't go out, nothing is written either, and the sheet isn't collected.

## 4. How to build `x-signature`

`x-signature` is an HMAC-SHA256 of the **request body**, keyed with your **API secret**,
written as lowercase hex.

```
x-signature = HMAC_SHA256( raw request body , apiSecret )  →  lowercase hex
```

`sha256=<hex>` is also accepted, if your HTTP library adds that prefix.

**The one rule that trips everyone up:** sign the **exact bytes you send**.

Build the JSON string **once**, sign *that string*, and send *that same string* as the
body. Don't build an object, sign one serialization of it, and then let your HTTP library
serialize the object again. The two strings can differ by a single space, and the
signature then fails with `401 invalid_signature`.

✅ Right:

```js
const body = JSON.stringify({ action: "add", reference: "LM-2026-000981", employeeId: 7, orderIds: [123458] }); // build ONCE
const signature = hmac(body);   // sign the string
fetch(url, { body });           // send the SAME string
```

❌ Wrong:

```js
const payload = { action: "add", reference: "LM-2026-000981", employeeId: 7, orderIds: [123458] };
const signature = hmac(JSON.stringify(payload));  // signs one string…
axios.post(url, payload);                         // …axios serializes a different one
```

## 5. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"add","reference":"LM-2026-000981","employeeId":7,"orderIds":[123458]}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://livra.mofavo.com/partner_last_mile \
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

async function callLastMile(payload) {
  const body = JSON.stringify(payload);
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://livra.mofavo.com/partner_last_mile", {
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

// A whole round:
await callLastMile({ action: "create", reference: "LM-2026-000981", depotId: 3, driverId: 42, date: "2026-11-03", employeeId: 7 });
await callLastMile({ action: "add", reference: "LM-2026-000981", employeeId: 7, orderIds: [123456, 123457, 123458] });
await callLastMile({ action: "remove", reference: "LM-2026-000981", employeeId: 7, orderIds: [123458] });
await callLastMile({ action: "collect", reference: "LM-2026-000981", employeeId: 7, orderIds: [123456, 123457] });
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"

def call_last_mile(payload: dict):
    body = json.dumps(payload, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://livra.mofavo.com/partner_last_mile",
        headers={
            "Content-Type": "application/json",
            "x-api-key": API_KEY,
            "x-signature": signature,
        },
        data=body,  # data=, NOT json= (json= would re-serialize the body and break the signature)
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()

call_last_mile({"action": "collect", "reference": "LM-2026-000981", "employeeId": 7, "orderIds": [123456, 123457]})
```

### PHP

```php
<?php
$apiKey    = getenv('API_KEY');
$apiSecret = getenv('API_SECRET');

$body = json_encode([
    'action'     => 'add',
    'reference'  => 'LM-2026-000981',
    'employeeId' => 7,
    'orderIds'   => [123458],
]);
$signature = hash_hmac('sha256', $body, $apiSecret);

$ch = curl_init('https://livra.mofavo.com/partner_last_mile');
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

**Done**: HTTP **200**.

```json
{ "ok": true }
```

This is also the answer when the step was **already done** (see [section 8](#8-retries-every-call-is-safe-to-send-again)).

**Some orders were refused**: HTTP **200**, `ok: false`, with an `errors` list.

```json
{
  "ok": false,
  "errors": [
    { "orderId": 123457, "error": "wrong_depot", "message": "This order is in another depot; transfer it here first" }
  ]
}
```

- On `create`, `add` and `remove`, **the other orders were applied.** Fix the listed ones
  and send them again, or send the whole call again: orders already done count as done.
- On `collect`, **nothing was written** and the sheet is **not collected**. Fix the listed
  orders (or `remove` them), then send the `collect` again.

**The whole call was refused**: HTTP **4xx/5xx**, nothing applied.

```json
{
  "ok": false,
  "error": "order_list_mismatch",
  "message": "The orders you sent do not match the orders on this sheet",
  "details": { "missingOrderIds": [123458], "unexpectedOrderIds": [123999] }
}
```

`details` appears in two cases:
- `order_list_mismatch`: `missingOrderIds` are on our sheet but not in your list;
  `unexpectedOrderIds` are in your list but not on our sheet.
- `invalid_payload`: `{ "field": "date" }`, the field that's wrong.

The answer never carries an id or status of ours. Every response, success or failure,
also carries an **`x-request-id`** header. **Log it.** If you ever open a support ticket,
that id is the first thing we ask for.

### What happens to each order

| Step | The order | Its timeline |
| --- | --- | --- |
| `add` (before `collect`) | Stays `inDepot`, on the sheet. If it still had a pick-up or a transfer **planned** (not on the road), that trip is closed as failed: the parcel leaves with this driver. | "mission created" and "set for delivery" |
| `remove` | Stays `inDepot`, off the sheet. It can go on another sheet. | "removed from runsheet" |
| `collect`, or `add` after `collect` | `inTransit` (out for delivery) with the sheet's driver. | "picked up" |

You can read any order back with [Get Order](partner-get-order.md) to see its `status`.

### Webhooks

The status changes above are **not sent back to your webhook feed**. You made them with
this endpoint, so you already know: a `collect` (orders `inTransit`) sends you no
`status.changed`.

Everything else still reaches you as before, including changes made on our side to the
same orders. The merchant's own order webhooks aren't affected: the merchant is still
told about every change.

## 7. Every error, and what to do about it

Read the **`error` string**, not just the HTTP status.

### Authentication problems

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **401** | `missing_headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `invalid_api_key` | The key is wrong or disabled. | Check the key. Contact us if it should work. |
| **401** | `invalid_signature` | Your HMAC doesn't match. **99% of the time you signed a different string than you sent.** | Re-read [section 4](#4-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Your key is valid, but it isn't enabled for partner endpoints yet. | Email `ops@mofavo.com`. There's nothing to fix in your code. |

### Your request was wrong

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body couldn't be parsed. | Send valid JSON. |
| **400** | `invalid_payload` | A field is missing, wrong, or unknown. `message` and `details.field` name it, e.g. `Unknown field: driverId (add accepts action, reference, employeeId, orderIds)`. | Fix the request. |
| **400** | `employee_not_found` | `employeeId` isn't one of your employees. | Check your employee ids. |
| **400** | `depot_not_found` | `create`: not one of your depots. | Check your depot ids. |
| **400** | `driver_not_found` | `create`: not one of your drivers. | Check your driver ids. |
| **400** | `invalid_date` | `create`: `date` isn't today or tomorrow (Tunis time). | Send today or tomorrow. |

### The sheet can't take this call

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **404** | `runsheet_not_found` | No sheet with this `reference`. | Send the `create` first. |
| **409** | `reference_in_use` | `create`: this `reference` is already a sheet with another depot, driver or date. | Use a new reference. |
| **409** | `driver_has_open_runsheet` | `create`: this driver already has a sheet with orders on it that hasn't been collected. | Collect it, or `remove` its orders, first. |
| **409** | `runsheet_closed` | The sheet is finished. | **Don't retry.** Open a new sheet. |
| **409** | `already_collected` | `remove`: the driver already has the parcels. | Use [Accept In Depot](partner-accept-in-depot.md) when a parcel comes back. |
| **409** | `runsheet_full` | The sheet would hold more than 300 orders. | Open another sheet for the rest. |
| **409** | `runsheet_empty` | `collect`: there's nothing on the sheet. | `add` orders first. |
| **409** | `order_list_mismatch` | `collect`: your list isn't the sheet's. See `details`. | `add` / `remove` to match, or fix your list, then `collect` again. |
| **409** | `busy` | Another call on this sheet is in flight. | Retry after a short pause. |

### Our side

| Status | `error` | What to do |
| --- | --- | --- |
| **500** | `internal_error` | Nothing was applied. Retry once; if it fails again, send us the `x-request-id`. |

### Errors on one order (in `errors`)

| `error` | Step | What it means | What to do |
| --- | --- | --- | --- |
| `order_not_found` | all | No order with this id. | Check the id. |
| `unauthorized_partner` | all | The order belongs to another delivery partner. | It isn't your order. |
| `order_cancelled` | create, add, collect | The order is cancelled. | It can't go out. Take it off your round. |
| `already_delivered` | create, add, collect | The order was already delivered. | Nothing to do. |
| `already_returned` | create, add, collect | The order was already returned to the merchant. | Nothing to do. |
| `not_for_customer` | create, add, collect | The order is going back to its merchant, not to the customer. | Fix its destination first. |
| `scheduled_later` | create, add | The order is scheduled for a later day than the sheet. | Put it on that day's sheet. |
| `not_in_depot` | create, add | The order isn't in a depot (still on the road, or never arrived). | [Accept it in depot](partner-accept-in-depot.md) first. |
| `wrong_depot` | create, add | The order is in another depot. | [Transfer it](partner-transfers.md) here first. |
| `in_progress_elsewhere` | create, add | A pick-up or a transfer of this order is on the road. | Finish it first. |
| `out_for_delivery` | create, add | A delivery of this order is already under way (with a driver, rescheduled, refused…). | [Accept it in depot](partner-accept-in-depot.md) first. |
| `on_other_runsheet` | create, add | The order is on another sheet that hasn't left yet. `message` names that sheet when it's yours. | `remove` it from that sheet first. |
| `not_on_this_runsheet` | remove | The order isn't on this sheet. | Nothing to remove. |
| `busy` | create, add, remove | Another call on this order is in flight. | Send it again. |
| `internal_error` | create, add, remove | Our error on this order. | Send it again later. |

## 8. Retries: every call is safe to send again

Lost the answer? Timed out? **Send the same call again.** What is already done counts as
`ok` and is never written twice:

- a `create` with the same reference, depot, driver and date,
- an order already on this sheet,
- an order already removed from this sheet,
- a `collect` of a sheet already collected, with the same list.

Worth retrying automatically: `busy` (after a short pause), `internal_error` (once), and
network errors. Every other error gives the same answer the second time.

## 9. Checklist before you go live

- [ ] One full round tested on **staging**: `create` → `add` → `remove` → `collect`.
- [ ] The signature is computed over the **exact body string** you send.
- [ ] Ids are sent as **numbers**, not strings, and no field outside the ones listed here is sent.
- [ ] Each delivery sheet has **one stable `reference`**, sent on every call for it, and a new one for each round.
- [ ] Parcels are [accepted in depot](partner-accept-in-depot.md) before they go on a sheet.
- [ ] You send `employeeId` on every call, and it is one of your employees.
- [ ] You read `ok` and, when it's `false`, the `errors` list: a `200` isn't always a full success.
- [ ] `collect` sends the **full** list of the sheet's orders.
- [ ] Only `busy`, `internal_error` and network errors are retried automatically.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 10. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 invalid_signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python, use `data=`, not `json=`. |
| `400 invalid_payload` with `Unknown field` | A field that action doesn't take, e.g. `driverId` on `add`. Only `create` takes `depotId`, `driverId` and `date`. |
| `404 runsheet_not_found` on `add` | The `create` wasn't sent, failed, or used a different `reference` (check the case). |
| `not_in_depot` on most orders | The parcels weren't accepted in depot with [Accept In Depot](partner-accept-in-depot.md) before going on the sheet. |
| `409 order_list_mismatch` on `collect` | Your list and ours drifted (an order was refused on `add`, or removed). `details` says which. |
| `409 driver_has_open_runsheet` on `create` | Yesterday's sheet for that driver still has orders and wasn't collected. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
