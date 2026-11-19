# Validation — Partner API

[← Back to Livra integration guide](README.md)

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_validation`, which decides **where your orders go**, and takes back the
parcels a driver brings back to the depot.

It replaces three actions of [Last Mile](partner-last-mile.md): `return`, `retry` (and
`add` with `"retry": true` for the destination part), and the way back to the merchant.
[Last Mile](partner-last-mile.md) keeps the sheet itself: `create`, `add`, `remove`,
`collect`.

Delivering is **not** here: it is a mission status, sent with the mission status API.

## Contents

- [1. The two actions](#1-the-two-actions)
- [2. The request](#2-the-request)
- [3. Rules](#3-rules)
- [4. Copy-paste example](#4-copy-paste-example)
- [5. The response](#5-the-response)
- [6. Every error, and what to do about it](#6-every-error-and-what-to-do-about-it)
- [7. Retries: every call is safe to send again](#7-retries-every-call-is-safe-to-send-again)
- [8. Moving from Last Mile](#8-moving-from-last-mile)

## 1. The two actions

| Action | What it does |
| --- | --- |
| `destination` | For each order: `"cancelled": true` cancels the delivery, the order goes **back to the merchant**. `"cancelled": false` sends it **to the customer again** (a retry). Only the destination changes. Works while the order is in a depot **or on the road**. |
| `return` | The driver of a collected sheet is back with parcels he couldn't deliver. Per order, `"cancelled": true` (going back to the merchant) or `false` (another attempt). Exactly the [Last Mile `return`](partner-last-mile.md#return): same body, same result. |

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
{
  "action": "destination",
  "employeeId": 7,
  "orders": [
    { "orderId": 123456, "cancelled": true },
    { "orderId": 123457, "cancelled": false }
  ]
}
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
| `action` | string | Required. `"destination"` or `"return"`. |
| `employeeId` | number | Required. One of your employees: the person who did it. |
| `orders` | list | Required. 1 to 500 `{ "orderId", "cancelled" }`, each order once. `cancelled` is required, `true` or `false`. |
| `reference` | string | `return` only, required: your delivery sheet, as in [Last Mile](partner-last-mile.md). Refused on `destination`. |

Ids are **numbers**, not strings. **Unknown fields are refused** with `400 invalid_payload`,
and `details.field` names the field.

## 3. Rules

- **`"cancelled": true`** works like our "return to merchant" button: the order now goes to
  the merchant. For an exchange, its last mission is marked as declined. Its open quality
  checks close. **A driver can no longer deliver it**: if the driver still has it, send
  `return` when he is back.
- **`"cancelled": false`** works like our "Retry" button: the order goes to the customer
  again. For an exchange, its missions go back to "exchange in progress". Then put it on a
  sheet with [Last Mile `add`](partner-last-mile.md#add).
- `destination` changes **only where the order goes**: not its status, not its sheet, not
  its missions. A parcel on a transfer keeps going to that depot.
- It is **accepted** for an order in a depot (`inDepot`) or on the road (`inTransit`).
- It is **refused** for an order cancelled, returned, delivered (an exchange already
  swapped counts), not picked up from the merchant yet, or in any other status (e.g.
  missing).
- An order **already going where you say**: `ok`, nothing is written.
- You can mix `true` and `false` in one call.

## 4. Copy-paste example

```bash
API_KEY="your-api-key"
API_SECRET="your-api-secret"
BODY='{"action":"destination","employeeId":7,"orders":[{"orderId":123456,"cancelled":true}]}'

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
  { "orderId": 123457, "error": "not_picked_up", "message": "This order has not been picked up from the merchant yet; to cancel it, use update_order with action cancel" } ] }
```

| Action | Order status | Timeline |
| --- | --- | --- |
| `destination`, `cancelled: true` | unchanged, now going to the merchant | "scheduled for return to merchant" |
| `destination`, `cancelled: false` | unchanged, now going to the customer | "retry" |
| `return` | as [Last Mile `return`](partner-last-mile.md#return) | as Last Mile `return` |

Each row shows "(via partner API)". These changes are **not** sent to your webhook feed:
you made them. The merchant's own webhooks still fire.

## 6. Every error, and what to do about it

Whole call refused, nothing applied: `{ "ok": false, "error": "...", "message": "..." }`.

| Status | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body is not JSON. | Fix the request. |
| **400** | `invalid_payload` | A field is missing, wrong or unknown (e.g. `"action": "deliver"`); `details.field` names it. | Fix the request. |
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
| `order_not_found` | both | No order with this id. |
| `unauthorized_partner` | both | The order belongs to another delivery partner. |
| `order_cancelled` | destination | The order is cancelled. |
| `already_delivered` | both | Already delivered (an exchange already swapped too). |
| `already_returned` | both | Already returned to the merchant. |
| `not_picked_up` | destination | Not picked up from the merchant yet. To cancel it, use Update Order with `"action": "cancel"`. |
| `invalid_status` | destination | Not in a depot or on the road (e.g. missing). |
| `not_on_this_runsheet` | return | Not on this sheet, or already returned from it. |
| `not_collected` | return | Collect left it in the depot. Use Last Mile `remove`. |
| `busy` | both | Another call on this order is in flight. Send it again. |
| `internal_error` | both | Our error on this order. Send it again later. |

## 7. Retries: every call is safe to send again

An order already going where you say, or already returned from the sheet, counts as `ok`
and is not written twice. A call cut short (timeout) is finished by sending it again.

## 8. Moving from Last Mile

| You send today on `partner_last_mile` | Send instead on `partner_validation` |
| --- | --- |
| `return` | `return`, same body |
| `retry` with `orderIds` | `destination` with `orders: [{ "orderId", "cancelled": false }]` |
| `add` with `"retry": true` | `destination` with `"cancelled": false`, then Last Mile `add` |
| `deliver` | the mission status API (`delivered`) |

The old actions still work, the same way, until you have switched. `deliver` refuses an
order going back to the merchant (`not_for_customer`).
