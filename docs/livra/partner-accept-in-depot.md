# Accept In Depot — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_accept_in_depot`, which checks a parcel **into one of your depots**.

It is the exact same thing an agent does when a truck arrives and they scan the
parcel on the depot console. Same result, just over HTTP.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The request](#2-the-request)
  - [Who approved it — `employeeId` and `employeeName`](#who-approved-it--employeeid-and-employeename)
- [3. How to build `x-signature`](#3-how-to-build-x-signature)
- [4. Copy-paste examples](#4-copy-paste-examples)
- [5. The success response](#5-the-success-response)
- [6. Every error, and what to do about it](#6-every-error-and-what-to-do-about-it)
- [7. Read this before you write your retry logic](#7-read-this-before-you-write-your-retry-logic)
- [8. Checklist before you go live](#8-checklist-before-you-go-live)
- [9. Common mistakes](#9-common-mistakes)

---

## 1. What you need before you start

Two secrets from us. Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent** — used to compute `x-signature` |

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 3](#3-how-to-build-x-signature).

You also need your **depot ids**. We give you the list. A depot id is a number, e.g. `7`.

> **You do not send your partner id.** We read it from your API key. That is deliberate —
> it means nobody can use their key to move parcels in *your* depots.

## 2. The request

- **URL:** `https://external-api.livra.tn/partner_accept_in_depot`
- **Method:** `POST`

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 3>
```

That's all. **There is no `Authorization` header and no bearer token on this endpoint.**
(Other Livra endpoints use a merchant token. This one does not — a depot receives parcels
for every merchant, so there is no single merchant to scope it to.)

### Body

```json
{
  "orderId": 1234,
  "depotId": 7,
  "employeeId": 482,
  "employeeName": "Mehdi Toumi"
}
```

Two required fields, two optional ones.

| Field | Type | Rules |
| --- | --- | --- |
| `orderId` | number | Required. Positive whole number. The order you scanned. |
| `depotId` | number | Required. Positive whole number. **One of your own depots.** |
| `employeeId` | number | Optional. The employee who approved the accept — see below. |
| `employeeName` | string | Optional. The name to show if we don't know that `employeeId`. |

- Ids must be **numbers**, not strings. `"1234"` is wrong. `1234` is right.
- Extra fields you add are ignored, they don't break anything.

### Who approved it — `employeeId` and `employeeName`

Without these, the order's timeline says the accept came from your platform. With them, it
names the person — which is what an agent looking at a parcel actually needs.

**`employeeId` is the id we gave you.** It is the same `agentId` we send you on the
settlement webhooks. When we recognise it, we display **our** name for that employee and
ignore `employeeName`. When we don't, we fall back to `employeeName`. With neither, the
accept is attributed to your platform.

| What you send | What the timeline shows |
| --- | --- |
| `employeeId` we recognise | the name **we** hold, e.g. `Sarra Ben Ali (Livra)` |
| an id we don't + `employeeName` | the name **you** sent, e.g. `Mehdi Toumi (Livra)` |
| neither | `Partner API (Livra)` |

> **Neither field can ever fail your call.** An id we don't recognise is not an error — the
> parcel is still accepted. Only sending them with the *wrong type* (a string id, a numeric
> name) is a `400`.

**Send both on every call.** Then you are covered whichever side is missing the employee.

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
const body = JSON.stringify({ orderId: 1234, depotId: 7 }); // build ONCE
const signature = hmac(body);                               // sign the string
fetch(url, { body });                                       // send the SAME string
```

❌ Wrong:

```js
const payload = { orderId: 1234, depotId: 7 };
const signature = hmac(JSON.stringify(payload));  // signs one string…
axios.post(url, payload);                         // …axios serializes a different one
```

## 4. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"orderId":1234,"depotId":7}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://external-api.livra.tn/partner_accept_in_depot \
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

export async function acceptInDepot(orderId, depotId) {
  const body = JSON.stringify({ orderId, depotId });
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://external-api.livra.tn/partner_accept_in_depot", {
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

def accept_in_depot(order_id: int, depot_id: int):
    body = json.dumps({"orderId": order_id, "depotId": depot_id}, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://external-api.livra.tn/partner_accept_in_depot",
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

$body = json_encode(['orderId' => 1234, 'depotId' => 7]);
$signature = hash_hmac('sha256', $body, $apiSecret);

$ch = curl_init('https://external-api.livra.tn/partner_accept_in_depot');
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
  "ok": true,
  "orderId": 1234,
  "depotId": 7,
  "depotName": "Sousse",
  "acceptedAt": "2026-09-11T10:22:03.114Z"
}
```

All five fields are always there. There is only one kind of success, so
`if (status === 200)` is enough — you do not need to branch on anything else.

When this returns 200, on our side:

- the order's status becomes `inDepot` at that depot,
- the incoming transfer leg is closed,
- the inter-agency runsheet is settled if this was its last open leg,
- the drop-off is written to the order's timeline.

Every response — success or failure — also carries an **`x-request-id`** header.
**Log it.** If you ever open a support ticket, that id is the first thing we ask for.

## 6. Every error, and what to do about it

Failures always look like this:

```json
{ "ok": false, "error": "<code>" }
```

Read the **`error` string**, not just the HTTP status. The table tells you exactly what
to do for each one.

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
| **400** | `Missing or invalid field: orderId (must be a positive integer)` | `orderId` missing, a string, zero, or negative. | Send a positive number. |
| **400** | `Missing or invalid field: depotId (must be a positive integer)` | Same, for `depotId`. | Send a positive number. |
| **400** | `Missing or invalid field: employeeId (must be a positive integer)` | `employeeId` was a string, zero, or negative. | Send a positive number, or leave it out. An id we don't recognise is fine. |
| **400** | `Missing or invalid field: employeeName (must be a string)` | `employeeName` wasn't a string. | Send a string, or leave it out. |
| **400** | `Invalid payload` | The body wasn't a JSON object. | Send `{"orderId":…,"depotId":…}`. |

### The parcel can't be accepted

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `orderNotFound` | No order with that id. | Check the id. **Don't retry.** |
| **403** | `unauthorizedPartnerForRequest` | That order belongs to a **different** delivery partner. | **Don't retry.** Not your parcel. |
| **400** | `orderShouldNotBeAcceptedHere` | That depot isn't yours, or the depot id is wrong. | Check your depot mapping. **Don't retry.** |
| **400** | `orderAlreadyDelivered` | Already delivered. Final. | **Don't retry.** |
| **400** | `orderAlreadyReturned` | Return already completed. Final. | **Don't retry.** |
| **400** | `last_mile_scan_block_active_mission` | A last-mile mission is still open on this order. | Close that mission, **then** retry. |
| **400** | `orderAlreadyInThisDepot` | Already accepted at this depot. Also returns `acceptedAt`. | **Treat as SUCCESS.** See [section 7](#7-read-this-before-you-write-your-retry-logic). |

### Our side

| Status | `error` | What to do |
| --- | --- | --- |
| **409** | `accept_in_progress` | Another accept of this same order is running right now. **Wait ~1 second and retry.** This is the only response you should retry automatically. |
| **500** | `internal_error` | Retry once. If it fails again, send us the `x-request-id`. |

## 7. Read this before you write your retry logic

### `orderAlreadyInThisDepot` means it worked

```json
{ "ok": false, "error": "orderAlreadyInThisDepot", "acceptedAt": "2026-09-11T09:05:41.882Z" }
```

Your call timed out, you retried, and the **first** call had actually succeeded. So the
retry tells you: it's already in. `acceptedAt` is when it actually landed.

**The parcel is in the depot. Show the agent a green check, not a red error.** This is the
single most common mistake integrators make on this endpoint.

Yes, the HTTP status is 400 and `ok` is `false`. Ignore that here. Handle it like this:

```js
const { status, data } = await acceptInDepot(orderId, depotId);

if (status === 200 || data.error === "orderAlreadyInThisDepot") {
  showSuccess();            // parcel is in the depot either way
} else if (status === 409) {
  retryAfter(1000);         // someone else is accepting it right now
} else if (status === 500) {
  retryOnce();
} else {
  showError(data.error);    // real problem — don't retry
}
```

### Retrying is always safe

We serialize accepts of the same order on our side. Two calls at the same time cannot both
check the parcel in: one wins, the other gets `orderAlreadyInThisDepot` (or `409` if it
waited too long). You can never double-accept a parcel.

### Every attempt is logged, including the failures

Each call writes an entry to the order's internal timeline, exactly like a scan on the
depot console does. Identical repeat attempts within **60 seconds** are collapsed into a
single entry — **attempts by two different employees are never collapsed**, so nobody's
scan disappears. These entries are visible to operations only, never to merchants.

So don't be surprised if your call count and our timeline entry count differ — that's the
60-second collapsing.

## 8. Checklist before you go live

- [ ] The signature is computed over the **exact body string** you send.
- [ ] `orderId` and `depotId` are sent as **numbers**, not strings.
- [ ] You are **not** sending a partner id in the body (we ignore it; it comes from your key).
- [ ] You send `employeeId` (and `employeeName` as a fallback), so the timeline names a person.
- [ ] `orderAlreadyInThisDepot` is handled as a **success**.
- [ ] `409 accept_in_progress` retries after a short wait.
- [ ] Nothing else is retried automatically.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 9. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 Invalid signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python use `data=`, not `json=`. |
| `401 Invalid signature` only on some calls | Unicode or spacing differences in your JSON. Build the string once, sign it, send it. |
| `403 partner_scope_not_configured` | Nothing wrong with your code. Your key isn't switched on for partner routes yet — email us. |
| `403 unauthorizedPartnerForRequest` | You're scanning another partner's parcel. |
| `400 orderShouldNotBeAcceptedHere` | Wrong `depotId` — you probably sent an address id, or a depot that isn't yours. |
| Agents see red errors on duplicate scans | You're treating `orderAlreadyInThisDepot` as a failure. It isn't. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call.

[← Back to Livra integration guide](README.md)
