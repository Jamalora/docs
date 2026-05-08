# Livra Webhooks — Staging Preview

> 🚫 **DO NOT IMPLEMENT AGAINST THIS YET.**
>
> This page describes an upcoming webhook format that is **only deployed to
> Livra's staging environment**. It is **not** what `livra.mofavo.com`
> production sends today, and it is **subject to change** before it ships.
> If you code against this format and point it at production, your endpoint
> will not receive anything it recognises.
>
> **What production sends today** — and what you should be coding against —
> is documented in
> [Order status webhooks](README.md#order-status-webhooks) in the main
> integration guide.

[← Back to Livra integration guide](README.md)

## What's changing

The webhook is moving from a single batched message with `deliveryStatus` /
`orderStatus` / `comment` to a per-event model. Each delivery represents one
event, with an explicit event name, a snapshot of the order's current
tracked fields, and the previous values for the fields that the event
covers. The driver `comment` field is replaced by dedicated `driver.*`
events.

What stays the same: the request method, the headers (`X-Webhook-ID`,
`X-Webhook-Signature`), HMAC-SHA256 signature verification with your
Livra API secret, the 10-second response timeout, the 5-attempt retry
schedule, and the rules for acknowledging deliveries. Refer to the
[production webhook section](README.md#order-status-webhooks) for those —
nothing in that material changes.

## Event catalogue

Two families of events: **order events**, driven by changes on the shipping
order itself, and **driver events**, driven by driver actions during
last-mile delivery. Both require `callback_link` to have been set when the
order was created.

### Order events

Emitted when any of `status`, `finalDestination`, `currentPosition`, or
`deliveryDate` changes on the order.

| `event`                 | Fires when                 |
| ----------------------- | -------------------------- |
| `status.changed`        | `status` changed           |
| `destination.changed`   | `finalDestination` changed |
| `position.updated`      | `currentPosition` changed  |
| `delivery_date.changed` | `deliveryDate` changed     |

Only **one** order event is emitted per update. If multiple trigger fields
change in the same update, the first match in this priority order wins:
`status` > `finalDestination` > `currentPosition` > `deliveryDate`.

### Driver events

Emitted when the driver records a last-mile outcome. Other mission
transitions (e.g. `success`, `inTransit`, `fail`, `blocked`, `ready`) do
**not** produce a webhook.

| `event`              | Fires when                           |
| -------------------- | ------------------------------------ |
| `driver.declined`    | recipient declined the delivery      |
| `driver.unreachable` | driver could not reach the recipient |
| `driver.dated`       | delivery rescheduled to a later date |

## Payload

Every event uses the same envelope:

```json
{
  "event": "<event-type>",
  "order_id": 1234,
  "timestamp": "2026-05-05T11:23:00Z",
  "data":     { ... },
  "previous": { ... }
}
```

`order_id` is the shipping order id returned by [Create Order](README.md#create-order). `timestamp` is when the event was detected on the platform, in UTC ISO 8601; on retries it reflects the **original event time**, not the retry time. The contents of `data` and `previous` differ between order events and driver events.

### Order event payload

For `status.changed`, `destination.changed`, `position.updated`, and `delivery_date.changed`, `data` is the current snapshot of the four tracked fields, and `previous` is their values immediately before the change.

```json
{
  "event": "status.changed",
  "order_id": 1234,
  "timestamp": "2026-05-05T11:23:00Z",
  "data": {
    "status": "inTransit",
    "finalDestination": "primaryRecipient",
    "currentPosition": "Acme Depot##42 Main St##Dubai##Dubai##00000",
    "deliveryDate": null
  },
  "previous": {
    "status": "inDepot",
    "finalDestination": "primaryRecipient",
    "currentPosition": "Acme Depot##42 Main St##Dubai##Dubai##00000",
    "deliveryDate": null
  }
}
```

### Driver event payload

For `driver.declined`, `driver.unreachable`, and `driver.dated`, `data` carries driver identity and last-mile context plus a snapshot of the order's headline fields. `previous` carries only the prior `missionStatus`.

```json
{
  "event": "driver.dated",
  "order_id": 1234,
  "timestamp": "2026-05-05T11:23:00Z",
  "data": {
    "missionStatus": "rescheduled",
    "driverNote": "Customer asked to redeliver tomorrow afternoon",
    "driverId": 42,
    "driverName": "Aymen",
    "missionOrderId": 9876,
    "orderStatus": "inTransit",
    "finalDestination": "primaryRecipient",
    "deliveryDate": null
  },
  "previous": {
    "missionStatus": "inTransit"
  }
}
```

| Field | Meaning |
| --- | --- |
| `missionStatus` | The new mission status (`declined`, `incompleted`, or `rescheduled`). |
| `driverNote` | Free-form note left by the driver. May be `null`. |
| `driverId`, `driverName` | Driver who recorded the outcome. |
| `missionOrderId` | Internal id of the mission record (distinct from the top-level `order_id`). |
| `orderStatus` | The shipping order's current `status` — same enum as in [`status`](#status) below. |
| `finalDestination`, `deliveryDate` | Snapshot of the order's other tracked fields. Note: `currentPosition` is **not** included in driver-event payloads. |

## Field reference

### `status`

Where the order sits in the operational pipeline.

| Value | Meaning |
| --- | --- |
| `readyForPickUp` | Created and waiting to be collected by a courier. |
| `missing` | The order was never received (no first-mile pickup recorded). |
| `inDepot` | Held at a depot (before first-mile pickup, between legs, or after a customer decline). |
| `inTransit` | On the road. Use `finalDestination` to tell whether it is heading toward the customer or back to the merchant. |
| `delivered` | Delivered to the customer. |
| `returned` | Returned to the merchant. |
| `exchange-complete` | Exchange happened with the customer; the goods collected from the customer have been returned to the merchant. |
| `exchange-returned` | Exchange was declined by the customer; the parcel sent for the exchange has been returned to the merchant. |
| _(other values)_ | New or internal statuses — treat unknown values gracefully. |

### `finalDestination`

Where the parcel is currently heading.

| Value | Meaning |
| --- | --- |
| `primaryRecipient` | Toward the customer. |
| `merchant` | Toward (or back at) the merchant. |

### `currentPosition`

A string with five `##`-separated fields:

```
<entityName>##<street>##<city>##<state>##<zipcode>
```

`entityName` is the merchant, the delivery-partner depot, or the primary
recipient that the parcel is currently at; the address fields are that
entity's address.

### `deliveryDate`

ISO 8601 date (`YYYY-MM-DD`), or `null` if delivery has not happened yet.

## Deriving delivery outcome

The webhook no longer sends a `deliveryStatus` field. Derive it from
`deliveryDate` and `finalDestination`:

| Condition | Outcome |
| --- | --- |
| `deliveryDate` is `null` and `finalDestination` is `primaryRecipient` | `pending` — still in play |
| `deliveryDate` is `null` and `finalDestination` is `merchant` | `declined` — customer refused, parcel is on its way back to the merchant |
| `deliveryDate` is not `null` | `delivered` |

For an exchange order, "delivered" means the exchange happened with the
customer. The exchanged-for goods are then on their way back to the
merchant, so `status` may still be `inTransit` even though the delivery
outcome is already settled.

## Testing your endpoint

Same approach as production — sign the raw body with your Livra API secret
and `POST` it to your endpoint. Only the body shape differs:

```bash
SECRET="your-secret"
BODY='{"event":"status.changed","order_id":1234,"timestamp":"2026-05-05T11:23:00Z","data":{"status":"inTransit","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null},"previous":{"status":"inDepot","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null}}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://your-endpoint.example.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Webhook-ID: test-$(uuidgen)" \
  -H "X-Webhook-Signature: $SIG" \
  -d "$BODY"
```

---

> 🚫 **Reminder — this page is staging only.**
> Production (`livra.mofavo.com`) is still sending the
> [legacy webhook format](README.md#order-status-webhooks). Do not switch
> your integration over to anything described above until we announce that
> the new format is live in production.
