# Merchant Settlement — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_merchant_settlement`, which records a merchant settlement **you already
paid on your own platform**.

When you call it, the settlement lands in our system exactly as if it had been made on the
Dentex merchant settlement console:

- the payment appears in the merchant's settlement history,
- the 3% retenue à la source is withheld and documented when it applies,
- the orders are marked **paid** everywhere.

**We compute the money, you confirm it.** You send the amount you paid. We compute ours
from the orders, with the same rules as the console. If the two differ, we record
**nothing** and send you our figures line by line, so you can find the difference.

**Paid the merchant anyway?** If you paid a merchant on your platform after we refused
the settlement (for example `amount_mismatch`), tell us which orders you paid with
`"action": "flag"`. We record no payment, but our finance team sees those orders as
**paid on Livra**, with your figures next to ours. See [section 11](#11-you-paid-without-us-flag-and-unflag).

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
- [11. You paid without us: flag and unflag](#11-you-paid-without-us-flag-and-unflag)

---

## 1. What you need before you start

Two secrets from us. Ask `ops@mofavo.com` if you don't have them:

| Thing | Looks like | Where it goes |
| --- | --- | --- |
| API key | an opaque string, e.g. `<apiKey>` | the `x-api-key` header |
| API secret | an opaque string, e.g. `<apiSecret>` | **never sent** — used to compute `x-signature` |

**Never put the secret in a header, a URL, or the body.** It only ever goes into the
HMAC calculation described in [section 4](#4-how-to-build-x-signature).

These are the same key and secret you use for [Accept In Depot](partner-accept-in-depot.md),
[Transfers Between Depots](partner-transfers.md) and [Get Order](partner-get-order.md).
Your key must be enabled for partner endpoints; if it isn't, every call returns
`403 partner_scope_not_configured`.

> **You do not send your partner id.** We read it from your API key. You can only ever
> settle **your own** orders.

You also need, for each settlement:

- the **merchant** you paid (`merchantId`),
- the **orders** the payment covers (`orderIds`),
- the **employee** who made the payment (`employeeId`): the same id we send you as
  `agentId` on our webhooks. Any of your employees can be named.

## 2. The request

- **URL:** `https://livra.mofavo.com/partner_merchant_settlement`
- **Method:** `POST`

> **Test on staging first:** `https://staging.livra.mofavo.com/partner_merchant_settlement`,
> with your staging key. A settlement records a real payment.

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

That's all. **There is no `Authorization` header and no bearer token on this endpoint.**

### Body

```json
{
  "merchantId": 12,
  "orderIds": [101, 102, 103],
  "method": "BANK_WIRE",
  "amount": 159.08,
  "employeeId": 482,
  "dryRun": false
}
```

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `merchantId` | number | yes | Positive whole number. Every order must belong to this merchant. |
| `orderIds` | number[] | yes | 1 to 500 order ids, **no repeats**. Each must be one of your orders, and delivered, returned or exchanged. |
| `method` | string | yes | `CASH` or `BANK_WIRE`. |
| `amount` | number | yes | The **net** you paid the merchant, after fees and retenue. Must match ours to the cent. Can be negative (for example a batch of cancellations only). |
| `employeeId` | number | yes | The employee who made the settlement (your `agentId`). Any of your employees. |
| `dryRun` | boolean | no | `true` computes and returns everything, and records **nothing**. Default: `false`. |
| `action` | string | no | `settle` (default: everything on this page up to section 10), `flag` or `unflag` ([section 11](#11-you-paid-without-us-flag-and-unflag)). Leave it out to settle, as before. |

- Ids and amounts must be **numbers**, not strings. `"12"` is wrong. `12` is right.
- Extra fields you add are ignored, they don't break anything.
- A known field with the wrong type is rejected, never guessed.
- **One merchant per call.** To settle several merchants, make one call each.

## 3. How we compute the amount

For each order we take the fee **in force on the day the order was created**, not today's
price. Renegotiating a contract never changes the price of an order that already exists.

| The order was… | Its fee | What it adds to the merchant's money |
| --- | --- | --- |
| delivered | the delivery fee | order amount − fee |
| exchanged | the exchange fee | order amount − fee |
| cancelled (returned) | the cancellation fee | − fee (nothing was collected) |

**Order sizes (sous-contrats).** A `medium`, `large` or `extra_large` order is billed at
the sous-contrat price for its size on the merchant's contract, again the price in force
when the order was created. If the contract has **no** sous-contrat for that size, we can't
price the order, and you get `400 orders_missing_subcontract`. A standard (`default`)
order never needs one.

**Retenue à la source (3%).** Withheld on the delivered and exchanged orders' net when the
merchant's contract says so, or, if the contract doesn't say, when the merchant has no
matricule fiscal.

So:

```
amount = gross − fees − retenue
```

Not sure? Send the request with `"dryRun": true` first. You get our exact figures, and
nothing is recorded.

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

✅ Right:

```js
const body = JSON.stringify(settlement); // build ONCE
const signature = hmac(body);            // sign the string
fetch(url, { body });                    // send the SAME string
```

❌ Wrong:

```js
const signature = hmac(JSON.stringify(settlement)); // signs one string…
axios.post(url, settlement);                        // …axios serializes a different one
```

## 5. Copy-paste examples

### curl

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"merchantId":12,"orderIds":[101,102,103],"method":"BANK_WIRE","amount":159.08,"employeeId":482,"dryRun":true}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://livra.mofavo.com/partner_merchant_settlement \
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

export async function settleMerchant(settlement) {
  const body = JSON.stringify(settlement);
  const signature = crypto.createHmac("sha256", API_SECRET).update(body, "utf8").digest("hex");

  const res = await fetch("https://livra.mofavo.com/partner_merchant_settlement", {
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

// 1. Our figures, nothing recorded:
//    await settleMerchant({ merchantId: 12, orderIds: [101, 102, 103], method: "BANK_WIRE",
//                           amount: 159.08, employeeId: 482, dryRun: true });
// 2. Pay the merchant, then record it: the same body with dryRun: false.
```

### Python

```python
import hmac, hashlib, json, requests

API_KEY = "your-api-key"
API_SECRET = "your-api-secret"

def settle_merchant(settlement: dict):
    body = json.dumps(settlement, separators=(",", ":"))
    signature = hmac.new(API_SECRET.encode(), body.encode(), hashlib.sha256).hexdigest()

    res = requests.post(
        "https://livra.mofavo.com/partner_merchant_settlement",
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

$body = json_encode([
    'merchantId' => 12,
    'orderIds'   => [101, 102, 103],
    'method'     => 'BANK_WIRE',
    'amount'     => 159.08,
    'employeeId' => 482,
    'dryRun'     => true,
]);
$signature = hash_hmac('sha256', $body, $apiSecret);

$ch = curl_init('https://livra.mofavo.com/partner_merchant_settlement');
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

## 6. The success response

**HTTP 200**, for a new settlement, a dry run, and a retry of a settlement already
recorded:

```json
{
  "ok": true,
  "dryRun": false,
  "duplicate": false,
  "paymentId": "5b0c2f7e-0d3c-4f8e-9c51-2a7f6b1e9d40",
  "merchantId": 12,
  "method": "BANK_WIRE",
  "totals": { "gross": 180, "fees": 16, "withholdingBase": 164, "withholding": 4.92, "amount": 159.08 },
  "lines": [
    { "orderId": 101, "operation": "delivered", "orderAmount": 100, "fee": 7, "net": 93 },
    { "orderId": 102, "operation": "exchange",  "orderAmount": 80,  "fee": 5, "net": 75 },
    { "orderId": 103, "operation": "cancelled", "orderAmount": 0,   "fee": 4, "net": -4 }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `dryRun` | `true` when nothing was recorded because you asked for a dry run. |
| `duplicate` | `true` when this settlement was **already recorded** — see [section 8](#8-retries-and-duplicates). Nothing new was recorded. |
| `paymentId` | Our id for the payment. **Store it** next to your transaction: it's the key for any reconciliation. `null` on a dry run. |
| `method` | As recorded. On a `duplicate`, this is the stored payment's method. |
| `totals.gross` | Order amounts of the delivered and exchanged orders. |
| `totals.fees` | All the fees together. |
| `totals.withholdingBase` | What the 3% retenue is computed on: the sum of `net` over the lines whose contract withholds, **cancelled lines included**, so a return's fee lowers it (example: 93 + 75 − 4 = 164; 3% = 4.92). |
| `totals.withholding` | The retenue withheld, `0` when it doesn't apply. |
| `totals.amount` | What the merchant was paid: `gross − fees − withholding`. |
| `lines[]` | One line per order: its `operation`, `orderAmount`, `fee`, and `net` (what it adds to the merchant's money, before retenue). |

Every response — success or failure — also carries an **`x-request-id`** header.
**Log it.** If you ever open a support ticket, that id is the first thing we ask for.

## 7. Every error, and what to do about it

Failures always look like this:

```json
{ "ok": false, "error": "<code>" }
```

Some errors add more fields (`detail`, `orderIds`, …), listed below. Read the **`error`
string**, not just the HTTP status.

### Authentication problems

| Status | `error` | What went wrong | Fix |
| --- | --- | --- | --- |
| **401** | `Missing authentication headers` | You forgot `x-api-key` or `x-signature`. | Send both headers. |
| **401** | `Invalid api key` | Key is wrong, or disabled. | Check the key. Contact us if it should work. |
| **401** | `Invalid signature` | Your HMAC doesn't match. **99% of the time you signed a different string than you sent.** | Re-read [section 4](#4-how-to-build-x-signature). |
| **403** | `partner_scope_not_configured` | Your key is valid, but it isn't enabled for partner endpoints yet. | Email `ops@mofavo.com`. Nothing you can fix in code. |

### Your request was wrong

| Status | `error` | Also returned | What to do |
| --- | --- | --- | --- |
| **400** | `invalid_payload` | `detail` names the field, e.g. `"orderIds must not contain duplicates"` | Fix the request. |
| **400** | `employee_not_found` | | The `employeeId` isn't one of your employees on our side. Check the id. |

### The orders can't be settled

| Status | `error` | Also returned | What it means | What to do |
| --- | --- | --- | --- | --- |
| **400** | `orders_not_found` | `orderIds` | Not your orders, or they don't exist. | Check the ids. **Don't retry.** |
| **400** | `orders_not_settleable` | `orderIds` | Not delivered, returned or exchanged yet. | Retry once they are. |
| **400** | `orders_of_another_merchant` | `orderIds` | These belong to a different merchant. | One call per merchant. |
| **400** | `orders_missing_subcontract` | `orderIds` | A `medium`, `large` or `extra_large` order whose contract has no sous-contrat for that size, so we can't price it. | Contact us. Retry once the sous-contrat exists. |
| **400** | `orders_with_multiple_open_lines` | `orderIds` | A data problem on our side for these orders. | Contact us. |
| **409** | `orders_already_settled` | `orderIds`, `paymentIds` | Already paid in **another** settlement. `paymentIds` can be empty for orders settled long ago, before payment ids existed. | **Don't retry.** Reconcile with `paymentIds`. |
| **422** | `amount_mismatch` | `totals`, `lines` (ours) | Our computation differs from your `amount`. **Nothing was recorded.** | Compare line by line, then send the corrected settlement. If you pay the merchant your amount anyway, **flag** the orders ([section 11](#11-you-paid-without-us-flag-and-unflag)). |

### Busy, or our side

| Status | `error` | What to do |
| --- | --- | --- |
| **409** | `settlement_in_progress` | Another settlement of these orders is running. **Retry after a short wait.** |
| **500** | `internal_error` | **Retry.** If it keeps failing, send us the `x-request-id`. |

## 8. Retries and duplicates

**Retrying is always safe.** If a call times out, send **exactly the same request** again
(same orders, same amount):

- If the first call was recorded, the retry answers **200 with `duplicate: true`** and the
  same `paymentId`, and records nothing. That's a success, not an error.
- If it wasn't, the retry records it.

**An order is never paid twice.** Naming orders that were paid in a **different**
settlement (only some of them, or with another amount) gets
`409 orders_already_settled`.

Retry **only** on a timeout, `409 settlement_in_progress` and `500`, with a wait between
tries. Every other error gives the same answer the second time.

> A settlement of the same orders made on the Dentex console is also answered with
> `duplicate: true` and the console's payment. If you get a `duplicate` you didn't expect,
> someone may have settled those orders on the console too: check before paying again.

## 9. Checklist before you go live

- [ ] You tested on **staging** first.
- [ ] You call with `"dryRun": true` before paying, and check `totals.amount`.
- [ ] After paying, you send the **same body** with `"dryRun": false` and the amount paid.
- [ ] You **store `paymentId`** with your transaction.
- [ ] The signature is computed over the **exact body string** you send.
- [ ] Ids and amounts are sent as **numbers**, not strings.
- [ ] You are **not** sending a partner id in the body (it comes from your key).
- [ ] On a timeout, `409 settlement_in_progress` or `500`, you retry the **identical** request.
- [ ] Your finance team is alerted on any `amount_mismatch`, `orders_already_settled` or unexpected `duplicate`.
- [ ] You log the `x-request-id` header of every response.
- [ ] Your API secret is in an environment variable, not in the source code.

## 10. Common mistakes

| Symptom | Almost always the cause |
| --- | --- |
| `401 Invalid signature` on every call | Your HTTP library re-serialized the body after you signed it. Send the raw string. In Python use `data=`, not `json=`. |
| `422 amount_mismatch` | Usually a fee: we use the price **on the day each order was created**, and the sous-contrat price for sized orders. Compare your figures with our `lines`. |
| `400 orders_missing_subcontract` | A sized order on a contract with no sous-contrat for that size. Not a code problem — contact us. |
| `409 orders_already_settled` on a retry | The retry wasn't identical: different orders or a different amount. Send exactly the same body. |
| `400 orders_not_settleable` | The order isn't delivered, returned or exchanged yet. Settle it later. |
| `400 invalid_payload` | Read `detail`: it names the field. Often an id sent as a string, or a repeated order id. |
| `403 partner_scope_not_configured` | Nothing wrong with your code. Your key isn't switched on for partner routes yet — email us. |

Still stuck? Email `ops@mofavo.com` with the `x-request-id` of a failing call and, if one
was recorded, the `paymentId`.

## 11. You paid without us: flag and unflag

Sometimes you pay a merchant on your platform even though we refused the settlement,
most often with `422 amount_mismatch`. Then **tell us which orders you paid**, with
`"action": "flag"`, on the same URL, with the same headers and signature.

What a flag does:

- **No payment is recorded in Mofavo.** The orders stay unsettled on our side.
- Our finance team sees them as **paid on Livra, not yet in Mofavo**, with **your figures
  next to ours**, order by order, to find where the two differ.
- They settle those orders later, with our amounts. **You are not asked to pay the
  merchant again** for them.
- The flag is recorded **even when your figures differ from ours**: that's what it's for.

### Flag: the body

```json
{
  "action": "flag",
  "merchantId": 12,
  "employeeId": 482,
  "amount": 158.96,
  "orders": [
    { "orderId": 101, "net": 93, "withholding": 2.79 },
    { "orderId": 102, "net": 75, "withholding": 2.25 },
    { "orderId": 103, "net": -4, "withholding": 0 }
  ],
  "dryRun": false
}
```

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `action` | string | yes | `flag`. |
| `merchantId` | number | yes | The merchant you paid. Every order must belong to it. |
| `employeeId` | number | yes | The employee who made the payment (your `agentId`). |
| `amount` | number | yes | The total you paid the merchant. **Must equal the sum of `net` minus the sum of `withholding`**, to the cent, or you get `400 invalid_payload`. |
| `orders` | object[] | yes | 1 to 500 orders, **no repeats**. |
| `orders[].orderId` | number | yes | One of your orders, delivered, returned or exchanged, **not yet settled** in Mofavo. |
| `orders[].net` | number | yes | Your net for this order, **before** the 3%: what it adds to the merchant's money (negative for a return). Same meaning as our `lines[].net`. |
| `orders[].withholding` | number | no | The 3% you withheld on this order. `0` or left out when none. |
| `dryRun` | boolean | no | `true` returns the comparison and records **nothing**. |

There is no `method` and no `orderIds` field on a flag: the orders are in `orders`.

### Flag: the response

**HTTP 200**, with your figures and ours:

```json
{
  "ok": true,
  "action": "flag",
  "dryRun": false,
  "duplicate": false,
  "merchantId": 12,
  "totals": { "livra": 158.96, "ours": 159.08, "difference": -0.12 },
  "lines": [
    { "orderId": 101, "livra": { "net": 93, "withholding": 2.79 }, "ours": { "net": 93, "withholding": 2.79 }, "difference": 0 },
    { "orderId": 102, "livra": { "net": 75, "withholding": 2.25 }, "ours": { "net": 75, "withholding": 2.25 }, "difference": 0 },
    { "orderId": 103, "livra": { "net": -4, "withholding": 0 }, "ours": { "net": -4, "withholding": -0.12 }, "difference": -0.12 }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `totals.livra` | Your `amount`. |
| `totals.ours` | What we would pay for the same orders (the `totals.amount` of a settlement). `null` if we can't price one of them. |
| `totals.difference` | `livra − ours`. |
| `lines[].ours` | Our net and our share of the 3% for that order. `null` when we can't price it, with `ourError` saying why (for example `orders_missing_subcontract`). The order is **still flagged**. |
| `lines[].difference` | Your `net − withholding` minus ours, for that order. |
| `duplicate` | `true` when these orders were already flagged with exactly the same figures: nothing new was recorded. |

In this example, the difference is the return (order 103): its fee lowers the 3% base, so
our share of the 3% on it is −0.12.

### Unflag

Flagged an order by mistake? Remove the flag:

```json
{
  "action": "unflag",
  "merchantId": 12,
  "employeeId": 482,
  "orderIds": [103],
  "reason": "added by mistake"
}
```

| Field | Type | Required | Rules |
| --- | --- | --- | --- |
| `action` | string | yes | `unflag`. |
| `merchantId` | number | yes | The merchant of the orders. |
| `employeeId` | number | yes | Who removes the flag. |
| `orderIds` | number[] | yes | 1 to 500 flagged orders, no repeats. |
| `reason` | string | no | Why, up to 500 characters. |

Response, **HTTP 200**:

```json
{ "ok": true, "action": "unflag", "merchantId": 12, "orderIds": [103] }
```

To **change** the figures of a flagged order, unflag it, then flag it again.

### Settling flagged orders

A normal settlement (no `action`) of flagged orders is **allowed**: if you and we later
agree on the amount, settle them as usual. They become paid in Mofavo and are no longer
shown as flagged.

### Flag and unflag errors

Authentication errors, `invalid_payload`, `employee_not_found`, `settlement_in_progress`
and `internal_error` work exactly as in [section 7](#7-every-error-and-what-to-do-about-it).
Also:

| Status | `error` | Also returned | What it means | What to do |
| --- | --- | --- | --- | --- |
| **400** | `orders_not_found` | `orderIds` | Not your orders, or they don't exist. | Check the ids. |
| **400** | `orders_of_another_merchant` | `orderIds` | These belong to a different merchant. | One call per merchant. |
| **400** | `orders_not_settleable` | `orderIds` | Not delivered, returned or exchanged yet. | Flag them once they are. |
| **400** | `orders_with_multiple_open_lines` | `orderIds` | A data problem on our side for these orders. | Contact us. |
| **409** | `orders_already_settled` | `orderIds` | Already paid in Mofavo: there is nothing to flag (or unflag). | **Don't retry.** |
| **409** | `orders_already_flagged` | `orderIds` | Already flagged, with different figures or in another call. Nothing was recorded. | Unflag them first if the figures changed. |
| **409** | `orders_not_flagged` | `orderIds` | Unflag of orders that aren't flagged. Nothing was changed. | Check the ids. |

A flag or unflag is **all or nothing**: if one order is refused, nothing is recorded for
the others. Retrying an identical flag is safe (`duplicate: true`).

[← Back to Livra integration guide](README.md)
