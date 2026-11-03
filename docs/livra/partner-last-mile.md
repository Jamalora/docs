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
- [2. How a delivery sheet works: seven actions](#2-how-a-delivery-sheet-works-seven-actions)
- [3. The request](#3-the-request)
  - [`create`](#create)
  - [`add`](#add)
  - [`remove`](#remove)
  - [`collect`](#collect)
  - [`return`](#return)
  - [`deliver`](#deliver)
  - [`retry`](#retry)
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

## 2. How a delivery sheet works: seven actions

All seven go to the same URL. Six work on a sheet, with its `reference`; `retry` works on
orders waiting in a depot, with no `reference`. The `action` field says which step it is.

```
create ──► add ──► add ──► (remove) ──► collect ──┬──► deliver   (delivered)
 (open)   (load parcels in the depot)     (driver    └──► return    (brought back undelivered)
                                           leaves)
                                                     ──► the sheet closes when nothing is still out
```

1. **`create`** opens the sheet for one depot, one driver and one day (today or
   tomorrow). You can already put orders on it in the same call.
2. **`add`** puts orders on it as you prepare the round. They wait in the depot.
3. **`remove`** takes orders off it before the driver leaves. They stay in the depot.
4. **`collect`**: the driver takes the parcels. Every order on the sheet goes
   **out for delivery** (`inTransit`) with the driver. You send only the `reference`.
5. **`return`**: the driver is back with parcels he couldn't deliver. For each one, you say
   whether it's **cancelled** (it goes back to the merchant) or not (it will be tried again).
6. **`deliver`**: the driver delivered these orders.
7. **`retry`**: orders going back to the merchant (for example after `return` with
   `cancelled: true`) go to the customer again. Or send `add` with `"retry": true` to do it
   and put them on a sheet in one call.

When every order on a collected sheet is delivered, returned or removed, **the sheet closes
on its own**.

Rules that apply to every sheet:

- **One sheet per driver that hasn't been collected yet.** Collect (or empty) the driver's
  current sheet before you open the next one.
- **No limit on the sheet size.** Send up to 500 order ids per `create`, `add` or `remove`
  call; we handle them in chunks, so the call stays fast.
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
| `action` | string | Required. `"create"`, `"add"`, `"remove"`, `"collect"`, `"return"`, `"deliver"` or `"retry"`. |
| `reference` | string | Required, except on `retry` (where it's refused). **Your** id for the delivery sheet: 1 to 100 characters, letters, digits and `-` `_` `.` `/` only. The same on every call for that sheet. We show it as the sheet's name, prefixed `partner:` (e.g. `partner:LM-2026-000981`). |
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
| `orderIds` | number[] | Optional. 0 to 500 order ids, no duplicates. It does exactly what an `add` right after would do. |
| `retry` | boolean | Optional. As on `add`. |

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
| `orderIds` | number[] | Required. 1 to 500 order ids, no duplicates. |
| `retry` | boolean | Optional. `true`: an order going back to the merchant is sent to the customer again, then put on the sheet. Without it, such an order is refused (`not_for_customer`). |

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
| `orderIds` | number[] | Required. 1 to 500 order ids, no duplicates. |

For parcels that **haven't left**. After `collect`, a parcel the driver has is refused
(`with_driver`): when it comes back, send [`return`](#return). You can still `remove` an
order that `collect` left on the sheet (see below).

### `collect`

```json
{
  "action": "collect",
  "reference": "LM-2026-000981",
  "employeeId": 7
}
```

No other field: **every order on the sheet goes out** with the driver. (An `orderIds` field
is refused with `400 invalid_payload`.)

- An order that changed since you added it (cancelled, delivered…) doesn't go out: it's
  listed in `errors` and stays on the sheet. `remove` it. The other orders go out.
- Sending `collect` again is safe. If a call was cut short (a timeout), the resend
  finishes it.

### `return`

```json
{
  "action": "return",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orders": [
    { "orderId": 123456, "cancelled": false },
    { "orderId": 123457, "cancelled": true }
  ]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orders` | list | Required. 1 to 500 entries, each order once. Each entry has exactly `orderId` (number) and `cancelled` (`true` or `false`). |

Only **after `collect`**, for parcels the driver took and brings back **undelivered**. Each
one is checked back into the sheet's depot, and its delivery is closed as failed:

- **`cancelled: false`**: it will be tried again. It stays for the customer: put it on a
  new sheet with `add`.
- **`cancelled: true`**: the customer refused it, or it can't be delivered. It now goes
  **back to the merchant**, who is asked to check it. Send it back with [Returns](partner-return.md).

Until you send `return`, a parcel that's out with a driver is refused by [Accept In Depot](partner-accept-in-depot.md),
[Transfers](partner-transfers.md) and [Returns](partner-return.md), and by `add` on another sheet.

For an **exchange** order you already sent with `deliver` (see below), `return` takes the old
item back into the depot: it goes back to the merchant, whatever `cancelled` says.

### `deliver`

```json
{
  "action": "deliver",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orderIds": [123458, 123459]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. 1 to 500 order ids, no duplicates. |

Only **after `collect`**, for orders the driver delivered. Each one is `delivered`, with its
delivery date, and counts in the merchant's settlement.

**Exchange orders:** the driver swaps the parcels at the door and takes the old item back.
After `deliver`, the order stays out with the driver. When the driver is back in the depot,
send `return` for it.

When nothing on the sheet is still out, the sheet closes: any later call on it answers
`409 runsheet_closed`.

### `retry`

```json
{
  "action": "retry",
  "employeeId": 7,
  "orderIds": [123457]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `orderIds` | number[] | Required. 1 to 500 order ids, no duplicates. |

**No `reference`**: these orders are on no sheet, they wait in a depot.

For orders going **back to the merchant** (for example after `return` with
`cancelled: true`) that you want to **deliver to the customer** after all. Each one must be
in a depot, with nothing open on it (not on a sheet, not on a return), and not cancelled,
delivered or returned. It stays in the depot, now going to the customer: put it on a sheet
with `add`. The merchant still sees that the order needed attention until they review it.

To do both in one call, send `add` with `"retry": true` instead.

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
await callLastMile({ action: "collect", reference: "LM-2026-000981", employeeId: 7 });
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

call_last_mile({"action": "collect", "reference": "LM-2026-000981", "employeeId": 7})
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
- On `collect`, **the other orders went out.** The listed ones changed since they were
  added: they stay on the sheet, not collected. `remove` them.

**The whole call was refused**: HTTP **4xx/5xx**, nothing applied.

```json
{
  "ok": false,
  "error": "invalid_payload",
  "message": "Unknown field: orderIds (collect accepts action, reference, employeeId)",
  "details": { "field": "orderIds" }
}
```

`details` comes with `invalid_payload` and names the field that's wrong.

The answer never carries an id or status of ours. Every response, success or failure,
also carries an **`x-request-id`** header. **Log it.** If you ever open a support ticket,
that id is the first thing we ask for.

### What happens to each order

| Step | The order | Its timeline |
| --- | --- | --- |
| `add` (before `collect`) | Stays `inDepot`, on the sheet. If it still had a pick-up or a transfer **planned** (not on the road), that trip is closed as failed: the parcel leaves with this driver. | "mission created" and "set for delivery" |
| `remove` | Stays `inDepot`, off the sheet. It can go on another sheet. | "removed from runsheet" |
| `collect`, or `add` after `collect` | `inTransit` (out for delivery) with the sheet's driver. | "picked up" |
| `return`, `cancelled: false` | `inDepot` in the sheet's depot, still for the customer. | "rescheduled" and "dropped off" |
| `return`, `cancelled: true` | `inDepot` in the sheet's depot, now going back to the merchant. | "declined" and "dropped off" |
| `deliver` | `delivered`. | "dropped off" |
| `deliver`, exchange order | `inTransit`: the driver brings the old item back. | "exchanged" |
| `return`, exchange already delivered | `inDepot`, going back to the merchant. | "dropped off" |
| `retry`, or `add` with `retry: true` | Stays `inDepot`, now going to the customer. | "retry" |

You can read any order back with [Get Order](partner-get-order.md) to see its `status`.

### Webhooks

The status changes above are **not sent back to your webhook feed**. You made them with
this endpoint, so you already know: a `collect`, a `return` or a `deliver` sends you no
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
| **409** | `runsheet_closed` | The sheet is finished: everything on it was delivered, returned or removed. | **Don't retry.** Open a new sheet. |
| **409** | `not_collected` | `return` or `deliver`: the sheet isn't collected yet, its parcels are still in the depot. | `collect` it first, or use `remove`. |
| **409** | `runsheet_empty` | `collect`: there's nothing on the sheet. | `add` orders first. |
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
| `order_cancelled` | create, add, collect, deliver, retry | The order is cancelled. | It can't go out. Take it off your round. |
| `already_delivered` | create, add, collect, return, deliver, retry | The order was already delivered (for `retry`, an exchange already swapped too). | Nothing to do. |
| `already_returned` | create, add, collect, return, deliver, retry | The order was already returned to the merchant. | Nothing to do. |
| `not_for_customer` | create, add, collect, deliver | The order is going back to its merchant, not to the customer. | To deliver it to the customer after all: `retry`, or `add` with `"retry": true`. |
| `scheduled_later` | create, add | The order is scheduled for a later day than the sheet. | Put it on that day's sheet. |
| `not_in_depot` | create, add, retry | The order isn't in a depot (still on the road, or never arrived). | [Accept it in depot](partner-accept-in-depot.md) first. |
| `wrong_depot` | create, add | The order is in another depot. | [Transfer it](partner-transfers.md) here first. |
| `in_progress_elsewhere` | create, add, retry | A pick-up or a transfer of this order is on the road (for `retry`: anything open on it, e.g. a planned return). | Finish or remove it first. |
| `out_for_delivery` | create, add, retry | The order is out with a driver on a sheet. | When it comes back, send `return` on that sheet. |
| `on_other_runsheet` | create, add | The order is on another sheet that hasn't left yet. `message` names that sheet when it's yours. | `remove` it from that sheet first. |
| `not_on_this_runsheet` | remove, return, deliver | The order isn't on this sheet, or it was already returned from it. | Check the order and the `reference`. |
| `with_driver` | remove | The driver already has this parcel. | When it comes back, send `return`. |
| `not_collected` | return, deliver | `collect` left this order in the depot (it changed after you added it). | `remove` it instead. |
| `busy` | all | Another call on this order is in flight. | Send it again. |
| `internal_error` | all | Our error on this order. | Send it again later. |

## 8. Retries: every call is safe to send again

Lost the answer? Timed out? **Send the same call again.** What is already done counts as
`ok` and is never written twice:

- a `create` with the same reference, depot, driver and date,
- an order already on this sheet,
- an order already removed from this sheet,
- an order already returned on this sheet,
- an order already delivered on this sheet,
- a `retry` of an order already going to the customer,
- a `collect` of a sheet already collected (it finishes anything left).

Worth retrying automatically: `busy` (after a short pause), `internal_error` (once), and
network errors. Every other error gives the same answer the second time.

## 9. Checklist before you go live

- [ ] One full round tested on **staging**: `create` → `add` → `remove` → `collect` → `deliver` and `return`, until the sheet closes.
- [ ] Every order on a collected sheet ends with `deliver` or `return`, so the sheet closes and the merchant sees the right status.
- [ ] The signature is computed over the **exact body string** you send.
- [ ] Ids are sent as **numbers**, not strings, and no field outside the ones listed here is sent.
- [ ] Each delivery sheet has **one stable `reference`**, sent on every call for it, and a new one for each round.
- [ ] Parcels are [accepted in depot](partner-accept-in-depot.md) before they go on a sheet.
- [ ] You send `employeeId` on every call, and it is one of your employees.
- [ ] You read `ok` and, when it's `false`, the `errors` list: a `200` isn't always a full success.
- [ ] Only `busy`, `internal_error` and network errors are retried automatically.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 10. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 invalid_signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python, use `data=`, not `json=`. |
| `400 invalid_payload` with `Unknown field` | A field that action doesn't take, e.g. `driverId` on `add`, or `reference` on `retry`. Only `create` takes `depotId`, `driverId` and `date`. |
| `404 runsheet_not_found` on `add` | The `create` wasn't sent, failed, or used a different `reference` (check the case). |
| `not_in_depot` on most orders | The parcels weren't accepted in depot with [Accept In Depot](partner-accept-in-depot.md) before going on the sheet. |
| `409 driver_has_open_runsheet` on `create` | Yesterday's sheet for that driver still has orders and wasn't collected. |
| `out_for_delivery` when you re-load yesterday's parcels | The driver's undelivered parcels were never sent back with `return` on yesterday's sheet. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
