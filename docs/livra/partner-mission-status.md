# Mission Status — Partner API

[← Back to Livra integration guide](README.md)

> **Staging only for now.** Live on `https://staging.livra.mofavo.com/partner_mission_status`
> for you to build and test against. It is not in production yet: we will tell you when it is.

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_mission_status`: what your **driver records at each customer**, sent as it
happens. Each result does exactly what the same button does in our own driver app: the
delivery and order status, the timeline entry, the customer SMS or the quality-control
check, the merchant's webhooks.

It is one of three APIs for a delivery round. **Read [Delivery Round](delivery-round.md)
first**: it shows when to call [Last Mile](partner-last-mile.md) (the sheet), Mission Status
(the driver's results) and [Validation](partner-validation.md) (where a parcel goes, and the
driver coming back).

## Contents

- [1. When to call it](#1-when-to-call-it)
- [2. The request](#2-the-request)
- [3. The statuses, one by one](#3-the-statuses-one-by-one)
- [4. Rules](#4-rules)
- [5. Offline driver app and retries](#5-offline-driver-app-and-retries)
- [6. The response](#6-the-response)
- [7. Every error, and what to do about it](#7-every-error-and-what-to-do-about-it)
- [8. Copy-paste example](#8-copy-paste-example)
- [9. Checklist before you go live](#9-checklist-before-you-go-live)

## 1. When to call it

**Every time your driver records something at a customer**, on a sheet you already
collected with [Last Mile `collect`](partner-last-mile.md#collect):

| The driver… | Send |
| --- | --- |
| handed the parcel over | `delivered` |
| couldn't reach the customer | `unreachable` |
| agreed a new day with the customer | `postponed` with `date` |
| wants to leave a comment | `postponed` without `date` |
| was refused the parcel | `declined` |
| has a wrong phone number | `wrong_phone` |
| can't find the parcel | `missing` |
| made a mistake and wants to start over | `reset` |

What comes next for each one (a new sheet, a Validation decision, the return to the depot)
is in [Delivery Round, §3](delivery-round.md#3-after-each-driver-result-what-comes-next), and
every action of the three APIs is in [Every action at a glance](delivery-round.md#every-action-at-a-glance).

Send it **right away**. If the app is offline, send the results when it's back, each with
the time it happened (`at`). You can send up to 100 results in one call, for any parcels of
the same sheet.

## 2. The request

- **URL (staging):** `https://staging.livra.mofavo.com/partner_mission_status`
- **URL (production, when live):** `https://livra.mofavo.com/partner_mission_status`
- **Method:** `POST`
- **Headers:** `Content-Type: application/json`, `x-api-key`, `x-signature`: exactly as
  [Last Mile](partner-last-mile.md#4-how-to-build-x-signature) (sign the exact body you send).

```json
{
  "reference": "LM-2026-000981",
  "driverId": 42,
  "results": [
    { "orderId": 123456, "status": "unreachable", "at": "2026-11-20T09:10:00+01:00" },
    { "orderId": 123457, "status": "postponed", "date": "2026-11-22", "note": "At work, Saturday" },
    { "orderId": 123458, "status": "declined", "reason": "cancelByClient" },
    { "orderId": 123459, "status": "delivered" }
  ]
}
```

| Field | Required | What it is |
| --- | --- | --- |
| `reference` | yes | Your delivery sheet, as in [Last Mile](partner-last-mile.md). It must be collected. |
| `driverId` | no | Our id for the driver. When sent, it must be the sheet's driver (else `409 driver_mismatch`). |
| `employeeId` | no | One of your employees, when someone records it **for** the driver (shown on the timeline). |
| `results` | yes | 1 to 100 results. |
| `results[].orderId` | yes | Our order id (a number). |
| `results[].status` | yes | One of the statuses in [section 3](#3-the-statuses-one-by-one). |
| `results[].at` | no | When the driver recorded it: ISO 8601 with an offset, e.g. `2026-11-20T09:10:00+01:00`. Default: now. At most 48 hours ago, not in the future, and not before the driver collected the parcel. |
| `results[].date` | `postponed` only | `YYYY-MM-DD`, today or later (Tunis time): the new delivery day. Without it, `postponed` is a comment. |
| `results[].reason` | `declined` only | `cancelByClient` (default), `tries` or `other`. |
| `results[].note` | no (yes with `reason: "other"`) | The driver's words, at most 500 characters. Shown on the timeline. |

Ids are **numbers**. **Unknown fields are refused** (`400 invalid_payload`, `details.field`
names it).

## 3. The statuses, one by one

Each status is a button of our driver app, and does what that button does.

### `delivered`

```json
{ "orderId": 123459, "status": "delivered", "at": "2026-11-20T16:02:00+01:00" }
```

The parcel is **delivered**, with `at` as its delivery date, and counts in the merchant's
settlement. Allowed even after an earlier unreachable, postponed or refused result (the
customer took it in the end). **Refused (`not_for_customer`) if the parcel is going back to
the merchant**: send [Validation `retry`](partner-validation.md) first.

**Exchange orders:** `delivered` means the parcels were swapped at the door. The order stays
`inTransit`: the driver now carries the old item, going back to the merchant. When he's back,
send [Validation `return`](partner-validation.md) with `"cancelled": true`.

### `unreachable`

```json
{ "orderId": 123456, "status": "unreachable" }
```

The driver keeps the parcel; it's flagged for attention. The customer gets an SMS, or a
check opens for your customer service (the contract decides).

### `postponed`

```json
{ "orderId": 123457, "status": "postponed", "date": "2026-11-22", "note": "At work, Saturday" }
```

With a `date`: the parcel is **planned for that day** (put it on that day's sheet once it's
back). SMS or check, as for unreachable. **Without** a `date`: a comment on the timeline;
nothing else changes.

### `declined`

```json
{ "orderId": 123458, "status": "declined", "reason": "other", "note": "Box damaged, refused" }
```

The customer refused it. The driver keeps it; **its destination doesn't change yet**. The
merchant is asked what to do (a check, or the contract's SMS). Then:

- your customer service decides with [Validation](partner-validation.md): `retry` (deliver
  after all) or `return_to_merchant`;
- or the driver comes back first: [Validation `return`](partner-validation.md) with
  `"cancelled": false`, and the parcel goes back to the merchant.

| `reason` | Driver app button | Note |
| --- | --- | --- |
| `cancelByClient` (default) | Cancel by client | |
| `tries` | Decline (tries) | Needs at least **3 delivery attempts** on the order, this one included (`too_few_attempts`). |
| `other` | Cancelled | `note` is required. |

A delivery already refused can't be refused again with `cancelByClient` / `other`
(`already_declined`): send `reset` first to change it.

### `wrong_phone`

```json
{ "orderId": 123460, "status": "wrong_phone" }
```

The driver keeps the parcel. A "wrong phone" check opens (unless the contract skips it).

### `missing`

```json
{ "orderId": 123461, "status": "missing" }
```

The order is marked **missing** for your depot to look into. Its delivery is still "out for
delivery": when the parcel is found, send its real result.

### `reset`

```json
{ "orderId": 123458, "status": "reset" }
```

Undoes the last result: the delivery is back to plain "out for delivery", and its pending
checks close. Then send the right result.

## 4. Rules

- **Only deliveries on this sheet.** The parcel must be out with the driver on this collected
  sheet (not delivered, returned or cancelled). Anything else is refused for that parcel,
  and nothing changes for it ([section 7](#7-every-error-and-what-to-do-about-it)).
- **Several results a day** are fine: each is on the timeline; the last one is the current
  status.
- **No partial delivery and no proof** (photo, signature, GPS) in this version.
- **Webhooks:** no webhook comes back to you for a result you sent. The merchant's webhooks
  and the customer SMS still go out.
- **A finished sheet** (every parcel delivered or returned: it closed by itself) still
  answers: a resent result is `duplicate`, anything else is refused per parcel (e.g.
  `already_delivered`). Nothing changes.

## 5. Offline driver app and retries

- **Send `at` with every result.** We apply a call's results **in the order of `at`**, not
  the order of the list.
- **Late results:** a result older than the parcel's latest one (an offline app syncing
  after another device) is answered `late`. It's added to the timeline at its own time;
  the current status doesn't change, and no SMS or check is sent for it.
- **Safe to retry:** the same result sent again (same `status`, `at`, `date`, `reason`) is
  answered `duplicate` and writes nothing, even if you resend a whole batch. So when an
  answer is lost or times out, **send the same call again**.
- `at` can't be more than 48 hours old, or earlier than the moment the driver collected the
  parcel (`at_before_collect`).

## 6. The response

**200**, one entry per result, in the order you sent them:

```json
{
  "ok": true,
  "results": [
    { "orderId": 123456, "ok": true, "applied": "applied", "orderStatus": "inTransit", "missionStatus": "incompleted" },
    { "orderId": 123457, "ok": true, "applied": "duplicate", "orderStatus": "inTransit", "missionStatus": "rescheduled" },
    { "orderId": 123458, "ok": true, "applied": "late", "orderStatus": "inTransit", "missionStatus": "declined" },
    { "orderId": 123459, "ok": true, "applied": "applied", "orderStatus": "delivered", "missionStatus": "success" }
  ]
}
```

| `applied` | Meaning |
| --- | --- |
| `applied` | Recorded. |
| `duplicate` | Already recorded: nothing written. |
| `late` | Older than the parcel's latest result: on the timeline only. |

`orderStatus` is the order's status now; `missionStatus` the delivery's: `inTransit` (out for
delivery), `incompleted` (unreachable / wrong phone), `rescheduled` (postponed), `declined`,
`success` (delivered).

When a result is refused, its entry is `{ "orderId", "ok": false, "error", "message" }` and
the top-level `ok` is `false`. **The other results were applied.**

Every answer carries an **`x-request-id`** header. Log it.

## 7. Every error, and what to do about it

Whole call refused, nothing written: `{ "ok": false, "error", "message", "details"? }`.

| HTTP | `error` | What it means | What to do |
| --- | --- | --- | --- |
| **400** | `invalid_json` | The body is not JSON. | Fix the request. |
| **400** | `invalid_payload` | A field is missing, wrong or unknown. `details` = `{ field, reason }`, e.g. `{ "field": "results[0].date", "reason": "invalid_date" }`. | Fix that field. |
| **400** | `employee_not_found` | `employeeId` isn't one of your employees. | Check your employee ids. |
| **401** | `missing_headers`, `invalid_api_key`, `invalid_signature` | Authentication failed. | Check your key, and that you sign the exact body you send. |
| **403** | `partner_scope_not_configured` | Your key isn't enabled for partner APIs. | Contact us. |
| **404** | `runsheet_not_found` | No sheet with this `reference`. | Check the reference (Last Mile `create` first). |
| **409** | `not_collected` | The sheet isn't collected yet: its parcels are in the depot. | Last Mile `collect` first. |
| **409** | `driver_mismatch` | `driverId` isn't the sheet's driver. | Send the sheet's driver, or no `driverId`. |
| **409** | `runsheet_closed` | The sheet was closed on our side. | Don't retry. |
| **409** | `busy` | Another call on this sheet is in flight. | Send it again after a short pause. |
| **500** | `internal_error` | Our side; nothing written. | Send it again once; then contact us with the `x-request-id`. |

`invalid_payload` reasons: `invalid_body`, `unknown_field`, `invalid_reference`,
`invalid_driver_id`, `invalid_employee_id`, `missing_results`, `too_many_results`,
`invalid_result`, `invalid_order_id`, `invalid_status`, `invalid_note`, `date_not_allowed`,
`invalid_date`, `invalid_reason`, `reason_not_allowed`, `note_required`, `invalid_at`.

Per result, in `results` (nothing changed for that parcel):

| `error` | What it means | What to do |
| --- | --- | --- |
| `order_not_found` | No order with this id. | Check the id. |
| `unauthorized_partner` | Another delivery partner's order. | It isn't yours. |
| `not_on_this_runsheet` | The parcel isn't out on this sheet. | Check the `reference` (it may be on another sheet). |
| `not_collected` | `collect` left it in the depot (it changed after you added it). | Last Mile `remove` it. |
| `not_last_mile` | It's on a pick-up or a transfer, not a delivery. | Only delivery results are accepted. |
| `mission_not_in_transit` | Its delivery isn't out with the driver. | Check the parcel ([Get Order](partner-get-order.md)). |
| `order_cancelled` | The order is cancelled. | Nothing to record. |
| `already_delivered` | Already delivered. | Nothing to do. |
| `already_returned` | Already returned to the merchant. | Nothing to do. |
| `already_declined` | Already refused. | Send `reset` first to change it. |
| `not_for_customer` | `delivered` on a parcel going back to the merchant. | Validation `retry` first, if it should be delivered after all. |
| `too_few_attempts` | `declined` with `reason: "tries"` before 3 delivery attempts. | Use `cancelByClient` or `other`. |
| `at_before_collect` | `at` is earlier than the driver's collect of the parcel. | Check the app's clock and `at`. |
| `busy` | Another call on this parcel is in flight. | Send it again. |
| `internal_error` | Our error on this result. | Send it again later. |

## 8. Copy-paste example

```bash
API_KEY="your-staging-api-key"
API_SECRET="your-staging-api-secret"
BODY='{"reference":"LM-2026-000981","driverId":42,"results":[{"orderId":123456,"status":"unreachable","at":"2026-11-20T09:10:00+01:00"}]}'

SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" | awk '{print $2}')

curl -X POST https://staging.livra.mofavo.com/partner_mission_status \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  -d "$BODY"
```

Node.js, Python and PHP versions of the same signing:
[Last Mile, section 5](partner-last-mile.md#5-copy-paste-examples) (change the URL and the body).

## 9. Checklist before you go live

- [ ] Each result is sent **when it happens**, with `at`.
- [ ] A lost or timed-out call is sent again unchanged (it's safe: `duplicate`).
- [ ] You read `ok`, and each result's `ok` / `error` when it's `false`.
- [ ] A refused delivery is followed by a Validation decision, or by `return` when the driver
      is back ([Delivery Round](delivery-round.md#3-after-each-driver-result-what-comes-next)).
- [ ] You no longer call Last Mile `deliver`: `delivered` here replaces it.
- [ ] You log the `x-request-id` of every answer.

[← Back to Livra integration guide](README.md)
