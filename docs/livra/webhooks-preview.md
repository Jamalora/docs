# Livra Webhooks — Advanced (v1)

[← Back to Livra integration guide](README.md)

The advanced webhook delivers the raw event payload exactly as recorded by the platform. You receive the full context — previous values, driver identity, mission details — and derive your own conclusions from it. If you prefer a pre-computed, compact summary instead, see the [simple webhook](README.md#simple-webhook).

## Event catalogue

Two families of events: **order events**, driven by changes on the shipping
order itself, and **driver events**, driven by driver actions during last-mile
delivery. Both require `callback_link` to have been set when the order was
created.

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

Emitted when the driver records a last-mile outcome. Other mission transitions
(e.g. `success`, `inTransit`, `fail`, `blocked`, `ready`) do **not** produce a
webhook.

| `event`              | Fires when                           |
| -------------------- | ------------------------------------ |
| `driver.declined`    | recipient declined the delivery      |
| `driver.unreachable` | driver could not reach the recipient |
| `driver.dated`       | delivery rescheduled to a later date |

### Invoice events

Emitted when the delivery partner settles the invoice associated with an order.

| `event`           | Fires when                                               |
| ----------------- | -------------------------------------------------------- |
| `invoice.settled` | `settlementDeliverypartner` is set for the first time — fires once per order, not on re-settlements |

## Request format

```
POST <your-callback-url>
Content-Type: application/json
X-Webhook-ID: <delivery-uuid>
X-Webhook-Signature: <hmac-hex>
X-Webhook-Type: advanced
X-Webhook-Version: 1
```

`X-Webhook-Type` identifies the webhook flavour (`simple` or `advanced`).
`X-Webhook-Version` is the version within that flavour. Use both together to
route parsing logic if you ever handle more than one flavour or version on the
same endpoint.

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

`order_id` is the shipping order id returned by [Create Order](README.md#create-order).
`timestamp` is when the event was detected on the platform, in UTC ISO 8601. On
retries it reflects the **original event time**, not the retry time.

### Order event payload

For `status.changed`, `destination.changed`, `position.updated`, and
`delivery_date.changed`, `data` is the current snapshot of the four tracked
fields; `previous` is their values immediately before the change.

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

For `driver.declined`, `driver.unreachable`, and `driver.dated`, `data` carries
driver identity and last-mile context plus a snapshot of the order's headline
fields. `previous` carries only the prior `missionStatus`.

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

### Invoice event payload

For `invoice.settled`, `data` carries the settlement date alongside a snapshot
of the order's headline fields at the time of settlement. `previous` carries
only the prior `settlementDate` (always `null` — this event fires once).

```json
{
  "event": "invoice.settled",
  "order_id": 1234,
  "timestamp": "2026-05-13T10:00:00Z",
  "data": {
    "settlementDate": "2026-05-13",
    "status": "delivered",
    "finalDestination": "primaryRecipient",
    "deliveryDate": "2026-05-10"
  },
  "previous": {
    "settlementDate": null
  }
}
```

| Field | Meaning |
| --- | --- |
| `settlementDate` | Date the delivery partner recorded settlement (`YYYY-MM-DD`). |
| `status`, `finalDestination`, `deliveryDate` | Snapshot of the order's state at settlement time — same fields and values as in order event payloads. |

---

## Field reference

### `status`

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

| Value | Meaning |
| --- | --- |
| `primaryRecipient` | Toward the customer. |
| `merchant` | Toward (or back at) the merchant. |

### `currentPosition`

A string with five `##`-separated fields:

```
<entityName>##<street>##<city>##<state>##<zipcode>
```

### `deliveryDate`

ISO 8601 date (`YYYY-MM-DD`), or `null` if delivery has not happened yet.

## Deriving delivery outcome

The advanced payload does not include a pre-computed `deliveryStatus`. Derive it
from `deliveryDate` and `finalDestination`:

| Condition | Outcome |
| --- | --- |
| `deliveryDate` is `null` and `finalDestination` is `primaryRecipient` | pending — still in play |
| `deliveryDate` is `null` and `finalDestination` is `merchant` | declined — customer refused, parcel heading back |
| `deliveryDate` is not `null` | delivered |

## Testing your endpoint

```bash
SECRET="your-secret"
BODY='{"event":"status.changed","order_id":1234,"timestamp":"2026-05-05T11:23:00Z","data":{"status":"inTransit","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null},"previous":{"status":"inDepot","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null}}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://your-endpoint.example.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Webhook-ID: test-$(uuidgen)" \
  -H "X-Webhook-Signature: $SIG" \
  -H "X-Webhook-Type: advanced" \
  -H "X-Webhook-Version: 1" \
  -d "$BODY"
```
