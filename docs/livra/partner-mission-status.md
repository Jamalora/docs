# Mission Status — Partner API

[← Back to Livra integration guide](README.md)

> **Staging only for now.** This endpoint is live on `https://staging.livra.mofavo.com/partner_mission_status`
> for testing. It is not in production yet.

This page is for **delivery partners**. It explains one endpoint:
`POST /partner_mission_status`: what your **driver records at the customer**, sent right
away. Each result does exactly what the same button does in our own driver app: the mission and order status, the
timeline entry, the customer SMS or the quality-control ticket, the merchant's webhooks.

It goes with [Last Mile](partner-last-mile.md) (the sheet) and [Validation](partner-validation.md) (back in
the depot, and where the parcel goes next).

- **URL:** `https://livra.mofavo.com/partner_mission_status`
  (staging: `https://staging.livra.mofavo.com/partner_mission_status`)
- **Method:** `POST`

## Request body

```json
{
  "reference": "LM-2026-000981",
  "driverId": 42,
  "employeeId": 7,
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
| `reference` | yes | The delivery sheet, as in [Last Mile](partner-last-mile.md). It must be collected. |
| `driverId` | no | Our id for the driver. When sent, it must be the sheet's driver (else `driver_mismatch`). |
| `employeeId` | no | One of your employees, when someone records it for the driver (on the timeline). |
| `results` | yes | 1 to 100 results. |
| `results[].orderId` | yes | Our order id. |
| `results[].status` | yes | One of the statuses below. |
| `results[].at` | no | When the driver recorded it, ISO 8601 with an offset (default: now). It can be in the past if the app was offline: at most 48 hours, and not before the driver collected the parcel (`at_before_collect`). Not in the future. |
| `results[].date` | `postponed` only | `YYYY-MM-DD`, today or later (Tunis time): the new delivery day. Without it, `postponed` is a comment. |
| `results[].reason` | `declined` only | `cancelByClient` (default), `tries` or `other`. |
| `results[].note` | no (yes with `reason: other`) | The driver's comment (at most 500 characters), on the timeline. |

Unknown fields are refused.

## Statuses

| `status` | Our driver app's button | Mission | Order | What follows |
| --- | --- | --- | --- | --- |
| `delivered` | Deliver | `success` | `delivered`, delivery date = `at` | Counted in the merchant's settlement. An exchange is swapped at the door instead: it stays `inTransit` with the driver, who brings the old item back (Validation `return` with `cancelled: true`). **Refused (`not_for_customer`) when the order is going back to the merchant.** |
| `unreachable` | Unreachable | `rescheduled` / `incompleted`, stays with the driver | flagged for attention | "Unreachable" SMS or QC ticket (the contract decides). |
| `postponed` + `date` | Reschedule | `rescheduled` | planned date = `date`, flagged | "Postponed" SMS or QC ticket. |
| `postponed`, no `date` | Comment | `rescheduled` | flagged | The note on the timeline. |
| `declined` | Cancel by client (`cancelByClient`), Decline (tries) (`tries`), Cancelled (`other` + `note`) | `declined`, stays with the driver | `inTransit`, **destination unchanged**, flagged | The merchant is asked what to do (QC ticket, or the contract's SMS). |
| `wrong_phone` | Incorrect phone | `rescheduled` / `incompleted` | flagged | An "incorrect phone" QC ticket. |
| `missing` | Missing | unchanged | `missing` | For the depot to look into. |
| `reset` | Refresh to init | back to `inTransit` | attention cleared, pending QC tickets closed | Undoes a mistake. Refused on a delivered order. |

## Rules

- **Only deliveries**: the order must be out with the driver on this collected sheet
  (its delivery `inTransit`, `rescheduled` or `declined`). A parcel left in the depot
  (`not_collected`), on another sheet (`not_on_this_runsheet`), on a pick-up or transfer
  (`not_last_mile`), another partner's (`unauthorized_partner`), delivered, returned or
  cancelled is refused, and nothing changes for it.
- **Several results a day**: unreachable, then postponed, then delivered… each one is on
  the timeline, the last one is the current status.
- **Offline**: results are applied **in the order of `at`**, not the order they arrive. A
  result older than the order's latest one is answered `late`: it goes on the timeline at
  its own time, and the current status doesn't change.
- **Safe to retry**: the same result sent again (same status, `at`, `date`, `reason`) is
  answered `duplicate` and writes nothing.
- **Declined**: the parcel doesn't go back to the merchant right away. `cancelByClient` or
  `other` on a delivery already declined is refused (`already_declined`; `reset` first).
  `tries` needs at least 3 delivery attempts on the order, the current one included
  (`too_few_attempts`). When the merchant has answered, or the driver is back:
  [Validation](partner-validation.md) `return_to_merchant` / `retry` while the driver still has it,
  `return` (with `cancelled: false`: the return sends it to the merchant) once it is back.
- **No partial delivery, no proof** (photo, signature, GPS) in this version.
- **Webhooks**: no webhook to you for a status you sent; the merchant's webhooks and the
  customer SMS still go out.

## Success

**200**, one entry per result, in your order:

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

`applied`: `applied` (recorded), `duplicate` (already recorded, nothing written), `late`
(an older result: on the timeline only). When a result is refused, its entry is
`{ "orderId", "ok": false, "error", "message" }` and the top-level `ok` is `false`; the
other results are applied.

## Errors

Whole call (nothing written):

| HTTP | `error` | When |
| --- | --- | --- |
| **400** | `invalid_payload` | The body is wrong: `details` = `{ field, reason }`, e.g. `{ "field": "results[0].date", "reason": "invalid_date" }`. Reasons: `invalid_body`, `unknown_field`, `invalid_reference`, `invalid_driver_id`, `invalid_employee_id`, `missing_results`, `too_many_results`, `invalid_result`, `invalid_order_id`, `invalid_status`, `invalid_note`, `date_not_allowed`, `invalid_date`, `invalid_reason`, `reason_not_allowed`, `note_required`, `invalid_at`. |
| **400** | `invalid_json` | The body is not JSON. |
| **400** | `employee_not_found` | `employeeId` is not one of your employees. |
| **401** | `missing_headers`, `invalid_api_key`, `invalid_signature` | Authentication. |
| **403** | `partner_scope_not_configured` | Your key is not enabled for partner routes. |
| **404** | `runsheet_not_found` | No delivery sheet with this `reference`. |
| **409** | `not_collected`, `runsheet_closed`, `driver_mismatch` | The sheet isn't collected yet, is finished, or `driverId` isn't its driver. |
| **409** | `busy` | Another call on this sheet is in flight. Send it again. |
| **500** | `internal_error` | Our side. Send it again later. |

Per result, in `results`:

| `error` | What it means |
| --- | --- |
| `order_not_found` | No order with this id. |
| `unauthorized_partner` | Another delivery partner's order. |
| `not_on_this_runsheet` | Not out on this sheet. |
| `not_collected` | Collect left it in the depot. |
| `not_last_mile` | On a pick-up or transfer: only delivery results are accepted. |
| `mission_not_in_transit` | Its delivery isn't out with the driver. |
| `order_cancelled` | The order is cancelled. |
| `already_delivered` | Already delivered. |
| `already_returned` | Already returned to the merchant. |
| `already_declined` | Already declined (`cancelByClient` / `other`); send `reset` first to change it. |
| `not_for_customer` | `delivered` on an order going back to the merchant. Validation `retry` first. |
| `too_few_attempts` | `declined` with `tries` before 3 delivery attempts. |
| `at_before_collect` | `at` is earlier than the driver's collect of the parcel. |
| `busy` | Another call on this order is in flight. Send it again. |
| `internal_error` | Our error on this result. Send it again later. |

## Signing

Same key and signing as every partner endpoint: `x-api-key` and `x-signature` (lowercase
hex HMAC-SHA256 of the raw body bytes with your secret). See
[How to build `x-signature`](partner-last-mile.md#4-how-to-build-x-signature) and the
copy-paste examples there.
