# Drivers and Employees — Partner API

[← Back to Livra integration guide](README.md)

> **Available on staging now, for testing.** Production comes after your tests: we will
> tell you when it is live. Until then, call the staging URL below.

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_staff`, which keeps **your drivers and employees** in sync with ours.

The other partner endpoints ask for **our** ids: `driverId` on
[Transfers](partner-transfers.md), [Returns](partner-return.md) and
[Last Mile](partner-last-mile.md), and `employeeId` almost everywhere. With this
endpoint, each time you create a driver or an employee on your side you create them on
ours in the same moment and get our id back. No more copying ids by hand.

Each person is found by **your own `externalId`**: your id for them, which we keep on our
driver or employee.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. Two actions](#2-two-actions)
- [3. The request](#3-the-request)
  - [`create`](#create)
  - [`link`](#link)
- [4. How to build `x-signature`](#4-how-to-build-x-signature)
- [5. Copy-paste examples](#5-copy-paste-examples)
- [6. The response](#6-the-response)
- [7. Every error, and what to do about it](#7-every-error-and-what-to-do-about-it)
- [8. Retries: every call is safe to send again](#8-retries-every-call-is-safe-to-send-again)
- [9. Checklist before you go live](#9-checklist-before-you-go-live)

---

## 1. What you need before you start

The same API key and secret you use for the other partner endpoints
([Accept In Depot](partner-accept-in-depot.md), [Transfers](partner-transfers.md), …).
Ask `ops@mofavo.com` if you don't have them. Your key must be enabled for partner
endpoints; if it isn't, every call returns `403 partner_scope_not_configured`.

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 4](#4-how-to-build-x-signature).

> **You do not send your partner id.** We read it from your API key. You can only ever
> see and change **your own** drivers and employees.

## 2. Two actions

Both go to the same URL. The `action` field says which one it is.

1. **`create`** — add a driver or an employee. We answer **our id** for them: store it
   next to your `externalId`.
2. **`link`** — for people who **already exist** on our side (created before this
   endpoint): put your `externalId` on our `driverId` or `employeeId`. Do this once for
   each existing person.

People created through this endpoint get **no login** to our apps. They exist so that
your calls can name them. This endpoint does not change whether someone is active, nor
their permissions.

## 3. The request

- **URL (staging):** `https://staging.livra.mofavo.com/partner_staff`
- **URL (production, once live):** `https://livra.mofavo.com/partner_staff`
- **Method:** `POST`

Build and test against **staging** first. It works the same way, with its own data and
its own API key and secret.

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

### Fields every action needs

| Field | Type | Rules |
| --- | --- | --- |
| `action` | string | Required. `"create"` or `"link"`. |
| `role` | string | Required. `"driver"` or `"employee"`. |
| `externalId` | string | Required. **Your** id for the person: 1 to 64 characters (spaces at the ends are trimmed), no control characters. Unique among your drivers, and among your employees (a driver and an employee may share one). |

Extra fields you add are ignored. A known field with the wrong type is a `400`.

### `create`

```json
{
  "action": "create",
  "role": "driver",
  "externalId": "LIV-DRV-0042",
  "firstName": "Ali",
  "lastName": "Ben Salah",
  "phone": "22123456",
  "email": "ali.bensalah@livra.tn"
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `firstName`, `lastName` | string | Required. 1 to 80 characters. |
| `phone` | string | Required. 8 to 20 digits or spaces, optional leading `+`. |
| `email` | string | Required. Must not already belong to another account (see `email_taken`). |

The same fields for a driver and an employee.

If the email belongs to someone who is **already yours in the other role** (for example a
driver you now create as an employee), the same person gets the new role: no second
account.

Name, phone and email are set **once**, at `create`: there is no update. To change them,
contact us.

### `link`

```json
{ "action": "link", "role": "employee", "externalId": "LIV-EMP-0003", "employeeId": 7 }
```

Send `driverId` with `role: "driver"`, `employeeId` with `role: "employee"`. It must be
one of **your** drivers or employees, without an `externalId` yet. Sending the same link
again is fine.

## 4. How to build `x-signature`

Exactly as on the other partner endpoints: an HMAC-SHA256 of the **request body**, keyed
with your **API secret**, written as lowercase hex.

```
x-signature = HMAC_SHA256( raw request body , apiSecret )  →  lowercase hex
```

**Sign the exact bytes you send.** Build the JSON string **once**, sign that string, and
send that same string as the body. See [Returns, section 4](partner-return.md#4-how-to-build-x-signature)
for the right and wrong way side by side.

## 5. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"create","role":"employee","externalId":"LIV-EMP-0007","firstName":"Sarra","lastName":"Ben Ali","phone":"51111111","email":"sarra@livra.tn"}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://staging.livra.mofavo.com/partner_staff \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

### Node.js

```js
import crypto from "node:crypto";

async function callStaff(payload) {
  const body = JSON.stringify(payload);
  const signature = crypto.createHmac("sha256", process.env.API_SECRET).update(body, "utf8").digest("hex");
  const res = await fetch("https://staging.livra.mofavo.com/partner_staff", {
    method: "POST",
    headers: { "content-type": "application/json", "x-api-key": process.env.API_KEY, "x-signature": signature },
    body, // the SAME string that was signed
  });
  return { status: res.status, requestId: res.headers.get("x-request-id"), data: await res.json() };
}

const { data } = await callStaff({
  action: "create", role: "driver", externalId: "LIV-DRV-0042",
  firstName: "Ali", lastName: "Ben Salah", phone: "22123456", email: "ali.bensalah@livra.tn",
});
// Store data.driverId next to LIV-DRV-0042, and send it as driverId on the other endpoints.
```

### Python

```python
import hmac, hashlib, json, requests

def call_staff(payload: dict):
    body = json.dumps(payload, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()
    res = requests.post(
        "https://staging.livra.mofavo.com/partner_staff",
        headers={"Content-Type": "application/json", "x-api-key": API_KEY, "x-signature": signature},
        data=body,  # data=, NOT json= — json= would re-serialize and break the signature
        timeout=30,
    )
    return res.status_code, res.headers.get("x-request-id"), res.json()

call_staff({"action": "link", "role": "employee", "externalId": "LIV-EMP-0003", "employeeId": 7})
```

## 6. The response

```json
{ "ok": true, "role": "driver", "externalId": "LIV-DRV-0042", "driverId": 512, "created": true }
```

| Field | Meaning |
| --- | --- |
| `driverId` / `employeeId` | **Our id** for the person. Send it on the other partner endpoints. |
| `created` | `create` only. `true`: we created the person (**201**). `false`: this `externalId` already existed; nothing was written and you get the same id (**200**). |

Every other success is **200**.

## 7. Every error, and what to do about it

```json
{ "ok": false, "error": "email_taken", "message": "This email is already one of your drivers; link it with action \"link\"", "driverId": 42 }
```

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body is not JSON. | Fix the request. |
| **400** | `invalid_payload` | A field is missing or wrong; `message` names it. | Fix the request. |
| **401** | `missing_headers` / `invalid_api_key` / `invalid_signature` | Authentication failed. | Check your key and [signing](#4-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Key not enabled for partner endpoints. | Contact us. |
| **404** | `staff_not_found` | `link`: the id is not one of your drivers / employees. | Check the id. |
| **409** | `email_taken` | The email already has an account. If it is already one of your people in this role, the answer carries its `driverId` / `employeeId`. | Same person: `link` that id. Otherwise use another email, or contact us. |
| **409** | `already_linked` | `link`: the person already has another `externalId`, or your `externalId` is on another person. | Check your mapping. |
| **500** | `internal_error` | Our side; nothing was written. | Retry once, then contact us with the `x-request-id`. |

## 8. Retries: every call is safe to send again

- A `create` with an `externalId` that exists answers **the same id** (`created: false`).
- A `link` already made answers **200** and changes nothing.

So on a timeout, just send the same call again.

## 9. Checklist before you go live

- [ ] On staging: create a driver and an employee, and store the ids you get back.
- [ ] Use those ids as `driverId` / `employeeId` on another partner endpoint.
- [ ] `link` every driver and employee that already existed before this endpoint.
- [ ] Log the `x-request-id` of every call: we need it to help you.
