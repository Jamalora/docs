# Transfers Between Depots — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint,
`POST /partner_transfer`, which sends a parcel **from one of your depots to another of
your depots**.

Call it when a parcel is **scanned at the departure depot** to leave on a truck. We create
the parcel's transfer mission, ready to go, with its driver, and record it on the order's
timeline. That's all: there is no transfer sheet to open, dispatch or close.

When the parcel arrives, check it in at the destination with
[Accept In Depot](partner-accept-in-depot.md). That closes the transfer. Changed your
mind before the parcel leaves? [Cancel it](#8-cancelling-a-transfer) with the same
endpoint.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The request](#2-the-request)
- [3. How to build `x-signature`](#3-how-to-build-x-signature)
- [4. Copy-paste examples](#4-copy-paste-examples)
- [5. The answer](#5-the-answer)
- [6. Every error](#6-every-error)
- [7. Scanning twice, and changing your mind](#7-scanning-twice-and-changing-your-mind)
- [8. Cancelling a transfer](#8-cancelling-a-transfer)
- [9. Checklist before you go live](#9-checklist-before-you-go-live)

---

## 1. What you need before you start

The same two secrets you already use for [Accept In Depot](partner-accept-in-depot.md).
Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent**, used to compute `x-signature` |

You also need **our ids** for your depots and your drivers. We give you the lists. Both are
numbers, e.g. depot `7`, driver `42`.

> **You do not send your partner id.** We read it from your API key. Nobody can use their
> key to move parcels in *your* depots, and you cannot move parcels in theirs.

## 2. The request

- **URL:** `https://livra.mofavo.com/partner_transfer`
  (staging: `https://staging.livra.mofavo.com/partner_transfer`)
- **Method:** `POST`

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 3>
```

No `Authorization` header and no bearer token.

### Body

```json
{
  "orderId": 1234,
  "sourceDepotId": 3,
  "destinationDepotId": 7,
  "driverId": 42,
  "onWrongDestination": "block",
  "employeeId": 482,
  "employeeName": "Mehdi Toumi"
}
```

| Field | Required | What it is |
| --- | --- | --- |
| `orderId` | yes | The parcel. It must be one of your orders, and not delivered or returned (an exchange travelling back to the merchant is fine). |
| `sourceDepotId` | yes | The depot it leaves from: one of yours. |
| `destinationDepotId` | yes | The depot it goes to: one of yours, and not the same as `sourceDepotId`. |
| `driverId` | yes | **Our** id for the driver taking it: one of your drivers. The timeline shows the name we hold for that driver. |
| `onWrongDestination` | no | `"block"` (the default) or `"warn"`: what to do when your zone routing sends this parcel to **another depot** than `destinationDepotId`. `block` refuses the scan; `warn` accepts it and tells you in `warnings`. |
| `action` | no | `"transfer"` (the default) to send the parcel, or `"cancel"` to [call it off](#8-cancelling-a-transfer). |
| `employeeId` | no | **Our** id for the employee who scanned it (the `agentId` we send you on the settlement webhooks). |
| `employeeName` | no | The name to show when we don't know `employeeId`, or when you send none. Trimmed to one line, cut at 80 characters. |

All ids are **numbers**, not strings. Unknown extra fields are ignored.

**The destination is checked against your zone routing**, the same one your console uses:
the recipient's town for a delivery, the merchant's town for a return. When no routing is
set up for the parcel, any destination is accepted.

**Who scanned it.** With an `employeeId` we know, the timeline shows our name for that
employee, e.g. `Sarra Ben Ali (Livra)`. With an id we don't know, it shows your
`employeeName`, e.g. `Mehdi Toumi (Livra)`. With neither, `Partner API (Livra)`. An
unknown `employeeId` is **never** an error. Send both on every call.

## 3. How to build `x-signature`

Exactly as on [Accept In Depot](partner-accept-in-depot.md#3-how-to-build-x-signature): an
HMAC-SHA256 of the **request body**, keyed with your **API secret**, as lowercase hex.

```
x-signature = HMAC_SHA256( raw request body , apiSecret )  →  lowercase hex
```

`sha256=<hex>` is also accepted if your HTTP library adds that prefix.

**The one rule that trips everyone up:** sign the **exact bytes you send**. Build the JSON
string once, sign *that string*, and send *that same string*. If your HTTP library
serializes the object again, the two strings can differ by one space and you get
`401 Invalid signature`.

## 4. Copy-paste examples

### Node.js

```js
import crypto from "node:crypto";

const API_KEY = process.env.API_KEY;
const API_SECRET = process.env.API_SECRET;
const URL = "https://livra.mofavo.com/partner_transfer";

async function transfer(payload) {
  const body = JSON.stringify(payload); // build ONCE
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch(URL, {
    method: "POST",
    headers: {
      "content-type": "application/json",
      "x-api-key": API_KEY,
      "x-signature": signature,
    },
    body, // the SAME string that was signed
  });

  return { status: res.status, requestId: res.headers.get("x-request-id"), data: await res.json() };
}

const { status, data } = await transfer({
  orderId: 1234,
  sourceDepotId: 3,
  destinationDepotId: 7,
  driverId: 42,
  employeeId: 482,
  employeeName: "Mehdi Toumi",
});
// status 200, data { ok: true, warnings: [] }
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"
URL = "https://livra.mofavo.com/partner_transfer"

def transfer(**payload):
    body = json.dumps(payload, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        URL,
        headers={
            "Content-Type": "application/json",
            "x-api-key": API_KEY,
            "x-signature": signature,
        },
        data=body,  # data=, NOT json= (json= would re-serialize and break the signature)
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()

status, request_id, data = transfer(
    orderId=1234, sourceDepotId=3, destinationDepotId=7, driverId=42,
    employeeId=482, employeeName="Mehdi Toumi",
)
```

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"orderId":1234,"sourceDepotId":3,"destinationDepotId":7,"driverId":42,"employeeId":482}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://livra.mofavo.com/partner_transfer \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

### PHP

```php
<?php
function transfer(array $payload, string $apiKey, string $apiSecret) {
    $body = json_encode($payload);
    $signature = hash_hmac('sha256', $body, $apiSecret);

    $ch = curl_init('https://livra.mofavo.com/partner_transfer');
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
    return [$status, json_decode($response, true)];
}
```

## 5. The answer

**200** means the parcel is on its way:

```json
{ "ok": true, "warnings": [] }
```

`warnings` lists what looked wrong but **did not stop** the scan. Show them to the
operator:

| `code` | Also in the warning | What it means |
| --- | --- | --- |
| `wrong_destination_depot` | `correctDestinationDepot: { id, name }` | Only with `onWrongDestination: "warn"`: your zone routing sends this parcel to `correctDestinationDepot`. |
| `not_in_depot` | `orderStatus` | The parcel was not checked in to a depot before this scan. |
| `wrong_source_depot` | | The parcel was last checked in at another depot than `sourceDepotId`. |

```json
{
  "ok": true,
  "warnings": [
    { "code": "wrong_destination_depot", "correctDestinationDepot": { "id": 7, "name": "Sousse" } }
  ]
}
```

Anything else is a refusal, and **nothing was changed**:

```json
{
  "ok": false,
  "category": "terminal",
  "error": "wrong_destination_depot",
  "message": "Zone routing sends this order to another depot",
  "correctDestinationDepot": { "id": 7, "name": "Sousse" }
}
```

- `category: "error"`: **your request is wrong** (a field, an id, the signature). Fix it.
- `category: "terminal"` (always **409**): **the request is fine, but this parcel can't do
  this now**. Show `message` to the operator.
- Branch on `error`. `message` is a readable English sentence and may change.

Every response carries an **`x-request-id`** header. **Log it**: it's the first thing we
ask for on a support ticket.

## 6. Every error

| Status | `category` | `error` | What went wrong | What to do |
| --- | --- | --- | --- | --- |
| **400** | `error` | `invalid_json` | The body isn't JSON. | Fix the request. |
| **400** | `error` | `invalid_payload` | A field is missing or wrong (e.g. a string where we expect a number, or both depots the same). `message` names it. | Fix the request. |
| **400** | `error` | `order_not_found` | No such order. | Check the id. Don't retry. |
| **403** | `error` | `unauthorized_partner` | The order belongs to another delivery partner. | Don't retry. |
| **400** | `error` | `source_depot_not_found` | The departure depot isn't one of yours, or the id is wrong. | Check your depot mapping. |
| **400** | `error` | `destination_depot_not_found` | Same, for the destination depot. | Check your depot mapping. |
| **400** | `error` | `driver_not_found` | The driver isn't one of yours, or the id is wrong. | Check your driver mapping. |
| **409** | `terminal` | `already_delivered` | The parcel was already delivered. | Don't retry. |
| **409** | `terminal` | `already_returned` | The parcel's return is complete. | Don't retry. |
| **409** | `terminal` | `active_last_mile_mission` | The parcel is out for delivery. | Resolve its delivery mission first. |
| **409** | `terminal` | `unresolved_qc_ticket` | A return whose quality check is still open. We note the attempt on the timeline. | Close the QC ticket, then scan again. |
| **409** | `terminal` | `wrong_destination_depot` | Your zone routing sends the parcel to `correctDestinationDepot`. | Send it there, or scan again with `onWrongDestination: "warn"`. |
| **409** | `terminal` | `transfer_already_started` | Cancel only: the parcel has already left the depot. | Don't retry. |
| **409** | `terminal` | `no_active_transfer` | Cancel only: the parcel has no open transfer. | Nothing to do. |
| **401** | `error` | `missing_headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `error` | `invalid_api_key` | The key is wrong, or disabled. | Check the key. |
| **401** | `error` | `invalid_signature` | Your HMAC doesn't match. Almost always: you signed a different string than you sent. | Re-read [section 3](#3-how-to-build-x-signature). |
| **403** | `error` | `partner_scope_not_configured` | Your key isn't enabled for partner endpoints yet. | Email `ops@mofavo.com`. |
| **409** | `error` | `transfer_in_progress` | Another call for this same parcel is running right now. | **Wait about a second and retry.** The only response to retry automatically. |
| **500** | `error` | `internal_error` | Our side. | Retry once, then send us the `x-request-id`. |

## 7. Scanning twice, and changing your mind

| You send | What happens |
| --- | --- |
| The **same** scan again (same parcel, depots and driver) | `200 { "ok": true, ... }`, and nothing changes. Retrying after a timeout is always safe. |
| The same parcel with **another destination or driver** | `200 { "ok": true, ... }`. The previous transfer of that parcel is closed as failed and a new one replaces it. |
| A **cancel** while the parcel is still in the depot | `200 { "ok": true }`. The transfer is closed as failed; see [section 8](#8-cancelling-a-transfer). |
| [Accept In Depot](partner-accept-in-depot.md) at the **destination** | The transfer is complete. |
| Accept In Depot at **another depot** | The parcel is checked in there, and its transfer is closed as failed. |

**Check a parcel in before you scan it out, never after.** Accepting a parcel into a depot
closes any transfer still leaving **that same depot**. So run Accept In Depot when the
parcel **arrives** at the departure depot, then this endpoint when it **leaves**. If you
run accepts as a reconciliation job, skip parcels you have already scanned out.

## 8. Cancelling a transfer

Send the same endpoint `action: "cancel"` and the parcel, **while it is still in the
departure depot**:

```json
{ "action": "cancel", "orderId": 1234, "employeeId": 482, "employeeName": "Mehdi Toumi" }
```

- **200** `{ "ok": true }`: the parcel's transfer is closed as failed (it stays on the
  timeline, with who cancelled it). You can scan the parcel onto a new transfer afterwards.
- **409** `transfer_already_started`: the driver has already picked the parcel up. A
  transfer can only be cancelled before it starts.
- **409** `no_active_transfer`: the parcel has no open transfer (never sent, already
  arrived, or already cancelled). Sending the same cancel twice gives this the second time.

No depot or driver is needed: a parcel has at most one open transfer.

## 9. Checklist before you go live

- [ ] The signature is computed over the **exact body string** you send.
- [ ] All ids are sent as **numbers**, not strings.
- [ ] You use **our** ids for depots and drivers, from the lists we gave you.
- [ ] You send `employeeId` (and `employeeName` as a fallback) on every call.
- [ ] `409 transfer_in_progress` retries after a short wait. Nothing else retries
      automatically.
- [ ] Your accept job doesn't re-accept parcels at the depot they were just scanned out of.
- [ ] You branch on `error` (and `category`), not on `message`.
- [ ] You show `warnings` to the operator, and `correctDestinationDepot` on a
      `wrong_destination_depot`.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.
- [ ] One full round trip tested on staging: transfer → Accept In Depot at the destination,
      and transfer → cancel.

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
