# Livra Webhooks — Advanced (v1)

[← Back to Livra integration guide](README.md)

The advanced webhook delivers a rich, **uniform** snapshot of the order on every
event. The `data` object has the **same shape for every event type** — the only
things that vary between events are the `event` name, which optional sections are
populated (`driver`, `depot`, `mission`, `settlement`), and the contents of
`previous`. Parse one shape, handle every event.

## Event catalogue

Three families of events: **order events**, driven by changes on the shipping
order; **driver events**, driven by driver actions during last-mile delivery;
and **invoice events**, driven by settlement.

### Order events

Emitted when any of `status`, `finalDestination`, `currentPosition`, or
`deliveryDate` changes on the order.

| `event`                 | Fires when                 |
| ----------------------- | -------------------------- |
| `status.changed`        | `status` changed           |
| `destination.changed`   | `finalDestination` changed |
| `position.updated`      | `currentPosition` changed  |
| `delivery_date.changed` | `deliveryDate` changed     |

Only **one** order event is emitted per update. If multiple tracked fields
change in the same update, the first match in this priority order wins:
`status` > `finalDestination` > `currentPosition` > `deliveryDate`. The full
`data` snapshot is identical regardless of which field triggered the event — use
`previous` to see exactly what changed.

### Driver events

Emitted when the driver records a last-mile outcome. Other mission transitions
(e.g. `success`, `inTransit`, `fail`, `blocked`, `ready`) do **not** produce a
webhook.

| `event`              | Fires when                           |
| -------------------- | ------------------------------------ |
| `driver.declined`    | recipient declined the delivery      |
| `driver.unreachable` | driver could not reach the recipient |
| `driver.dated`       | delivery rescheduled to a later date |

On a driver event, the `driver` and `mission` sections describe **the mission
the driver acted on** (see [`mission`](#driver-depot-and-mission)).

### Invoice events

Emitted when the delivery partner settles the invoice associated with an order.

| `event`           | Fires when                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------- |
| `invoice.settled` | `settlementDeliverypartner` is set for the first time — fires once per order, not on re-settlements. |

On an invoice event, the `settlement` section is populated.

## Request format

```
POST <your-callback-url>
Content-Type: application/json
X-Webhook-ID: <delivery-uuid>
X-Webhook-Signature: <hmac-hex>
X-Webhook-Type: advanced
X-Webhook-Version: 1
```

`X-Webhook-Type` is always `advanced` for these webhooks. `X-Webhook-Version`
is the payload version — use it to guard your parsing logic if the payload ever
changes.

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

- `order_id` is the shipping order id returned by [Create Order](README.md#create-order).
- `timestamp` is when the event was detected on the platform, in UTC ISO 8601.
  On retries it reflects the **original event time**, not the retry time.
- `data` is the uniform snapshot (below).
- `previous` holds only the fields that changed, taken at the moment of the
  event (see [Previous values](#previous-values)).

> **Snapshot timing.** `data` is read when the webhook is dispatched, so it
> reflects the order's state **as of delivery**, which is effectively the moment
> of the event but may include changes that landed microseconds later. `previous`
> always reflects the exact pre-change values captured when the event fired.

### The `data` object

The same keys are always present. Optional relations are `null` when they don't
apply: `driver`/`mission`/`depot` are populated when the order has a relevant
mission, and `settlement` only after the delivery partner settles.

```json
{
  "event": "status.changed",
  "order_id": 1234,
  "timestamp": "2026-05-05T11:23:00Z",
  "data": {
    "status": "inTransit",
    "finalDestination": "primaryRecipient",
    "currentPosition": "Acme Depot##42 Main St##Dubai##Dubai##00000",
    "deliveryDate": null,
    "amount": 120.5,
    "isExchange": false,
    "oldOrderId": null,

    "customer": {
      "name": "Tarek M.",
      "phone": "55916219",
      "phone2": null,
      "address": {
        "street": "12 Rue de Carthage",
        "zone": "Beni Khalled",
        "city": "Beni Khalled",
        "state": "Nabeul",
        "zipcode": "8021"
      },
      "deliveryInstructions": "Call on arrival"
    },

    "merchant": { "id": 153, "name": "Promo Shop" },

    "sender": {
      "id": 332,
      "name": "Promo Shop Sender",
      "address": { "street": "Centre Ville", "city": "Ksar Hellal", "state": "Monastir" }
    },

    "products": [
      { "name": "Hoodie — brown XL", "quantity": 1, "price": 37.0 }
    ],

    "driver": { "id": 165, "name": "Ala B." },
    "depot":  { "id": 7,   "name": "Hammamet Hub" },
    "mission": {
      "id": 1742169,
      "status": "inTransit",
      "originType": "depot",
      "origin": "Sousse Hub##Soukra##Ksar Hellal##Monastir##5070",
      "destinationType": "finalRecipient",
      "destination": "Tarek M.##12 Rue de Carthage##Beni Khalled##Nabeul##8021"
    },

    "settlement": null
  },
  "previous": {
    "status": "inDepot",
    "finalDestination": "primaryRecipient",
    "currentPosition": "Acme Depot##42 Main St##Dubai##Dubai##00000",
    "deliveryDate": null
  }
}
```

#### Order fields

| Field | Type | Meaning |
| --- | --- | --- |
| `status` | string | Order status — see [`status`](#status). |
| `finalDestination` | string | Where the parcel is ultimately headed — see [`finalDestination`](#finaldestination). A **category**, not an address. |
| `currentPosition` | string \| null | Where the parcel currently is — `##`-separated, see [Address formats](#address-formats). |
| `deliveryDate` | string \| null | ISO 8601 timestamp when delivered, or `null` if not delivered yet. |
| `amount` | number \| null | Order amount (COD value). |
| `isExchange` | boolean | `true` when this order replaces a previous one. |
| `oldOrderId` | number \| null | The replaced order's id when `isExchange` is `true`; otherwise `null`. |

#### `customer`

The primary recipient.

| Field | Type | Meaning |
| --- | --- | --- |
| `name` | string \| null | Recipient name. |
| `phone`, `phone2` | string \| null | Primary and secondary phone numbers. |
| `address` | object | `{ street, zone, city, state, zipcode }` — any field may be `null`. |
| `deliveryInstructions` | string \| null | Free-text instructions, if any. |

#### `merchant` and `sender`

| Field | Type | Meaning |
| --- | --- | --- |
| `merchant.id` / `merchant.name` | number / string | The order's owning merchant. |
| `sender.id` / `sender.name` | number / string | The expedition origin (the merchant's sending entity). |
| `sender.address` | object | `{ street, city, state }` — senders have no zone or zipcode. |

#### `products`

Array of line items (empty array when none):

```json
"products": [ { "name": "Hoodie — brown XL", "quantity": 1, "price": 37.0 } ]
```

| Field | Type | Meaning |
| --- | --- | --- |
| `name` | string | Product name. |
| `quantity` | number | Quantity. |
| `price` | number \| null | Unit price, if recorded. |

#### `driver`, `depot`, and `mission`

These describe a delivery leg and are `null` together when the order has no
relevant mission.

- For **driver events**, they describe the mission the driver acted on.
- For **all other events**, they describe the mission currently `inTransit`; if
  none is in transit, the most recent mission for the order.

`depot` comes from that mission's depot and is `null` when the mission has none.

| Field | Type | Meaning |
| --- | --- | --- |
| `driver.id` / `driver.name` | number / string | Driver assigned to the mission (`null` when the mission has no driver). |
| `depot.id` / `depot.name` | number / string | Depot referenced by the mission. |
| `mission.id` | number | Mission id. |
| `mission.status` | string | The mission's own status (e.g. `inTransit`, `declined`) — distinct from the order `status`. |
| `mission.originType` / `mission.destinationType` | string | `depot`, `finalRecipient`, or `system`. |
| `mission.origin` / `mission.destination` | string | `##`-separated addresses — see [Address formats](#address-formats). |

#### `settlement`

`null` until the delivery partner settles the order's invoice; populated on the
`invoice.settled` event (and on later events once settlement has occurred).

| Field | Type | Meaning |
| --- | --- | --- |
| `settlement.date` | string | ISO 8601 timestamp the delivery partner recorded settlement. |

### Previous values

`previous` contains only the fields that changed, captured exactly at the event.
Its shape depends on the event family:

| Event family | `previous` |
| --- | --- |
| Order events | `{ status, finalDestination, currentPosition, deliveryDate }` — the tracked fields' values before the change. |
| Driver events | `{ missionStatus }` — the mission's prior status. |
| `invoice.settled` | `{ settlementDate: null }` — settlement always transitions from unset. |

A driver event, for example:

```json
{
  "event": "driver.dated",
  "order_id": 1234,
  "timestamp": "2026-05-05T11:23:00Z",
  "data": { "...": "same uniform shape; driver + mission populated" },
  "previous": { "missionStatus": "inTransit" }
}
```

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

### Address formats

`currentPosition` and the mission `origin`/`destination` are strings with five
`##`-separated fields:

```
<entityName>##<street>##<city>##<state>##<zipcode>
```

The `customer.address` and `sender.address` objects are structured instead (see
above), so you do not need to parse them.

### Dates

`deliveryDate`, `settlement.date`, and `timestamp` are ISO 8601 in UTC.
`deliveryDate` is `null` until delivery happens.

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
BODY='{"event":"status.changed","order_id":1234,"timestamp":"2026-05-05T11:23:00Z","data":{"status":"inTransit","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null,"amount":120.5,"isExchange":false,"oldOrderId":null,"customer":{"name":"Tarek M.","phone":"55916219","phone2":null,"address":{"street":"12 Rue de Carthage","zone":"Beni Khalled","city":"Beni Khalled","state":"Nabeul","zipcode":"8021"},"deliveryInstructions":null},"merchant":{"id":153,"name":"Promo Shop"},"sender":{"id":332,"name":"Promo Shop Sender","address":{"street":"Centre Ville","city":"Ksar Hellal","state":"Monastir"}},"products":[{"name":"Hoodie","quantity":1,"price":37.0}],"driver":{"id":165,"name":"Ala B."},"depot":null,"mission":{"id":1742169,"status":"inTransit","originType":"depot","origin":"Sousse Hub##Soukra##Ksar Hellal##Monastir##5070","destinationType":"finalRecipient","destination":"Tarek M.##12 Rue de Carthage##Beni Khalled##Nabeul##8021"},"settlement":null},"previous":{"status":"inDepot","finalDestination":"primaryRecipient","currentPosition":"Acme Depot##42 Main St##Dubai##Dubai##00000","deliveryDate":null}}'
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$SECRET" | awk '{print $2}')

curl -X POST https://your-endpoint.example.com/webhook \
  -H "Content-Type: application/json" \
  -H "X-Webhook-ID: test-$(uuidgen)" \
  -H "X-Webhook-Signature: $SIG" \
  -H "X-Webhook-Type: advanced" \
  -H "X-Webhook-Version: 1" \
  -d "$BODY"
```
