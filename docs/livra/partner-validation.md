# Validation — Partner API

[← Back to Livra integration guide](README.md)

> **Staging only for now.** This endpoint is live on `https://staging.livra.mofavo.com/partner_validation`
> for testing. It is not in production yet: until then, keep using [Last Mile](partner-last-mile.md)
> `return` and `retry` in production.

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_validation`, which decides **where your orders go**, and takes back the
parcels a driver brings back to the depot.

It replaces three actions of [Last Mile](partner-last-mile.md): `return`, `retry` (and
`add` with `"retry": true` for the destination part), and the way back to the merchant.
[Last Mile](partner-last-mile.md) keeps the sheet itself: `create`, `add`, `remove`,
`collect`.

Delivering is **not** here: it is a driver result, sent with [Mission Status](partner-mission-status.md) (`delivered`).

## Last Mile and Validation together

The two endpoints cover one driver's round. [Last Mile](partner-last-mile.md) builds the sheet and records
the driver leaving; [Validation](partner-validation.md) decides where each order goes and takes back what
the driver brings back.

| When | Send |
| --- | --- |
| You load parcels for a driver in your depot | Last Mile `create`, `add`, `remove` |
| The driver leaves with them | Last Mile `collect` |
| The driver still has a parcel and you decide, after calling the customer, that it goes back to the merchant, or to the customer after all | Validation `return_to_merchant` / `retry` |
| The driver can't deliver a parcel (unreachable, postponed, refused…) | [Mission Status](partner-mission-status.md), right away |
| The driver is back with parcels he couldn't deliver | Validation `return` (after their result) |
| A parcel is back in the depot and you decide again | Validation `retry` (then Last Mile `add` on a new sheet) or `return_to_merchant` (then [Returns](partner-return.md)) |
| A parcel is delivered | [Mission Status](partner-mission-status.md) |

Same key, same signing, same `employeeId` and order ids on both.

## Contents

- [Last Mile and Validation together](#last-mile-and-validation-together)
- [1. The three actions](#1-the-three-actions)
- [2. The request](#2-the-request)
- [3. Rules](#3-rules)
- [4. Copy-paste example](#4-copy-paste-example)
- [5. The response](#5-the-response)
- [6. Every error, and what to do about it](#6-every-error-and-what-to-do-about-it)
- [7. Retries: every call is safe to send again](#7-retries-every-call-is-safe-to-send-again)
- [8. Moving from Last Mile](#8-moving-from-last-mile)

## 1. The three actions

| Action | What it does |
| --- | --- |
| `retry` | The orders go **to the customer again**. |
| `return_to_merchant` | The orders go **back to the merchant**. |
| `return` | The driver of a collected sheet is back with parcels he couldn't deliver, once their result was sent ([Mission Status](partner-mission-status.md)). The result decides where each goes (refused → merchant); `cancelled` is our depot's scan button: `false` for an order going to the customer (a refused one included), `true` for one already going to the merchant. Exactly the [Last Mile `return`](partner-last-mile.md#return): same body, same rules. |

`retry` and `return_to_merchant` do, for each order, what our own staff do **for where the
parcel is**: a quality-control agent while the driver still has it, a depot employee once
it is back in the depot. You don't say which: we see where the parcel is.

## 2. The request

- **URL (production):** `https://livra.mofavo.com/partner_validation`
- **URL (staging):** `https://staging.livra.mofavo.com/partner_validation`
- **Method:** `POST`

Build and test against **staging** first, with your staging key and secret.

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <lowercase hex HMAC-SHA256 of the raw body, with your api secret>
```

The signature is built exactly as for Last Mile: see
[How to build `x-signature`](partner-last-mile.md#4-how-to-build-x-signature).

### Body

```json
{ "action": "retry", "employeeId": 7, "orderIds": [123457] }
```

```json
{ "action": "return_to_merchant", "employeeId": 7, "orderIds": [123456, 123460] }
```

```json
{
  "action": "return",
  "reference": "LM-2026-000981",
  "employeeId": 7,
  "orders": [
    { "orderId": 123458, "cancelled": false },
    { "orderId": 123459, "cancelled": true }
  ]
}
```

| Field | Type | Rules |
| --- | --- | --- |
| `action` | string | Required. `"retry"`, `"return_to_merchant"` or `"return"`. |
| `employeeId` | number | Required. One of your employees: the person who did it. |
| `orderIds` | list | `retry` and `return_to_merchant`, required. 1 to 500 order ids, each once. |
| `orders` | list | `return` only, required. 1 to 500 `{ "orderId", "cancelled" }`, each order once. `cancelled`: `false` = going to the customer (a refused parcel included), `true` = already going to the merchant. See [Last Mile `return`](partner-last-mile.md#return). |
| `reference` | string | `return` only, required: your delivery sheet, as in [Last Mile](partner-last-mile.md). |

Ids are **numbers**, not strings. **Unknown fields are refused** with `400 invalid_payload`,
and `details.field` names the field.

## 3. Rules

| Where the order is | `retry` | `return_to_merchant` |
| --- | --- | --- |
| **With the driver**: on a collected sheet (delivery still out, redated or declined) | Like our quality-control "Retry": it goes to the customer again; a declined delivery is back in progress with the driver. | Like our quality-control "Cancel": it goes back to the merchant; the delivery is marked declined, so **the driver can no longer deliver it**. When he is back, send `return`. |
| **In the depot**, nothing open on it | Like our inbound "Retry": it goes to the customer again. Then put it on a sheet with [Last Mile `add`](partner-last-mile.md#add). | Like our "Return to merchant" button: it goes back to the merchant. |
| In the depot on a sheet **not collected yet**, or on a **transfer** | Refused: `has_open_mission`. | Refused: `has_open_mission`. |

- **Exchanges:** `return_to_merchant` marks the exchange as declined; `retry` puts its
  missions back to "exchange in progress".
- `return_to_merchant` closes the order's open quality checks.
- It is **refused** for an order cancelled, returned, delivered (an exchange already
  swapped counts), not picked up from the merchant yet, or in any other status (e.g.
  missing).
- An order **already done** (already going there, nothing left to change): `ok`, nothing
  is written.

## 4. Copy-paste example

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"return_to_merchant","employeeId":7,"orderIds":[123456]}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://staging.livra.mofavo.com/partner_validation \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

## 5. The response

Everything applied, **200**:

```json
{ "ok": true }
```

Some orders refused, **200**. The others **were applied**:

```json
{ "ok": false, "errors": [
  { "orderId": 123460, "error": "has_open_mission", "message": "This order has an open mission (on a sheet not collected yet, or on a transfer); remove it from the sheet or wait for it to arrive" } ] }
```

| Action | Where | Order status | Timeline |
| --- | --- | --- | --- |
| `retry` | with the driver | `inTransit`, now going to the customer | "retry" |
| `retry` | in the depot | `inDepot`, now going to the customer | "retry" |
| `return_to_merchant` | with the driver | `inTransit`, now going to the merchant | "declined" |
| `return_to_merchant` | in the depot | `inDepot`, now going to the merchant | "scheduled for return to merchant" |
| `return` | | as [Last Mile `return`](partner-last-mile.md#return) | as Last Mile `return` |

Each row shows "(via partner API)". These changes are **not** sent to your webhook feed:
you made them. The merchant's own webhooks still fire.

## 6. Every error, and what to do about it

Whole call refused, nothing applied: `{ "ok": false, "error": "...", "message": "..." }`.

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body is not JSON. | Fix the request. |
| **400** | `invalid_payload` | A field is missing, wrong or unknown (e.g. `"action": "destination"`); `details.field` names it. | Fix the request. |
| **401** | `missing_headers` / `invalid_api_key` / `invalid_signature` | Authentication failed. | Check your key and signing. |
| **403** | `partner_scope_not_configured` | Your key is not enabled for partner routes. | Contact us. |
| **400** | `employee_not_found` | `employeeId` isn't one of your employees. | Check your employee mapping. |
| **404** | `runsheet_not_found` | `return`: no sheet with this reference. | Check the reference. |
| **409** | `runsheet_closed` | `return`: the sheet is finished. | Don't retry. |
| **409** | `not_collected` | `return`: the sheet isn't collected yet. | Use Last Mile `remove`. |
| **409** | `busy` | Another call on this sheet is in flight. | Retry after a short pause. |
| **500** | `internal_error` | Our side; nothing applied. | Retry once, then contact us with the `x-request-id`. |

Per order, in `errors`:

| `error` | Action | What it means → what to do |
| --- | --- | --- |
| `order_not_found` | all | No order with this id. |
| `unauthorized_partner` | all | The order belongs to another delivery partner. |
| `order_cancelled` | retry, return_to_merchant | The order is cancelled. |
| `already_delivered` | all | Already delivered (an exchange already swapped too). |
| `already_returned` | all | Already returned to the merchant. |
| `not_picked_up` | retry, return_to_merchant | Not picked up from the merchant yet. To cancel it, use Update Order with `"action": "cancel"`. |
| `has_open_mission` | retry, return_to_merchant | On a sheet not collected yet, or on a transfer or pick-up. Remove it from the sheet (Last Mile `remove`), or wait for it to arrive. |
| `invalid_status` | retry, return_to_merchant | Not in a depot or with a driver (e.g. missing). |
| `not_on_this_runsheet` | return | Not on this sheet, or already returned from it. |
| `not_collected` | return | Collect left it in the depot. Use Last Mile `remove`. |
| `still_in_delivery` | return | Still out for delivery: no driver result was sent. Send it ([Mission Status](partner-mission-status.md)) first. |
| `should_be_return` | return | Goes to the customer (a refused one too): send it with `"cancelled": false`. |
| `should_be_merchant` | return | Already goes to the merchant: send it with `"cancelled": true`. |
| `busy` | all | Another call on this order is in flight. Send it again. |
| `internal_error` | all | Our error on this order. Send it again later. |

## 7. Retries: every call is safe to send again

An order already done, or already returned from the sheet, counts as `ok`
and is not written twice. A call cut short (timeout) is finished by sending it again.

## 8. Moving from Last Mile

| You send today on `partner_last_mile` | Send instead on `partner_validation` |
| --- | --- |
| `return` | `return`, same body |
| `retry` with `orderIds` | `retry`, same `orderIds` |
| `add` with `"retry": true` | `retry`, then Last Mile `add` |
| `deliver` | [Mission Status](partner-mission-status.md) (`delivered`) |

The old actions still work until you have switched; Last Mile `retry` now follows the same
rules as `retry` here. `deliver` refuses an
order going back to the merchant (`not_for_customer`).
