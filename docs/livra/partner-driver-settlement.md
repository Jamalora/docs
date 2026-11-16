# Driver Settlement — Partner API

[← Back to Livra integration guide](README.md)

> **Available on staging now, for testing.** Production comes after your tests: we will
> tell you when it is live. Until then, call the staging URL below.

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_driver_settlement`, which records a driver settlement **you already did in
your own tool**.

It does two things, chosen with `action`:

- **`settle`**: a driver hands over the cash they collected.
  - `gross`: the driver hands over **everything**. Their fees stay owed until you pay them.
  - `net`: the driver hands over the cash **minus their fees**. Their fees are then paid too.
- **`pay_fees`**: you pay the driver the fees a `gross` settlement left owed.

When you call it, the settlement lands in our system exactly as if your agent had done it
on the Dentex driver settlement screen:

- the driver's deliveries are marked **settled**,
- the depot payment appears in the depot's payments, with the agent and the type (gross or
  net), linked to the driver's Last Mile sheet when there is one,
- the driver's fee balance is up to date.

**We compute the money, you confirm it.** You send the amount. We compute ours with the
same rules as the Dentex screen. If the two differ, we record **nothing** and send you our
figures line by line, so you can find the difference.

Follow the sections **in order**. Everything you need to copy-paste is here.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The request](#2-the-request)
- [3. How we compute the amount](#3-how-we-compute-the-amount)
- [4. How to build `x-signature`](#4-how-to-build-x-signature)
- [5. Copy-paste examples](#5-copy-paste-examples)
- [6. The success response](#6-the-success-response)
- [7. Every error, and what to do about it](#7-every-error-and-what-to-do-about-it)
- [8. Retries and duplicates](#8-retries-and-duplicates)
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

These are the same key and secret you use for the other partner endpoints
([Merchant Settlement](partner-merchant-settlement.md), [Last Mile](partner-last-mile.md), …).
Your key must be enabled for partner endpoints; if it isn't, every call returns
`403 partner_scope_not_configured`.

> **You do not send your partner id.** We read it from your API key. You can only ever
> settle **your own** drivers.

You also need, for each settlement:

- the **driver** (`driverId`): the `driver.id` we send on our webhooks and in
  [Get Order](partner-get-order.md),
- the **depot** that received the cash (`depotId`), for `settle`,
- the **orders** the settlement covers (`orderIds`),
- the **employee** who took the cash or paid the fees (`employeeId`): the same id we send
  you as `agentId` on our webhooks. Any of your employees can be named.

## 2. The request

- **URL (staging):** `https://staging.livra.mofavo.com/partner_driver_settlement`
- **URL (production, once live):** `https://livra.mofavo.com/partner_driver_settlement`
- **Method:** `POST`

> Staging has its own key and secret (ask `ops@mofavo.com`). Your production key doesn't
> work on staging, and your staging key doesn't work in production. A settlement records
> real data, even on staging.

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

That's all. **There is no `Authorization` header and no bearer token on this endpoint.**

### Body: settle

```json
{
  "action": "settle",
  "driverId": 318,
  "depotId": 12,
  "orderIds": [90211, 90214, 90230],
  "type": "net",
  "amount": 171,
  "employeeId": 482,
  "dryRun": false
}
```

### Body: pay_fees

```json
{
  "action": "pay_fees",
  "driverId": 318,
  "orderIds": [90211, 90214, 90230],
  "amount": 9,
  "employeeId": 482
}
```

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `action` | string | no | `settle` (default) or `pay_fees`. |
| `driverId` | number | yes | One of your **active** drivers. Settle a driver **before** you deactivate them: an inactive driver is `driver_not_found`. |
| `depotId` | number | `settle` | One of your depots: the one that received the cash. Ignored by `pay_fees`. |
| `orderIds` | number[] | yes, unless `dryRun` | 1 to 500 order ids, **no repeats**, that this driver delivered or tried to deliver. Leave it out in a dry run to list **everything open** (below). |
| `type` | string | `settle` | `gross` or `net`. Ignored by `pay_fees`. |
| `amount` | number | yes, unless `dryRun` | `settle`: the cash handed over. `pay_fees`: the fees you paid. Must match ours to the cent. A `net` can be negative (only failed deliveries, with a fee). |
| `employeeId` | number | yes | The employee who took the cash or paid the fees (your `agentId`). Any of your employees. |
| `dryRun` | boolean | no | `true` computes and returns everything, and records **nothing**. Default: `false`. |

- Ids and amounts must be **numbers**, not strings. `"318"` is wrong. `318` is right.
- Extra fields you add are ignored, they don't break anything.
- A known field with the wrong type is rejected, never guessed.
- **One driver per call.**
- No permission is checked on the driver or the employee: you already settled in your own
  tool. They only have to be yours.

### List everything open

Send `dryRun: true` **without** `orderIds`:

- with `action: "settle"`: everything the driver has finished and not settled yet,
  **failed deliveries included**;
- with `action: "pay_fees"`: every fee still owed to the driver after gross settlements.

You get up to 500 orders, newest first. If there are more, the answer has
`"truncated": true` and `openOrderCount`. Then settle by sending those order ids back.

> **Send failed deliveries too.** A refused or unreachable parcel brings in no cash, but
> it still has to be settled (for 0). If you leave it out, it stays "not settled" on the
> Dentex screen.

## 3. How we compute the amount

We make **one line per delivery attempt** of this driver on each order:

| The attempt was… | Cash collected | The driver's fee |
| --- | --- | --- |
| delivered to the customer | the order amount | the delivery fee |
| an exchange that happened | the order amount | the delivery fee |
| failed (refused, unreachable, returned) | 0 | the cancellation fee (0 for most drivers) |

So:

```
gross = cash collected
net   = gross − the driver's fees
```

- **`settle`** uses the driver's fees as they are **now** in Dentex.
- **`pay_fees`** uses, for each delivery, the fee that was in force **when its cash was
  handed over**. Changing a driver's fee later never changes what they are owed.

**Which attempts an order settles.** Every finished attempt **of this driver** on the
order that isn't settled yet. Another driver's attempts on the same order are never
touched. An attempt settled earlier stays as it is: a parcel that failed on Monday
(settled Monday) and was delivered on Tuesday settles only Tuesday's delivery.

**A driver still on the road can't be settled.** If they still have a parcel in transit,
rescheduled or declined, `settle` is refused (`409 driver_has_active_missions`). Paying
fees is never blocked.

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
signature fails with `401 Invalid signature`.

## 5. Copy-paste examples

These call **staging**. Once the endpoint is live, change the host to `livra.mofavo.com`.

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
# Everything driver 318 has open, nothing recorded:
BODY='{"driverId":318,"depotId":12,"type":"net","employeeId":482,"dryRun":true}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://staging.livra.mofavo.com/partner_driver_settlement \
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

export async function driverSettlement(payload) {
  const body = JSON.stringify(payload);
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://staging.livra.mofavo.com/partner_driver_settlement", {
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

// 1. What is open, nothing recorded:
//    const open = await driverSettlement({ driverId: 318, depotId: 12, type: "net",
//                                          employeeId: 482, dryRun: true });
// 2. Count the cash, then record it:
//    const orderIds = [...new Set(open.data.lines.map((l) => l.orderId))];
//    await driverSettlement({ driverId: 318, depotId: 12, orderIds, type: "net",
//                             amount: open.data.totals.amount, employeeId: 482 });
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"

def driver_settlement(payload: dict):
    body = json.dumps(payload, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://staging.livra.mofavo.com/partner_driver_settlement",
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

## 6. The success response

**200**, for a new settlement, a dry run, and a retry of one already recorded.

### settle

```json
{
  "ok": true,
  "action": "settle",
  "dryRun": false,
  "duplicate": false,
  "paymentId": "0f6d2c1e-…",
  "driverId": 318,
  "depotId": 12,
  "type": "net",
  "runsheetId": "b7c1…",
  "totals": { "gross": 180, "fees": 9, "net": 171, "amount": 171 },
  "lines": [
    { "orderId": 90211, "missionId": 771201, "result": "delivered", "collected": 100, "fee": 4,   "net": 96 },
    { "orderId": 90214, "missionId": 771205, "result": "exchange",  "collected": 80,  "fee": 4,   "net": 76 },
    { "orderId": 90230, "missionId": 771150, "result": "failed",    "collected": 0,   "fee": 0.5, "net": -0.5 },
    { "orderId": 90230, "missionId": 771240, "result": "failed",    "collected": 0,   "fee": 0.5, "net": -0.5 }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `paymentId` | Our id for the depot payment. **Store it** with your transaction. `null` on a dry run. |
| `duplicate` | `true` when this exact settlement was already recorded: nothing new was written ([section 8](#8-retries-and-duplicates)). |
| `runsheetId` | The driver's Last Mile sheet the payment was linked to, or `null`. |
| `totals.amount` | The figure for the `type` you sent: `gross` or `net`. |
| `lines` | One per delivery attempt. An order tried twice has two lines. |

### pay_fees

```json
{
  "ok": true,
  "action": "pay_fees",
  "dryRun": false,
  "duplicate": false,
  "paymentId": null,
  "paidAt": "2026-11-16T15:04:05.000Z",
  "driverId": 318,
  "totals": { "fees": 9, "amount": 9 },
  "lines": [
    { "orderId": 90211, "missionId": 771201, "result": "delivered", "fee": 4, "cashSettledAt": "2026-11-10T10:00:00.000Z" }
  ]
}
```

A fee payout leaves **no payment record** on our side (the Dentex screen doesn't record one
either), so `paymentId` is always `null`. **Store `paidAt`** instead. `cashSettledAt` is
when that delivery's cash was handed over: the date its fee is priced at.

## 7. Every error, and what to do about it

Errors look like `{ "ok": false, "error": "<code>", … }`.

| Status | `error` | Also returned | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_payload` | `detail` names the field | Fix the request. |
| 400 | `employee_not_found` | | The `employeeId` is not one of your employees on our side. |
| 400 | `driver_not_found` | | Not one of your active drivers. |
| 400 | `driver_contract_missing` | | `settle`: the driver has no fees set in Dentex. Retry once they do. |
| 400 | `depot_not_found` | | `settle`: not one of your depots. |
| 400 | `orders_not_found` | `orderIds` | This driver never delivered these orders (or they are not yours). Don't retry. |
| 400 | `orders_not_settleable` | `orderIds` | `settle`: this driver's delivery of them is not finished. Retry once it is. |
| 400 | `cash_not_handed_over` | `orderIds` | `pay_fees`: settle the cash of these orders first. |
| 409 | `driver_has_active_missions` | `orderIds` (the active ones) | `settle`: the driver still has parcels on the road. Settle once they are delivered or handed back. |
| 409 | `orders_already_settled` | `orderIds` | `settle`: already settled in another settlement. **Don't retry**; a human looks. |
| 409 | `fees_already_paid` | `orderIds` | `pay_fees`: already paid, by a net settlement or an earlier payout. **Don't retry.** |
| 422 | `amount_mismatch` | `totals`, `lines` (ours) | Our computation differs. Compare line by line, then send the corrected amount. **Nothing was recorded.** |
| 409 | `settlement_in_progress` | | Another settlement of this driver's orders is running. **Retry after a short wait.** |
| 500 | `internal_error` | | Our side. **Retry**; if it keeps failing, send us the `x-request-id`. |
| 401 | `Missing authentication headers` / `Invalid api key` / `Invalid signature` | | Check the headers and [section 4](#4-how-to-build-x-signature). |
| 403 | `partner_scope_not_configured` | | Your key isn't enabled for partner endpoints. Contact us. |

Every response carries an `x-request-id` header. Send it to us with any question.

## 8. Retries and duplicates

**Retrying is always safe.** If a call times out, send **exactly the same request** again
(same orders, same amount, same type and depot):

- if the first call was recorded, the retry answers **200 with `duplicate: true`** (the
  same `paymentId`, or the same `paidAt` for `pay_fees`) and records nothing. That is a
  success, not an error;
- if it wasn't, the retry records it.

Retry on a timeout, `409 settlement_in_progress` and `500`, with a short wait. Never retry
the other errors unchanged: they'll give the same answer.

A settlement of the same orders made on the Dentex screen is recognised the same way.
Naming orders settled in a **different** settlement (only some of them, or another amount,
type or depot) is `409 orders_already_settled`: no cash is ever recorded twice.

## 9. Checklist before you go live

1. On staging, `dryRun: true` without `orderIds` for a driver: check our list and totals
   against your tool.
2. Settle that driver for real on staging, then send the same call again: you get
   `duplicate: true`.
3. Do a `gross` settlement, then `pay_fees` for those orders.
4. Store `paymentId` (settle) and `paidAt` (pay_fees) with your transactions.
5. Alert a human on any `amount_mismatch`, `orders_already_settled` or `fees_already_paid`.
6. Settle each driver in **one** place: your tool or the Dentex screen, not both.

## 10. Common mistakes

| Mistake | What happens | Fix |
| --- | --- | --- |
| Sending only delivered orders | Failed attempts stay "not settled" in Dentex | Use the dry-run list; send every order in it |
| Settling while the driver is still delivering | `409 driver_has_active_missions` | Settle at the end of the round |
| `type` missing on `settle` | `400 invalid_payload` | Send `gross` or `net` |
| `pay_fees` before the cash | `400 cash_not_handed_over` | `settle` with `gross` first |
| Ids as strings (`"318"`) | `400 invalid_payload` | Send numbers |
| Signing a different string than the one sent | `401 Invalid signature` | Build the body once, sign it, send it |
