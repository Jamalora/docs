# Livra Integration Guide (Production)

This document describes how to call Livra integration endpoints from your app.

## Contents

- [Shared authentication headers](#shared-authentication-headers)
- [Create Merchant](#create-merchant)
- [Update Merchant](partner-update-merchant.md)
- [Create Order](#create-order)
- [Update Order](#update-order)
- [Order Hash](#order-hash)
- [Change Request](#change-request)
- [Order status webhooks](#order-status-webhooks)
- [Accept In Depot (delivery partners)](partner-accept-in-depot.md)
- [Transfers Between Depots (delivery partners)](partner-transfers.md)
- [Get Order (delivery partners)](partner-get-order.md)
- [Merchant Settlement (delivery partners)](partner-merchant-settlement.md)
- [Driver Settlement (delivery partners)](partner-driver-settlement.md) (staging only for now)
- [Returns (delivery partners)](partner-return.md)
- [Delivery Round (delivery partners)](delivery-round.md): **start here** for Last Mile, Mission Status and Validation: who calls what, when (staging only for now)
- [Last Mile (delivery partners)](partner-last-mile.md)
  - [Validation (delivery partners)](partner-validation.md): where its orders go, and what the driver brings back (staging only for now)
  - [Mission Status (delivery partners)](partner-mission-status.md): what the driver records at the customer: delivered, unreachable, postponed, refused… (staging only for now)
- [Drivers and Employees (delivery partners)](partner-staff.md) (staging only for now)

> **Are you a delivery partner working with your own depots?**
> Those endpoints have their own pages:
> [Accept In Depot](partner-accept-in-depot.md) to check a parcel in when a truck
> arrives, [Transfers Between Depots](partner-transfers.md) to send parcels on to
> the next depot, [Get Order](partner-get-order.md) to read one of your orders back
> (status, dates, and the slip hash for printing),
> [Merchant Settlement](partner-merchant-settlement.md) to record a merchant settlement you
> already paid on your platform, [Driver Settlement](partner-driver-settlement.md) to record a
> driver's cash handover or fee payout done in your tool (on staging now, for testing), [Returns](partner-return.md) to keep your return
> sheets (parcels going back to their merchant) in sync, and [Last Mile](partner-last-mile.md)
> to keep your drivers' delivery sheets in sync, up to the moment they collect the parcels,
> and [Validation](partner-validation.md) to decide where an order goes (back to the merchant
> or to the customer again) and take back what a driver brings back.
> To create your drivers and employees on our side and get the ids those endpoints need, use
> [Drivers and Employees](partner-staff.md) (on staging now, for testing).
> To change a merchant you created (information, sender, fees, retenue), use
> [Update Merchant](partner-update-merchant.md).

## Shared authentication headers

Use these headers for all Livra integration endpoints:

- `Content-Type: application/json`
- `x-api-key: <apiKey>`
- `x-signature: <hexHmac>`

`x-signature` must be `HMAC-SHA256(rawRequestBody, apiSecret)` encoded as lowercase hex (optionally prefixed with `sha256=`).

## Create Merchant

- **URL:** `https://livra.mofavo.com/create_merchant`
- **Method:** `POST`

Creates a merchant with its sender and its contract and, optionally, the contract's sous-contrats: separate prices for `medium`, `large` and `extra_large` orders.

### Request body

```json
{
  "merchant": {
    "name": "Example Merchant LLC",
    "state": "Dubai",
    "city": "Dubai",
    "street": "Example Street 1",
    "phoneNumber": "+971500000000",
    "zipcode": "00000",
    "TRN": "100000000000003",
    "CIN": "01234567"
  },
  "sender": {
    "name": "Example Sender LLC",
    "state": "Dubai",
    "city": "Dubai",
    "street": "Business Bay",
    "phoneNumber": "+971511111111"
  },
  "contract": {
    "deliveryPartnerId": 10,
    "deliveryFee": 12.5,
    "exchangeFee": 4.25,
    "cancellationFee": 3.0,
    "hasRetenue": true,
    "sizeFees": {
      "medium":      { "deliveryFee": 13.5, "exchangeFee": 5.25, "cancellationFee": 3.0 },
      "large":       { "deliveryFee": 15,   "exchangeFee": 6,    "cancellationFee": 3.0 },
      "extra_large": { "deliveryFee": 18,   "exchangeFee": 8,    "cancellationFee": 3.0 }
    }
  }
}
```

### Rules

- `merchant.TRN` and `merchant.CIN` are each optional, but **at least one** must be provided.
- When provided, `merchant.CIN` must be a **string** of **exactly 8 digits** (e.g. `"01234567"`), so leading zeros are kept. `merchant.TRN` must be a non-empty string.
- All other merchant/sender string fields must be non-empty.
- `contract.deliveryPartnerId` must be a positive integer.
- Contract fees must be non-negative numbers.
- `contract.sizeFees` is **optional**. Its keys can only be `medium`, `large` and `extra_large`, and you can send any of them. Each size you send needs all three fees (`deliveryFee`, `exchangeFee`, `cancellationFee`), each a non-negative number.
- A size you leave out gets no sous-contrat: orders of that size **cannot be settled** until one is added for it. They are not billed at the base fees.
- `contract.hasRetenue` is **optional**. It says whether the 3% retenue à la source applies to this merchant's orders: `true` it applies, `false` it does not, `null` or left out it is decided automatically (the retenue applies when the merchant has no matricule fiscal, a TRN with letters such as `1935742/N/N/M/000`). Any other value (`"yes"`, `"true"`, `1`) is refused with a **400** `Invalid field in contract: hasRetenue (must be true, false or null)` and nothing is created. [Update Merchant](partner-update-merchant.md) accepts it with the same rules.

### Success

- **201**: `{ "merchantId": <number> }`

### Errors

- **400** invalid payload: `{ "ok": false, "error": "<what is wrong>" }`, e.g. `Invalid field in contract.sizeFees: small (allowed sizes: medium, large, extra_large)`. Nothing is created.
- **400** `merchant_cin_already_exists` / `merchant_trn_already_exists`: a merchant with this CIN or TRN already exists.
- **401** missing/invalid auth headers/signature
- **500** internal error

## Create Order

- **URL:** `https://livra.mofavo.com/create_order`
- **Method:** `POST`

### Request body

```json
{
  "products": [
    { "name": "string", "quantity": 1, "price": 12.5 },
    { "name": "string", "quantity": 2 }
  ],
  "productsToRetrieve": [
    { "name": "string", "quantity": 1 },
    { "name": "", "quantity": 0 }
  ],
  "merchantId": 1,
  "deliveryPartnerId": 1,
  "primaryName": "string",
  "primaryPhone": "string",
  "primaryPhone2": "",
  "primaryStreet": "",
  "primaryZone": "",
  "primaryCity": "string",
  "primaryState": "string",
  "primaryZipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false,
  "size": "medium",
  "callback_link": "https://your-app.example.com/livra/webhook"
}
```

`callback_link` is optional. Omit it to disable the per-order webhook for that order.

> **Delivery partners with a partner feed: don't send `callback_link`.** If your
> delivery partner already receives the [delivery-partner feed](#delivery-partner-feed-advanced),
> you get every change for all your orders there. Sending `callback_link` as well
> makes you receive each change a second time, in the
> [simple format](#per-order-webhook-simple-callback_link), with a different `X-Webhook-ID`.

### Rules

- `products` is required and must be a non-empty array.
- Each `products[]` item needs non-empty `name`, positive integer `quantity`, optional non-negative `price`.
- `merchantId` and `deliveryPartnerId` must be positive integers.
- `primaryName`, `primaryPhone`, `primaryCity`, `primaryState` must be non-empty.
- `amount` must be non-negative.
- `allowOpen`, `isExchange`, `isFragile` must be booleans.
- If `isExchange=true`, `productsToRetrieve` is expected. Invalid/missing entries still proceed with fallback name `"unknown"`.
- **`size` (optional):** the parcel's size, one of `small`, `medium`, `large`, `extra_large`. It decides which of the merchant's contract prices applies: `small` is the main contract price, and each other size uses the contract's price for that size. Leave it out (or send `null`) and the order is `small`, exactly as before. Any other value is refused with **400** `Invalid field: size (must be one of small, medium, large, extra_large)`, and no order is created.
- **`callback_link` (optional):** when present, Livra sends a signed `POST` to this URL whenever the order changes, in the **simple** format (see [Per-order webhook](#per-order-webhook-simple-callback_link)). Must be a valid absolute HTTP or HTTPS URL that accepts JSON `POST`s from Livra’s infrastructure.

### Success

- **201**: `{ "orderId": <number>, "hash": "<string>" }`
  - `hash` is the order-slip QR hash. A slip's QR code must encode `"<orderId>#<hash>"` to be accepted by depot scanners. The hash covers the order's content (recipient, products, amount, …), so after any order update the previous hash is stale — always use the latest one returned.

### Errors

- **400** one of:
  - `merchant_not_found`
  - `no_active_contract`
  - `sender_not_found`
  - validation errors
- **401** missing/invalid auth headers/signature
- **500** internal error

## Update Order

- **URL:** `https://livra.mofavo.com/update_order`
- **Method:** `POST`

One endpoint, two actions on an order:

| Action | Body | What it does |
|---|---|---|
| **Update** | no `action` | Patch-style edit of the order's fields while it waits for its pickup (`readyForPickUp`). After the pickup: the recipient's address and phones and the parcel's `size`, until the order is delivered, returned or cancelled (staging only for now); and switching it to an exchange or back to a regular order (`isExchange`). |
| **Cancel** | `"action": "cancel"` | Cancels the order before the driver collects it. |

If your API key is bound to a delivery partner, both actions reach only that partner's orders; any other order is answered with `order_not_found`, the same as one that doesn't exist.

Every response carries an `x-request-id` header. Quote it when you report a problem: it finds the one log line for your request.

### Update: request body

Patch-style payload. Only `orderId` is required; all other fields are optional.

```json
{
  "orderId": 1234,
  "products": [{ "name": "string", "quantity": 1 }],
  "productsToRetrieve": [{ "name": "string", "quantity": 1 }],
  "primaryName": "string",
  "primaryPhone": "string",
  "primaryPhone2": "",
  "primaryStreet": "",
  "primaryZone": "",
  "primaryCity": "string",
  "primaryState": "string",
  "primaryZipcode": "",
  "deliveryInstructions": "",
  "amount": 12.5,
  "allowOpen": true,
  "isExchange": false,
  "isFragile": false,
  "size": "small"
}
```

### Update: constraints

- `orderId` must exist.
- While the order is `readyForPickUp`, every field above can change.
- *(Staging only for now.)* The address, the phones and `size` can change at **any status until the order is delivered, returned or cancelled**: see [Update: address, phones and size after pickup](#update-address-phones-and-size-after-pickup).
- After the pickup, the only other change is the exchange switch: see [Update: exchange after pickup](#update-exchange-after-pickup). Any other change is rejected with `order_update_not_permitted`.
- `size` (staging only for now) is one of `small`, `medium`, `large`, `extra_large`, as in Create Order. It sets which of the contract's prices applies to the order. `medium`, `large` and `extra_large` need a sous-contrat for that size on the order's contract; without one the update is refused with **400** `size_not_in_contract` and nothing changes. `small` always works (the contract's base fees).
- Missing fields keep their current DB value.
- If a provided field has the same value, it is ignored (no rewrite).

### Update: address, phones and size after pickup

> **On staging only for now** (`https://staging.livra.mofavo.com/update_order`), for testing. On production, these fields still change only while the order is `readyForPickUp`, and `size` is not accepted on Update yet.

Until the order is **delivered, returned or cancelled**, you can correct where it goes, how to reach the customer, and the parcel's size, whatever its status (`inDepot`, `inTransit`, …):

- **Address:** `primaryStreet`, `primaryZone`, `primaryCity`, `primaryState`, `primaryZipcode`
- **Phones:** `primaryPhone`, `primaryPhone2`
- **Size:** `size`

```json
{
  "orderId": 1234,
  "primaryStreet": "Rue de Marseille 12",
  "primaryCity": "Tunis",
  "primaryState": "Tunis",
  "primaryPhone": "22123456",
  "size": "large"
}
```

- The API checks the order on **every call**. The order counts as finished when its status says so (`delivered`, `exchange-complete`, `returned`, `exchange-returned`, `cancelled`) **or** when it has a delivery, return or cancellation date. A finished order is refused with **400** `order_already_delivered`, `order_already_returned` or `order_cancelled`, with its `orderStatus`, and nothing changes.
- The same checks as Create Order: `primaryPhone`, `primaryCity` and `primaryState` can't be empty, and `size` must be one of the four sizes.
- *(Staging only for now.)* **The governorate (`primaryState`) can't change while the parcel is `inTransit`**: the parcel is on the road to the old governorate's depot and would not be re-routed. Such an update is refused with **400** `governorate_change_in_transit` (with `orderStatus`), and nothing changes; send it again once the parcel is `inDepot`. Sending the same governorate again is fine.
- After the pickup it needs a **delivery partner's key** (Livra's), and reaches only that partner's orders. Orders made in the merchant app, and orders handed over to another delivery partner (outsourced), are refused (`order_managed_in_merchant_app`, `order_outsourced`).
- You can send these fields together with an exchange switch. Any other field after the pickup (name, amount, products, instructions, …) is still rejected with `order_update_not_permitted`.
- The order's history shows each change (the size as the order type), who made it, when, and the order's status at that moment.
- **The slip on the parcel becomes outdated** when the address or a phone changes: they are part of the slip hash. The response carries the new `hash`; the next depot scan of the old slip asks for a reprint. `size` is not part of the hash.

### Update: exchange after pickup

An order the driver has already collected can still become an **exchange**, at any status up to delivered or returned. This is how Livra corrects swaps for customers whose first order another delivery company delivered: the swap was created as a regular order, and the driver did an exchange at the door. An exchange can also go back to a **regular** order while the parcel is still on its way to the customer.

The order's predecessor (`oldOrderId`) is an **external exchange** placeholder standing for the first sale: `delivered`, `source: externalExchange`, with no contract or partner, and the products to retrieve as its products. It is never picked up, invoiced or sent to a webhook feed, and the merchant is not notified of it.

Make it an exchange, before the driver reaches the customer (`inDepot`, or `inTransit` to a depot or to the customer):

```json
{
  "orderId": 1234,
  "isExchange": true,
  "productsToRetrieve": [{ "name": "T-shirt M", "quantity": 1, "price": 30 }]
}
```

Correct an order the driver has already been to the door with: say what happened there, and where the things travelling back are now:

```json
{
  "orderId": 1234,
  "isExchange": true,
  "productsToRetrieve": [{ "name": "T-shirt M", "quantity": 1 }],
  "exchangeOutcome": "happened",
  "itemsAt": "depot",
  "depotId": 12
}
```

Make it a regular order again (only before the door):

```json
{ "orderId": 1234, "isExchange": false }
```

| Field | When | Meaning |
|---|---|---|
| `exchangeOutcome` | **required** once the order is `delivered`, on its way back to the merchant, or `returned`; **refused** before that | `happened`: the customer swapped. `declined`: the customer refused, the new parcel goes back. |
| `itemsAt` | optional, only with `exchangeOutcome`; default `driver` | Where the things travelling back are now (the old products, or the refused parcel): `driver`, `depot` or `merchant`. Used for `delivered` orders. |
| `depotId` | required when `itemsAt` is `depot` | One of your depots. |

What each case does:

| The order is | You send | It becomes |
|---|---|---|
| `inDepot`, `inTransit` to the customer, waiting for a retry | no outcome | An exchange: the driver swaps or records the refusal at the door, as for any exchange. Status unchanged. |
| `delivered` | `happened`, `itemsAt: driver` | On its way back to the merchant (`inTransit`) with the old products. Ends `exchange-complete`, billed as an exchange. |
| `delivered` | `happened`, `itemsAt: depot` | `inDepot` in that depot, then back to the merchant. Ends `exchange-complete`. |
| `delivered` | `happened`, `itemsAt: merchant` | `exchange-complete` now, billed as an exchange. |
| `delivered` | `declined` (`driver` / `depot`) | Not delivered after all (`deliveryDate` cleared), on its way back (`inTransit` / `inDepot`). Ends `exchange-returned`, billed as a return. |
| `delivered` | `declined`, `itemsAt: merchant` | `exchange-returned` now, billed as a return. |
| on its way back to the merchant | `declined` | Still on its way; ends `exchange-returned` (same return fee). |
| `returned` | `declined` | `exchange-returned` now (same return fee). |

- It needs a **delivery partner's key** (Livra's), and reaches only that partner's orders. With any other key, an order past its pickup is refused with `order_update_not_permitted`, as before.
- Orders made in the merchant app, and orders handed over to another delivery partner (outsourced), can't be changed this way.
- After the pickup only `isExchange`, `productsToRetrieve`, `exchangeOutcome`, `itemsAt` and `depotId` are accepted, besides the address, phones and size (see [above](#update-address-phones-and-size-after-pickup)). Any other field is rejected with `order_update_not_permitted`, and nothing changes.
- `productsToRetrieve` lists what the customer gives back. Without it, one `unknown` product. Before the door, on an order that is already an exchange, sending it replaces the list.
- An order whose fee is already paid (settled with Livra, or paid to the merchant) can't be corrected: `order_already_settled`.
- The order's history shows the change: who made it, when, its status before and after, and the outcome.
- Your own correction is not sent back to your webhook feed (merchant webhooks still fire).
- **The slip on the parcel becomes outdated**: the products to retrieve are part of the slip hash. The response carries the new `hash`; the next depot scan of the old slip asks for a reprint.

### Update: success

- **200**: `{ "orderId": <number>, "hash": "<string>" }`
  - `hash` is the recomputed order-slip QR hash after the update (see Create Order). Any slip printed with an older hash must be reprinted.

### Update: errors

- **400** one of:
  - `order_not_found`
  - `order_update_not_permitted`: the order isn't `readyForPickUp` and the request carries a field that can't change after pickup (see above), or its status allows none (e.g. `missing`).
  - `order_already_delivered`: the order was already handed to the customer (by status or delivery date): no address, phone or size change, and no `isExchange: false`.
  - `order_already_returned`: the order is returned (by status or return date): no address, phone or size change. Also `isExchange: false` on an order on its way back to the merchant.
  - `order_already_exchange`: the order is already an exchange past the door (or finished as one); it can be corrected only once.
  - `order_already_settled`: the order's fee is already paid; it can't be corrected.
  - `exchange_outcome_required`: the order is past the door (delivered, on its way back, returned); send `exchangeOutcome`.
  - `exchange_outcome_not_allowed`: the order hasn't reached the customer yet; leave `exchangeOutcome` out.
  - `exchange_outcome_not_possible`: `happened` on an order that never reached the customer (on its way back, or returned).
  - `depot_not_found`: `depotId` isn't one of your depots.
  - `order_cancelled`: the order is cancelled (by status or cancellation date).
  - `order_managed_in_merchant_app`: the order was made in the merchant app.
  - `order_outsourced`: the order has been handed over to another delivery partner.
  - `governorate_change_in_transit` *(staging only for now)*: a new `primaryState` while the parcel is `inTransit`. Wait until it is `inDepot`.
  - `size_not_in_contract` *(staging only for now)*: the order's contract has no sous-contrat (price) for the `size` sent. Ask Mofavo to add one to the merchant's contract, or keep the current size.
  - Field errors, e.g. `Invalid field: size (must be one of small, medium, large, extra_large)`, `Invalid field: exchangeOutcome (must be happened or declined)`, `Invalid field: depotId (required when itemsAt is depot)`.
  - plus all create-order 400 errors

  These refusals (finished order, merchant-app, outsourced, exchange), and `order_update_not_permitted` for an exchange after pickup, also carry the order's current `orderStatus`, e.g. `{ "ok": false, "error": "exchange_outcome_required", "orderStatus": "delivered" }`. Don't retry them.
- **401** missing/invalid auth headers/signature
- **409** `order_busy`: the order was being changed at the same moment (e.g. a driver scanning it). Retry once after a second.
- **500** internal error

### Cancel: request body

Use it when the merchant cancels an order **before the driver has collected it**.

**Which orders can be cancelled:**
- Orders created through these APIs: Create Order, or External Create Order. Orders made in the merchant app are refused: their stock goes back to the merchant only through the app's own cancel.
- Orders still waiting for the pickup (`readyForPickUp`), or whose pickup was declined (`pickUp-declined`).
- Orders that haven't been handed over to another delivery partner (outsourced).
- The API key must be a delivery partner's key: Livra's. Other keys get a **403**.
- The order must belong to the merchant in `merchantId`. Another merchant's order is `order_not_found`, so a merchant can't cancel orders that aren't theirs.

```json
{
  "orderId": 1234,
  "merchantId": 56,
  "action": "cancel",
  "reason": "Customer changed their mind",
  "comment": "Called the customer on 05/10"
}
```

| Field | Required | Rules |
|---|---|---|
| `orderId` | yes | Positive integer. |
| `merchantId` | yes | Positive integer. The merchant the order belongs to: the `merchantId` the order was created with. Send the merchant who asks for the cancel, as your platform knows them, never a value the merchant typed. |
| `action` | yes | Exactly `"cancel"`. |
| `reason` | no | String, up to 255 characters. The cancellation reason, shown with the order. |
| `comment` | no | String, up to 1000 characters. Free text, shown with the order. |

- A cancel takes **only** these five fields. Any other field (e.g. `amount`) is a **400**: cancel and edit are never mixed in one request.
- `reason` and `comment` may be left out, sent as `null`, or sent blank: all three mean "not given". Longer text is cut to the limit.

### Cancel: what happens

- The order becomes **`cancelled`**, and its pending pickup is called off, so no driver is sent for it.
- A cancel is **not a return**. A returned order is one whose delivery was attempted, failed, and came back to the merchant (`orderStatus: "returned"`). A cancelled order never had a delivery attempt.
- A cancel before pickup is **free**: no cancellation fee is charged.
- The order's history shows an **order cancelled** event with the reason and comment.
- The Status API reports it as `orderStatus: "cancelled"`, `deliveryStatus: "cancelled"`.
- A cancelled order can't be edited or un-cancelled through the API.

### Cancel: success

- **200**:

  ```json
  {
    "ok": true,
    "orderId": 1234,
    "status": "cancelled",
    "cancelledAt": "2026-10-05T09:12:44.120Z",
    "alreadyCancelled": false
  }
  ```

- **Safe to retry.** Cancelling an order that is already cancelled is also a **200**, with `alreadyCancelled: true` and the original `cancelledAt` (`null` for an order cancelled before the cancellation date was recorded). Nothing changes. After a timeout or a network error, just send the cancel again.

### Cancel: errors

| Status | `error` | Meaning | What to do |
|---|---|---|---|
| **400** | validation message | Missing/invalid `orderId` or `merchantId`, an `action` other than `"cancel"`, a non-string `reason`/`comment`, or extra fields. | Fix the request. |
| **400** | `order_not_found` | No such order, or it isn't yours: another delivery partner's order, or an order of another merchant than `merchantId`. | Check the `orderId` and the `merchantId`. |
| **409** | `order_already_picked_up` | The driver has already collected the parcel. The body also has the order's current `orderStatus` (e.g. `"inTransit"`). | It can't be cancelled here any more. To change an order in the network, use Change Request. |
| **409** | `order_not_cancellable` | Not collected, but in a status this API can't cancel (e.g. `"readyForPackaging"`). The body has `orderStatus`. | Cancel it in the merchant app. |
| **409** | `order_managed_in_merchant_app` | The order was made in the merchant app, not through these APIs. | Cancel it in the merchant app. |
| **409** | `order_outsourced` | The order has been handed over to another delivery partner. | Contact operations. |
| **409** | `order_busy` | The order was being changed at the same moment (e.g. the driver was scanning it). | Retry once after a second. The answer will then be `200` or `order_already_picked_up`. |
| **401** | `Missing authentication headers` / `Invalid api key` / `Invalid signature` | Authentication failed. | Check `x-api-key` and how you sign the body. |
| **403** | `partner_scope_not_configured` | This API key isn't a delivery partner's key, so it can't cancel. | Use the delivery partner's key (ask `ops@mofavo.com`). |
| **500** | `internal_error` | Our error. | Retry later; send us the `x-request-id` if it persists. |

## Order Hash

Use this to fetch the current **order-slip QR hash** for an order — for example right before printing (or reprinting) a delivery slip. The QR must encode `"<orderId>#<hash>"`. The hash covers the order's content, so it changes whenever the order changes; a slip carrying an outdated hash is rejected at scan time.

Create Order and Update Order already return the same `hash` in their responses; this endpoint is for when you need it again later for an existing order.

- **URL:** `https://livra.mofavo.com/order_hash`
- **Method:** `POST`

### Request body

```json
{
  "orderId": 1234
}
```

### Rules

- `orderId` is required and must be a positive integer.

### Success

- **200**: `{ "orderId": <number>, "hash": "<string>" }`

### Errors

- **400** `order_not_found` or validation errors
- **401** missing/invalid auth headers/signature
- **500** internal error

## Change Request

- **URL:** `https://livra.mofavo.com/change_request`
- **Method:** `POST`
- **Auth:** same `x-api-key` + `x-signature` flow as other endpoints

### Request body

```json
{
  "orderId": 1234,
  "changes": [
    {
      "type": "PHONE_CHANGE",
      "oldValue": "50000000",
      "newValue": "51111111"
    },
    {
      "type": "AMOUNT_CHANGE",
      "oldValue": 110,
      "newValue": 125.5
    }
  ],
  "comment": "Customer requested phone and amount correction",
  "makeRegular": false
}
```

### Rules

- `orderId` must be a positive integer.
- `changes` must be a non-empty array. One request can carry one change or several of
  different types (e.g. a phone and an amount together, as in the example above).
- Each change requires `type`, one of the types below, and a `newValue` of the type shown.
  Values are checked strictly: `"79"` is not an amount, and `"true"` is not a boolean.

  | `type` | `newValue` | Example |
  |---|---|---|
  | `AMOUNT_CHANGE` | number, `>= 0` | `79`, `125.5` |
  | `ALLOW_OPEN_CHANGE` | boolean | `true` |
  | `PHONE_CHANGE` | string: optional `+`, then 8 to 15 digits. Spaces and dashes are allowed and removed before saving (`"51 111 111"` and `"51-111-111"` are saved as `"51111111"`). Dots, brackets, tabs and other characters are rejected. | `"51111111"` |
  | `PHONE2_CHANGE` | same as `PHONE_CHANGE`, or `""` to remove the second phone. `null` is rejected: send `""` instead. | `"51111111"`, `""` |
  | `DELIVERY_DATE_CHANGE` | the planned delivery day, as a `"YYYY-MM-DD"` date that exists, today or later (Tunisia time). A past date is rejected. `null` clears the planned date. | `"2026-10-02"`, `null` |
  | `ADDRESS_CHANGE` | an object with any of `street`, `zone`, `city`, `state`, `zipcode`. Each value is a string or `null`; strings are trimmed before saving and may be at most 255 characters after trimming. A value of only spaces is rejected. At least one field must be non-empty. Any other key is rejected. | `{ "street": "12 Rue X", "zipcode": "2036" }` |

- In `ADDRESS_CHANGE`, a `null` or `""` field is **ignored** when the request is approved, not
  cleared: the order keeps its current value. Send only the fields that change.
- `oldValue` is optional, is not checked, and is only shown to whoever reviews the request.
  Use the same type as `newValue` so it displays well.
- Each `type` may appear only once per request. To change several fields, put all of them in
  the **same** request rather than sending them one by one: a new request for the order replaces
  its pending one, so sending them separately would keep only the last.
- `comment` is optional string.
- `makeRegular` is optional boolean.
- If your API key is linked to a delivery partner, the order must belong to that partner. Any
  other order is answered with `order_not_found`, exactly as if it didn't exist.
- Order must currently be in `inDepot` or `inTransit` status.
- If the order is exchange-linked and `deliveryDate` is set (exchange already happened), request creation is blocked.

### Success

- **201**: `{ "ok": true, "id": "<uuid>" }` *(the `id` is on staging only for now)*
  - `id` is the change request's id. Keep it: it's how you approve or reject this request later (see [Approve or reject a request](#approve-or-reject-a-request)). It is the same `change_request.id` the `change_request.approved` / `change_request.rejected` webhooks carry.

### Errors

- **400** one of:
  - `order_not_found` (also for an order that belongs to another delivery partner)
  - `order_status_not_eligible_for_change_request`
  - `exchange_already_completed_change_request_not_allowed`
  - `no_changes_provided`
  - `body is not valid JSON`
  - validation errors. Each names the change and says what to send, for example:
    - `changes[0].newValue must be a number, not a string: send 79, not "79"`
    - `changes[0].newValue must be true or false without quotes: send true, not "true"`
    - `changes[0].newValue must be a phone number like "51111111"`
    - `changes[0].newValue is in the past (today is 2026-10-02)`
    - `changes[0].newValue.primaryStreet is not an address field (use street, zone, city, state, zipcode)`
    - `changes[1] repeats AMOUNT_CHANGE from changes[0]`
- **401** missing/invalid auth headers/signature
- **500** internal error

### Approve or reject a request

> **On staging only for now** (`https://staging.livra.mofavo.com/change_request`), for testing. On production, the create response is still `{ "ok": true }` and `action` is not accepted yet.

When the merchant's request was decided on your platform, send the decision so the order is updated on our side too. Same URL and authentication; the body carries an `action` instead of `changes`:

```json
{ "action": "approve", "id": "6f1c2a9e-3b7d-4c41-9e2a-0d5b8f7a1c34", "employeeId": 107 }
```

| Field | Required | Meaning |
|---|---|---|
| `action` | yes | `approve` applies the requested changes to the order; `reject` applies nothing. |
| `id` | yes | The request's `id` from the create response. |
| `employeeId` | no | **Our** id of your employee who decided (the same id the settlement webhooks send as `agentId`). We show their name in the order history. |
| `employeeName` | no | Their name, used when `employeeId` is unknown to us. |

Only these four keys are accepted with an `action`. It needs a **delivery partner's key**, and reaches only that partner's orders.

**Rules**

- You can decide a request that is **pending**, that was filed **for a merchant** (through this API, or by the merchant in our app), on **one of your orders**, while the order is **`inDepot` or `inTransit`**, and its exchange has not already happened at the door.
- **The first decision wins.** Your team can still decide the same request in the dashboard. Whichever decision lands first is final; the other one gets "already decided" and changes nothing.
- Approving applies the changes exactly as a dashboard approval does (for `ADDRESS_CHANGE`, `null` or `""` fields are ignored; `makeRegular` turns the order back into a regular one).
- The `change_request.approved` / `change_request.rejected` webhook is sent **once**, whichever side decided, including when you did.
- The order's history records the decision, who made it (your employee's name, or "Partner API"), and when.

**Success**

- **200**: `{ "ok": true, "id": "<uuid>", "status": "approved" | "rejected", "skipped": [<string>, …], "hash": "<string>" | null }`
  - `skipped` lists the requested changes that were **not** applied. An approve can be partial: the request is approved and every other change is applied, but these weren't. Values: the change type when the order can't take it (e.g. `"FOO_CHANGE"`), `"AMOUNT_CHANGE:not-a-number"` when the new amount isn't a number, `"DELIVERY_DATE_CHANGE:not-a-date"` when the new date isn't a date. An empty list means everything was applied; it is always empty on `reject`.
  - `hash` is the order's slip hash after the decision. An approved address or phone change changes it: reprint the slip. In the rare case it can't be computed, `hash` is `null` but the decision **is saved**: don't retry the decision, fetch the hash with [Order Hash](#order-hash) instead.

**Errors**

- **409** `change_request_not_pending`: the request was already decided, e.g. `{ "ok": false, "error": "change_request_not_pending", "status": "approved" }`. If `status` is the decision you sent, your earlier call went through (for example a retry after a timeout): treat it as done. Otherwise the request was decided the other way, in the dashboard.
- **409** `order_busy`: the order was being changed at the same moment (e.g. a driver scanning it). Retry once after a second.
- **400** one of:
  - `change_request_not_found`: no such request, it was replaced by a newer request on the same order, or it isn't on one of your orders.
  - `change_request_source_not_allowed`: the request was filed by a driver or from the delivery app; those are for the merchant to decide.
  - `order_status_not_eligible_for_change_request`: the order is no longer `inDepot` / `inTransit` (carries `orderStatus`).
  - `exchange_already_completed_change_request_not_allowed`: the exchange already happened at the door (carries `orderStatus`).
  - `no_applicable_changes`: `approve` on a request with nothing the order can take.
  - Field errors: `Invalid field: action (must be approve or reject)`, `Missing or invalid field: id (must be a uuid)`, `Unexpected field with action: <key>`, `Invalid field: employeeId (must be a positive integer)`, `Invalid field: employeeName (must be a string)`.
- **403** `partner_scope_not_configured`: the API key isn't bound to a delivery partner.
- **401** missing/invalid auth headers/signature
- **500** internal error

When the request is approved or rejected, your delivery-partner feed receives `change_request.approved` or `change_request.rejected`. See [Change request events](webhooks-preview.md#change-request-events).

## Order status webhooks

Livra sends order webhooks through two separate channels, each with its own format:

| Channel | How you get it | Covers | Format (`X-Webhook-Type` / `X-Webhook-Version`) |
| --- | --- | --- | --- |
| [Delivery-partner feed](#delivery-partner-feed-advanced) | Configured by the platform team for your delivery partner (one callback URL) | **Every** order of your delivery partner | `advanced` / `1` |
| [Per-order webhook](#per-order-webhook-simple-callback_link) | `callback_link` on [create order](#create-order) | Only that order | `simple` / `1` |

The two are independent. If an order has both, **each change is sent twice**:
once per channel, with a different `X-Webhook-ID`. If you already receive the
delivery-partner feed, don't send `callback_link`.

Always branch on the `X-Webhook-Type` and `X-Webhook-Version` headers before
parsing the body.

---

### Delivery-partner feed (advanced)

**Current version: 1**

The raw event payload as recorded by the platform. Each delivery represents one discrete change, with an explicit event name (`event`), a full snapshot of the order (`data`), and the previous values (`previous`). Driver outcomes are separate `driver.*` events.

This feed does **not** receive the changes you make yourself through the partner APIs ([Accept In Depot](partner-accept-in-depot.md), [Transfers](partner-transfers.md), [Returns](partner-return.md)): you already know about them. Changes made on our side still reach you.

This feed also receives a `status.changed` as soon as an order becomes a Livra order: when it is created as `readyForPackaging` or `readyForPickUp`, or when its contract is switched to Livra. See [New orders for Livra](webhooks-preview.md#new-orders-for-livra).

It also receives `order_type.changed` when our depot changes a parcel's size ([Size changes](webhooks-preview.md#size-changes)), and `change_request.approved` / `change_request.rejected` when a change request is decided ([Change request events](webhooks-preview.md#change-request-events)).

Full documentation: [Livra Webhooks — Advanced](webhooks-preview.md)

---

### Per-order webhook (simple, `callback_link`)

**Current version: 1**

Sent to the `callback_link` of an order, on the same changes as the advanced
feed (status, final destination, position, delivery date, driver outcomes and
invoice settlement). Instead of the raw event, each request carries a compact,
pre-computed status of the order **at the time it is sent**. There is no event
name, so the same status can arrive more than once (for example at delivery and
again at settlement).

```
POST <callback_link>
Content-Type: application/json
X-Webhook-ID: <delivery-uuid>
X-Webhook-Signature: <hmac-hex>
X-Webhook-Type: simple
X-Webhook-Version: 1
```

```json
{
  "ok": true,
  "timestamp": "2026-10-15T10:08:13.897915Z",
  "orders": [
    {
      "id": 4817459,
      "deliveryStatus": "pending",
      "orderStatus": "inDepot",
      "comment": null
    }
  ]
}
```

| Field | Meaning |
| --- | --- |
| `timestamp` | When the change happened (UTC). Unchanged on retries. |
| `orders[].id` | The order id returned by create order. Always one order per request. |
| `orders[].deliveryStatus` | Overall outcome, see below. |
| `orders[].orderStatus` | Where the order is now, see below. |
| `orders[].comment` | `null`, except for driver outcomes: `declined`, `unreachable` or `rescheduled`. |

`deliveryStatus`:

| Value | When |
| --- | --- |
| `pending` | Still in progress (not delivered, heading to the customer). |
| `delivered` | Delivered to the customer (`deliveryDate` is set). |
| `cancelled` | Not delivered and heading back to, or back at, the merchant. |

`orderStatus` is the order's status (same values as [`status`](webhooks-preview.md#status)
in the advanced feed), except that `inTransit` is split by direction:

| Value | Meaning |
| --- | --- |
| `inTransitToCustomer` | On the road toward the customer. |
| `inTransitToMerchant` | On the road back to the merchant. |

Examples:

| Payload | Meaning |
| --- | --- |
| `pending` + `inDepot` | Parcel held in a depot, still going to the customer. |
| `pending` + `inTransitToCustomer` | Out for delivery. |
| `delivered` + `delivered` | Delivered to the customer. |
| `cancelled` + `inTransitToMerchant` | Being returned to the merchant. |
| `pending` + `inTransitToCustomer` + `comment: "unreachable"` | Driver couldn't reach the customer. |

---

### Shared delivery mechanics

The following applies to all Livra webhooks, advanced and simple.

#### Verifying signatures

Every request includes an `X-Webhook-Signature` header containing an **HMAC-SHA256** of the **raw request body**, hex-encoded, using your **Livra API secret** (the same secret you use to sign requests to Livra).

Always verify this header before processing the payload.

**Node.js**

```js
const crypto = require('crypto');

function verifySignature(secret, rawBody, signature) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(rawBody)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(expected),
    Buffer.from(signature)
  );
}

// Express example
app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const sig = req.headers['x-webhook-signature'];
  if (!verifySignature(process.env.WEBHOOK_SECRET, req.body, sig)) {
    return res.status(401).send('Invalid signature');
  }
  const event = JSON.parse(req.body);
  // process event...
  res.sendStatus(200);
});
```

**Python**

```python
import hmac, hashlib

def verify_signature(secret: str, raw_body: bytes, signature: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        raw_body,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)
```

**Go**

```go
import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
)

func verifySignature(secret, signature string, body []byte) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(body)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}
```

> **Important:** always read the raw request body for signature verification. Parsing the JSON first and re-serialising it may produce a different byte sequence and cause verification to fail.

#### Responding to events

Reply with any **2xx status code** to acknowledge successful delivery. The response body is ignored.

If your endpoint returns a non-2xx status or does not respond within **10 seconds**, the delivery is retried automatically.

#### Retry schedule

| Attempt | Delay before retry |
| --- | --- |
| 1 | 30 seconds |
| 2 | 5 minutes |
| 3 | 30 minutes |
| 4 | 2 hours |
| 5 | 8 hours |

After 5 failed attempts the delivery is marked permanently failed and no further retries are made. The platform team can manually re-queue a delivery on request.

#### Identifying deliveries

Each delivery has a unique UUID in the `X-Webhook-ID` header. Use it to deduplicate events if your endpoint receives the same delivery more than once.
