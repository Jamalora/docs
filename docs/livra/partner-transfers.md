# Transfers Between Depots — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint,
`POST /partner_transfer`, which moves parcels **from one of your depots to another of
your depots**.

It is the exact same thing an agent does on the Transfert console: pick a route, scan the
parcels onto the sheet, confirm the driver, and send the truck. Same result, just over
HTTP.

It is the other half of [Accept In Depot](partner-accept-in-depot.md) — that one checks a
parcel *in* when a truck arrives, this one sends it *out*.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The idea in one minute](#2-the-idea-in-one-minute)
- [3. The request](#3-the-request)
  - [Who did it — `employeeId` and `employeeName`](#who-did-it--employeeid-and-employeename)
- [4. How to build `x-signature`](#4-how-to-build-x-signature)
- [5. Copy-paste examples](#5-copy-paste-examples)
- [6. `add` — put parcels on a transfer](#6-add--put-parcels-on-a-transfer)
  - [Wrong destination depot](#wrong-destination-depot)
- [7. `dispatch` — send the truck](#7-dispatch--send-the-truck)
- [8. `status` — look at a transfer](#8-status--look-at-a-transfer)
- [9. `list` — your recent transfers](#9-list--your-recent-transfers)
- [10. `remove` — take parcels back off](#10-remove--take-parcels-back-off)
- [11. `cancel` — call the whole thing off](#11-cancel--call-the-whole-thing-off)
- [12. Why a parcel was not loaded](#12-why-a-parcel-was-not-loaded)
- [13. Every call-level error](#13-every-call-level-error)
- [14. Read this before you write your retry logic](#14-read-this-before-you-write-your-retry-logic)
- [15. Checklist before you go live](#15-checklist-before-you-go-live)
- [16. Common mistakes](#16-common-mistakes)

---

## 1. What you need before you start

The same two secrets you already use for [Accept In Depot](partner-accept-in-depot.md).
Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent** — used to compute `x-signature` |

You also need your **depot ids** — we give you the list. A depot id is a number, e.g. `7`.

> **You do not send your partner id.** We read it from your API key. Nobody can use their
> key to move parcels in *your* depots, and you cannot move parcels in theirs.

Two things must be set up on our side before your key works here. Ask us to confirm both:

- your key is **enabled for partner endpoints** (otherwise every call returns `403`),
- every route you plan to use has a **transfer driver configured** (otherwise `add`
  returns `no_route_driver` and creates nothing).

## 2. The idea in one minute

A **transfer** is one truckload going from one of your depots to another, in one
direction, for one kind of parcel. It has an id (`transferId`), a list of parcels, a
driver and a status.

The whole happy path is **one call**:

```jsonc
{
  "action": "add",
  "sourceDepotId": 3,
  "destinationDepotId": 7,
  "type": "delivery",
  "orderIds": [1234, 1235, 1236],
  "dispatch": true
}
```

That creates the transfer, loads three parcels onto it, and sends the truck.

**`type`** says which flow the parcels are in:

| `type` | Parcels | We route them by |
| --- | --- | --- |
| `"delivery"` | going to the customer | the customer's address |
| `"returned"` | going back to the merchant | the merchant's address |

**One open transfer per route.** For a given source → destination → type there is at most
one transfer still being loaded. Call `add` again for the same route and your parcels join
the same transfer — you get the same `transferId` back with `"reused": true`. That is how
you load a truck in several batches, and it is why a retry after a timeout never creates a
second truck.

**A transfer closes itself.** You never call a "finish" action. When the last parcel on it
is accepted at the destination depot (through
[Accept In Depot](partner-accept-in-depot.md)), the transfer becomes `completed` on its
own.

| `status` | Meaning |
| --- | --- |
| `ready` | being loaded; you can still add and remove parcels |
| `in_progress` | dispatched; the driver is on the road |
| `completed` | every parcel arrived (or failed) at the destination |
| `cancelled` | called off before it left |

## 3. The request

- **URL:** `https://external-api.livra.tn/partner_transfer`
- **Method:** `POST`

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

That's all. **No `Authorization` header and no bearer token on this endpoint** — a depot
handles parcels for every merchant, so there is no single merchant to scope it to.

### One URL, six actions

Every request carries an **`action`** field saying what to do, plus the fields that action
needs:

| `action` | What it does | Section |
| --- | --- | --- |
| `add` | put parcels on a transfer (and optionally send it) | [6](#6-add--put-parcels-on-a-transfer) |
| `dispatch` | send the truck | [7](#7-dispatch--send-the-truck) |
| `status` | look at a transfer and the parcels on it | [8](#8-status--look-at-a-transfer) |
| `list` | your recent transfers | [9](#9-list--your-recent-transfers) |
| `remove` | take parcels back off | [10](#10-remove--take-parcels-back-off) |
| `cancel` | call the whole transfer off | [11](#11-cancel--call-the-whole-thing-off) |

An action we don't know gets a `400` listing the ones we do.

> **Why is the action in the body and not in the URL?**
> Because the signature covers the body. `status` and `cancel` take exactly the same
> fields, so if the action lived in the path, a signed "read this transfer" request could
> be replayed as "cancel this transfer". Keeping it in the signed body makes every request
> bound to one operation.

### Who did it — `employeeId` and `employeeName`

Every action accepts two optional fields. The ones that write (`add`, `dispatch`,
`remove`, `cancel`) put the result on the parcel's timeline, so an agent reading an order
in our console sees **a person**, not "some API call":

```jsonc
{
  "employeeId": 482,             // optional — OUR id for your employee
  "employeeName": "Mehdi Toumi"  // optional — the name to show if we don't know that id
}
```

**`employeeId` is the id we gave you.** It is the same `agentId` we send you on the
settlement webhooks. When we recognise it, we display **our** name for that employee,
attribute the transfer to their account exactly as if they had used our console, and
ignore `employeeName`.

**`employeeName` is the fallback.** We use it when the id matches nobody on our side, or
when you send no id at all. It is trimmed to one line and cut at 80 characters. A name
that is empty or only spaces counts as no name at all.

| What you send | What the timeline shows |
| --- | --- |
| `employeeId` we recognise | the name **we** hold for that employee, e.g. `Sarra Ben Ali (Livra)` |
| an id we don't + `employeeName` | the name **you** sent, e.g. `Mehdi Toumi (Livra)` |
| neither, or a blank name | `Partner API (Livra)` |

> **Neither field can ever fail your call.** An id we don't recognise is not an error —
> the parcels still load, the truck still leaves. Only sending them with the *wrong type*
> (a string id, a numeric name) is a `400`.
>
> The same holds on our side: if our employee directory is briefly unreachable the
> transfer is still created and the parcels still load — it is simply attributed to your
> platform, `Partner API (Livra)`, instead of to the person. Never a reason to retry.

**Send both on every call.** Then you are covered whichever side is missing the employee.

## 4. How to build `x-signature`

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

✅ Right:

```js
const body = JSON.stringify({ action: "dispatch", transferId }); // build ONCE
const signature = hmac(body);                                    // sign the string
fetch(url, { body });                                            // send the SAME string
```

❌ Wrong:

```js
const payload = { action: "dispatch", transferId };
const signature = hmac(JSON.stringify(payload));  // signs one string…
axios.post(url, payload);                         // …axios serializes a different one
```

**`action` is part of the signed body.** If you build the body first and add `action`
afterwards, the signature will not match.

## 5. Copy-paste examples

One helper, six actions. Write this once and every call below is a one-liner.

### Node.js

```js
import crypto from "node:crypto";

const API_KEY = process.env.API_KEY;
const API_SECRET = process.env.API_SECRET;
const URL = "https://external-api.livra.tn/partner_transfer";

async function transfer(action, payload = {}) {
  const body = JSON.stringify({ action, ...payload });        // action included, signed with the rest
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

// Load a truck and send it, in one call.
const { data } = await transfer("add", {
  sourceDepotId: 3,
  destinationDepotId: 7,
  type: "delivery",
  orderIds: [1234, 1235, 1236],
  dispatch: true,
});

console.log(`${data.added} loaded, ${data.refused} refused, ${data.dispatched} sent`);
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"
URL = "https://external-api.livra.tn/partner_transfer"

def transfer(action: str, **payload):
    body = json.dumps({"action": action, **payload}, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        URL,
        headers={
            "Content-Type": "application/json",
            "x-api-key": API_KEY,
            "x-signature": signature,
        },
        data=body,  # data=, NOT json= — json= would re-serialize and break the signature
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()

status, request_id, data = transfer(
    "add",
    sourceDepotId=3,
    destinationDepotId=7,
    type="delivery",
    orderIds=[1234, 1235, 1236],
    dispatch=True,
)
```

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"add","sourceDepotId":3,"destinationDepotId":7,"type":"delivery","orderIds":[1234,1235],"dispatch":true}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://external-api.livra.tn/partner_transfer \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

### PHP

```php
<?php
$apiKey    = getenv('API_KEY');
$apiSecret = getenv('API_SECRET');

function transfer(string $action, array $payload, string $apiKey, string $apiSecret) {
    $body = json_encode(array_merge(['action' => $action], $payload));
    $signature = hash_hmac('sha256', $body, $apiSecret);

    $ch = curl_init('https://external-api.livra.tn/partner_transfer');
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

## 6. `add` — put parcels on a transfer

Creates the transfer if it doesn't exist yet, loads the parcels, and optionally sends it.

### Request

```jsonc
{
  "action": "add",
  "sourceDepotId": 3,          // required — where the parcels are now. One of your depots.
  "destinationDepotId": 7,     // required — where they're going. Must differ from the source.
  "type": "delivery",          // required — "delivery" or "returned"
  "orderIds": [1234, 1235],    // required — 1 to 200 order ids
  "dispatch": false,           // optional — true sends the truck at the end of this call
  "driverId": 42,              // optional — defaults to the driver configured for this route
  "blockWrongDestination": true, // optional, default true — see "Wrong destination depot" below
  "employeeId": 482,           // optional — who loaded it (see section 3)
  "employeeName": "Mehdi Toumi" // optional — fallback name if we don't know that id
}
```

All ids are **numbers**, not strings. `"1234"` is wrong, `1234` is right.

### Response — HTTP 200

```jsonc
{
  "ok": true,
  "transferId": "4f1c8a02-…",
  "name": "RS-11/09/2026-10:22:03",   // what this transfer is called on our console
  "createdAt": "2026-09-11T10:22:03.114Z",
  "reused": false,             // true = your parcels joined a transfer that already existed
  "status": "ready",           // "in_progress" if you passed "dispatch": true
  "type": "delivery",
  "source":      { "id": 3, "name": "Tunis" },
  "destination": { "id": 7, "name": "Sousse" },
  "driver":      { "id": 42, "name": "Ali Ben Salah" },

  "added": 1,                  // now on the transfer because of this call
  "alreadyAdded": 1,           // were already on it
  "needsRepair": 0,            // couldn't be loaded — their record disagrees, see section 12
  "refused": 1,                // couldn't be loaded — see section 12
  "wrongDestination": 0,       // loaded although routing sends them elsewhere (blockWrongDestination: false)
  "dispatched": 0,             // 0 unless you passed "dispatch": true

  "parcels": [
    { "orderId": 1234, "status": "added", "missionId": 55501, "missionStatus": "ready" },
    { "orderId": 1235, "status": "already_added", "missionId": 55499, "missionStatus": "ready" },
    { "orderId": 1236, "status": "refused", "error": "wrong_destination_depot",
      "category": "terminal", "retryable": false,
      "reason": "Order 1236 is routed to Sfax, not to this transfer's destination Sousse",
      "correctDestinationDepot": { "id": 9, "name": "Sfax" } }
  ]
}
```

`parcels` comes back in the order you sent it, one entry per id.

| parcel `status` | Meaning | What to do |
| --- | --- | --- |
| `added` | On the transfer. If it carries `"warning": "wrong_destination_depot"`, it was loaded although routing sends it to `correctDestinationDepot` — only with `blockWrongDestination: false`. | Nothing — or reroute it at the destination. |
| `already_added` | Was already on it — you re-sent it, or retried. | Nothing. **This is success.** |
| `needs_repair` | The parcel is fine but its record disagrees with the transfer. **Not loaded.** | [Section 12](#12-why-a-parcel-was-not-loaded). |
| `refused` | Not loaded. `error` says why. | [Section 12](#12-why-a-parcel-was-not-loaded). |

> ### ⚠️ `ok: true` does not mean every parcel was loaded
>
> `ok` describes **the call**, not the parcels. A truckload where every single parcel was
> refused still returns `200` with `"ok": true` and `"added": 0`.
>
> **Always read `added` / `refused` / `parcels`, not just `ok`.** This is the single most
> common mistake on this endpoint.

**One exception: a parcel that belongs to another delivery partner fails the whole call.**
You get `403` and nothing is created or loaded — not even your own parcels in the same
call. The response says which ones:

```jsonc
{ "ok": false, "error": "unauthorizedPartnerForRequest", "category": "auth", "retryable": false,
  "reason": "Order 1236 belongs to another delivery partner", "orderIds": [1236] }
```

Drop those ids and send the rest again.

### Wrong destination depot

We know which depot each parcel should go to next from the source (zone routing). When
that isn't this transfer's destination, `blockWrongDestination` decides what happens:

| `blockWrongDestination` | What happens to the parcel |
| --- | --- |
| `true` (default) | **Refused**: `"error": "wrong_destination_depot"`, a `reason`, and `correctDestinationDepot`. |
| `false` | **Loaded anyway**: `"status": "added"` with `"warning": "wrong_destination_depot"`, the same `reason` and `correctDestinationDepot`. Counted in `wrongDestination`. |

```jsonc
{ "orderId": 1236, "status": "added", "missionId": 55502, "missionStatus": "ready",
  "warning": "wrong_destination_depot",
  "reason": "Order 1236 is routed to Sfax, not to this transfer's destination Sousse",
  "correctDestinationDepot": { "id": 9, "name": "Sfax" } }
```

A route with no zone routing configured on our side is never "wrong" — the parcel is
simply added.

**`dispatch: true` sends the whole transfer**, not only the parcels in this call. If an
earlier batch left parcels on it, they leave too — the truck goes with everything on it.
To keep some behind, [`remove`](#10-remove--take-parcels-back-off) them first.

## 7. `dispatch` — send the truck

Everything loaded on the transfer goes in transit, the transfer goes `in_progress`, and
the driver is on the road.

```jsonc
{ "action": "dispatch", "transferId": "4f1c8a02-…" }
```

**By `transferId` only — the whole truck leaves.** There is no partial dispatch: sending
`orderIds` returns `400`. To keep parcels behind, [`remove`](#10-remove--take-parcels-back-off)
them first.

### Response — HTTP 200

```jsonc
{
  "ok": true,
  "transferId": "4f1c8a02-…",
  "status": "in_progress",
  "dispatched": 2,          // how many parcels left
  "orderIds": [1234, 1235]  // which ones
}
```

**Sending twice is safe.** The second call returns `"dispatched": 0` with the transfer
already `in_progress`. That means the first call landed — it is success, not an error.

Dispatching an empty transfer is also `"dispatched": 0`, not an error. If you expected
parcels, check the count.

## 8. `status` — look at a transfer

One transfer and everything on it, **by `transferId` only** — there is no lookup by
parcel. Keep the `transferId` that `add` returns.

```jsonc
{ "action": "status", "transferId": "4f1c8a02-…" }
```

### Response — HTTP 200

```jsonc
{
  "ok": true,
  "transferId": "4f1c8a02-…",
  "name": "RS-11/09/2026-10:22:03",
  "status": "in_progress",
  "type": "delivery",
  "source":      { "id": 3, "name": "Tunis" },
  "destination": { "id": 7, "name": "Sousse" },
  "driver":      { "id": 42, "name": "Ali Ben Salah" },
  "createdAt": "2026-09-11T10:22:03.114Z",
  "prunedLines": 0,        // parcels this call dropped, because they moved on elsewhere
  "parcels": [
    { "orderId": 1234, "missionId": 55501, "status": "inTransit",
      "recipientName": "Sonia B.", "primaryPhone": "+216…", "primaryPhone2": null,
      "merchantName": "Example Store" }
  ]
}
```

On the `returned` flow each parcel also carries `hasUnresolvedQcTicket`.

Two things worth knowing:

- **Reading a transfer can close it.** If every parcel on it has meanwhile been accepted at
  the destination, this call completes it and answers `"status": "completed"`. Nothing is
  lost — it just means your call is what noticed.
- **A parcel that moved on elsewhere drops off a transfer that hasn't left yet**, and
  `prunedLines` counts how many. Usually that means the parcel was checked back into the
  source depot after it was loaded — see [rule 1 in section 14](#1-check-a-parcel-in-before-you-transfer-it-out--never-after).

## 9. `list` — your recent transfers

```jsonc
{
  "action": "list",
  "type": "delivery",        // required
  "sourceDepotId": 3,        // optional
  "status": "in_progress",   // optional — ready | in_progress | completed | cancelled
  "limit": 50                // optional — default 50, max 200
}
```

### Response — HTTP 200

```jsonc
{
  "ok": true,
  "transfers": [
    { "transferId": "4f1c8a02-…", "name": "RS-11/09/2026-10:22:03",
      "status": "in_progress", "type": "delivery",
      "source": { "id": 3, "name": "Tunis" }, "destination": { "id": 7, "name": "Sousse" },
      "driver": { "id": 42, "name": "Ali Ben Salah" },
      "parcelCount": 34, "createdAt": "2026-09-11T10:22:03.114Z" }
  ]
}
```

Newest first. Parcel details are not included — use [`status`](#8-status--look-at-a-transfer)
for those.

This is a plain read and never changes anything, so a transfer whose parcels have all
arrived may still be listed as `in_progress` until someone reads it directly. That keeps
listing cheap no matter how busy your network is.

## 10. `remove` — take parcels back off

Only works while the transfer is still being loaded.

```jsonc
{ "action": "remove", "transferId": "4f1c8a02-…", "orderIds": [1234] }
```

### Response — HTTP 200

```jsonc
{ "ok": true, "transferId": "4f1c8a02-…", "removed": 1, "orderIds": [1234] }
```

- `orderIds` is **required** — name the parcels to take off. There is no "remove
  everything" form, so no call can empty a truck someone else is still loading. To call
  off a whole transfer, use [`cancel`](#11-cancel--call-the-whole-thing-off).
- Removing a parcel that isn't on the transfer is not an error: `"removed": 0`.
- After the truck has left you get `transfer_closed`. A parcel on the road isn't ours to
  unload over an API — that's a console job.

## 11. `cancel` — call the whole thing off

Cancels a transfer that hasn't left, and releases every parcel on it.

```jsonc
{ "action": "cancel", "transferId": "4f1c8a02-…" }
```

### Response — HTTP 200

```jsonc
{ "ok": true, "transferId": "4f1c8a02-…", "status": "cancelled", "released": 12 }
```

**Cancelling twice is safe** — the second call returns `"released": 0`. A transfer that
already left, or already finished, answers `transfer_closed`.

## 12. Why a parcel was not loaded

These come back **per parcel**, inside `parcels[]`. They never fail the whole call — the
one exception is a parcel belonging to another partner (see [section 6](#6-add--put-parcels-on-a-transfer)).

Each one carries `category` and `retryable`, so you can branch on two fields instead of
matching every code by hand, plus a `reason` sentence you can show the agent holding the
scanner.

| `error` | `category` | What it means | What to do |
| --- | --- | --- | --- |
| `already_delivered_or_returned` | `terminal` | This parcel's journey is over. | Don't retry. |
| `wrong_final_destination` | `terminal` | A return sent on `"delivery"`, or the other way round. | Send it on the other `type`. |
| `wrong_destination_depot` | `terminal` | Routing says this parcel goes somewhere else. Carries `correctDestinationDepot`. Only when `blockWrongDestination` is `true` (the default). | Put it on a transfer to that depot — or send `blockWrongDestination: false` to load it anyway. |
| `active_last_mile_mission` | `terminal` | It's already out for final delivery. | Recall that mission, then retry. |
| `active_last_mile_runsheet` | `terminal` | It's loaded on a last-mile sheet. | Take it off there, then retry. |
| `unresolved_qc_ticket` | `repairable` | Returns flow only: an open quality-check ticket. | Close the ticket, then retry. |
| `order_not_found` | `terminal` | No order with that id. | Check the id. |
| `unauthorizedPartnerForRequest` | `terminal` | That parcel belongs to another delivery partner. Normally the whole call fails with `403` first (see [section 6](#6-add--put-parcels-on-a-transfer)). | Don't retry. Not your parcel. |

### `needs_repair`

The parcel is where it should be, but its record doesn't say so yet — most often it was
never checked into the source depot. It was **not** loaded, and the entry tells you what
is wrong:

```jsonc
{ "orderId": 1237, "status": "needs_repair",
  "repairs": ["not_in_depot", "wrong_depot"], "retryable": true }
```

| repair | Meaning |
| --- | --- |
| `not_in_depot` | The order isn't marked as being in a depot. |
| `wrong_depot` | It's recorded at a different depot than the source. |
| `unresolved_missions` | An older mission on it is still open. |
| `active_ready_transfer` | It's loaded on another transfer that hasn't left. |

**Usually a step was skipped.** A parcel that arrived at the source depot should have been
checked in with [Accept In Depot](partner-accept-in-depot.md) first — do that and re-send
it, and most `needs_repair` results disappear.

If they don't, the parcel's record has to be corrected on the console and then re-sent. We
deliberately don't correct it for you over the API: those corrections rewrite a parcel's
status, its recorded position and its open missions, which is a lot more power than "move
this parcel". Tell us if you hit this often — we'd rather see the numbers first.

## 13. Every call-level error

These fail the whole call:

```jsonc
{ "ok": false, "error": "no_route_driver", "category": "configuration", "retryable": false }
```

`category` and `retryable` come back on every failure, including validation ones.

### Authentication problems

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **401** | `Missing authentication headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `Invalid api key` | Key is wrong, or disabled. | Check the key. |
| **401** | `Invalid signature` | Your HMAC doesn't match. **99% of the time you signed a different string than you sent.** | Re-read [section 4](#4-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Your key is valid but isn't enabled for partner endpoints. | Email `ops@mofavo.com`. Nothing to fix in code. |
| **403** | `unauthorizedPartnerForRequest` | The transfer — or an order you sent in `add` or `remove` — belongs to another delivery partner. `reason` says which; `add` and `remove` also list the foreign `orderIds`. | Don't retry. Drop those ids. |

### Your request was wrong

| Status | `error` | Fix |
| --- | --- | --- |
| **400** | `Missing or invalid field: action (must be one of …)` | Use one of the six actions in [section 3](#3-the-request). |
| **400** | `Missing or invalid field: …` | The message names the field. Send it as a number where a number is expected. |
| **400** | `same_depot` | Source and destination are the same depot. |
| **400** | `depot_not_found` | A depot id isn't one of yours, or doesn't exist. Check your depot mapping. |
| **400** | `driver_not_found` | `driverId` isn't an active transfer driver of yours. Leave it out to use the route's driver. |
| **400** | `transfer_not_found` | Unknown `transferId`. Check the id. |
| **400** | `Invalid field: orderIds (dispatch sends the whole transfer; …)` | `dispatch` takes only `transferId`. `remove` the parcels that should stay, then dispatch. |
| **400** | `Invalid field: blockWrongDestination (must be a boolean)` | Send `true` or `false`, or leave it out. |
| **400** | `transfer_closed` | The transfer already left, finished, or was cancelled. Start a new one. |
| **400** | `Missing or invalid field: orderIds (must be a non-empty array of order ids)` | `remove` needs the parcels to take off. To drop the whole transfer, use `cancel`. |
| **400** | `Invalid field: employeeId (must be a positive integer)` | Send the id as a number, e.g. `482`, not `"482"`. An id we don't recognise is fine — see [section 3](#who-did-it--employeeid-and-employeename). |
| **400** | `Invalid field: employeeName (must be a string)` | Send a string, or leave the field out. |

### Our side

| Status | `error` | What to do |
| --- | --- | --- |
| **400** | `no_route_driver` | No transfer driver configured for this route. **Nothing was created.** Email us — it's a setup gap, not a bug in your code. |
| **400** | `no_driver_assigned` | The transfer has no driver. Same: contact us. |
| **400** | `destination_depot_missing` | The transfer's destination depot is misconfigured on our side. Send us the `x-request-id`. |
| **409** | `transfer_in_progress` | Another call is working on this same transfer right now. **Wait ~1 second and retry.** This is the only response to retry automatically. |
| **500** | `internal_error` | Retry once. If it fails again, send us the `x-request-id`. |

Every response — success or failure — carries an **`x-request-id`** header. **Log it.**
It's the first thing we ask for on a support ticket.

## 14. Read this before you write your retry logic

### 1. Check a parcel in before you transfer it out — never after

The order is: [Accept In Depot](partner-accept-in-depot.md) when the parcel **arrives** at
a depot, then `add` when it **leaves**.

Accepting a parcel into a depot cancels any onward leg still waiting to leave **that same
depot**. That's right when the parcel has just arrived — and wrong when you loaded it onto
a transfer ten minutes ago.

So if you run accepts as a reconciliation job, **skip parcels that are already on a
transfer** — keep the `transferId` each `add` returns; `status` on it lists what's aboard.
A parcel unloaded this way isn't lost: it drops off the transfer on the next read
(`prunedLines`), and `dispatch` leaves it behind instead of sending a parcel that isn't on
the truck.

### 2. Repeating a call is always safe — and each repeat means something

| You repeat | You get | Meaning |
| --- | --- | --- |
| `add` with the same parcels | `already_added` | They're on the transfer. Success. |
| `add` for the same route | `"reused": true` | Same transfer, no second truck. |
| `dispatch` | `"dispatched": 0` | It already left. Success. |
| `cancel` | `"released": 0` | Already cancelled. Success. |
| `remove` | `"removed": 0` | Already off. Success. |

**None of these are errors.** Only `409 transfer_in_progress` should be retried
automatically, after a short wait.

```js
const { status, data } = await transfer("add", { ...route, orderIds });

if (status === 409) {
  retryAfter(1000);                       // someone else is touching this transfer
} else if (status === 500) {
  retryOnce();
} else if (status !== 200) {
  showError(data.error);                  // real problem — don't retry
} else {
  // 200: the CALL worked. Now look at the parcels.
  for (const parcel of data.parcels) {
    if (parcel.status === "added" || parcel.status === "already_added") {
      markLoaded(parcel.orderId);
      if (parcel.warning) showParcelWarning(parcel.orderId, parcel.reason); // wrong depot, loaded anyway
    }
    else if (parcel.retryable) queueForLater(parcel.orderId);
    else showParcelProblem(parcel.orderId, parcel.reason ?? parcel.repairs);
  }
}
```

### 3. Every attempt is recorded, including the refused ones

Each parcel you send writes an entry to that order's internal timeline, exactly like a
scan on the console. Identical repeat attempts within **60 seconds** are collapsed into
one entry. These entries are visible to operations only, never to merchants.

So don't be surprised if your call count and our timeline entry count differ.

## 15. Checklist before you go live

- [ ] The signature is computed over the **exact body string** you send, **`action`
      included**.
- [ ] All ids are sent as **numbers**, not strings.
- [ ] You are **not** sending a partner id (we ignore it; it comes from your key).
- [ ] You send `employeeId` (and `employeeName` as a fallback) on every writing call, so
      the timeline names a person.
- [ ] You read `added` / `refused` / `parcels`, not just `ok`.
- [ ] `already_added` is treated as **success**, not an error.
- [ ] `409 transfer_in_progress` retries after a short wait. Nothing else retries
      automatically.
- [ ] Your accept reconciliation job skips parcels already on a transfer.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.
- [ ] One full round trip tested on staging: `add` → `dispatch` → accept at the
      destination → the transfer reaches `completed` on its own.

## 16. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 Invalid signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python use `data=`, not `json=`. |
| `401 Invalid signature` after you added `action` | You built and signed the body first, then added the field. Build it once, with `action` in it. |
| Agents told "everything loaded" when it didn't | You checked `ok` and stopped. Read `added` and `parcels`. |
| `403 unauthorizedPartnerForRequest` on an `add` and nothing loaded | One of the ids belongs to another partner. `orderIds` in the response lists them; drop them and re-send. |
| Parcels vanish off a transfer before dispatch | Your accept job is re-accepting them at the source depot. See [rule 1](#1-check-a-parcel-in-before-you-transfer-it-out--never-after). |
| Two trucks for the same route | Not possible through this API — `add` reuses the open transfer. If you see two, one was built on the console. |
| `400 no_route_driver` | No driver configured for that source → destination pair. Setup on our side; email us. |
| `403 partner_scope_not_configured` | Nothing wrong with your code. Your key isn't switched on for partner routes yet. |
| Everything returns `needs_repair` | Parcels aren't being checked into the source depot first. Run [Accept In Depot](partner-accept-in-depot.md) on arrival. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
