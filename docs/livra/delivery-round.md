# Delivery Round — how Last Mile, Mission Status and Validation work together

[← Back to Livra integration guide](README.md)

> **Staging only for now.** [Mission Status](partner-mission-status.md),
> [Validation](partner-validation.md) and the new `return` rules described here are live on
> **staging** (`https://staging.livra.mofavo.com`) for you to build and test against. They
> are not in production yet: we will tell you when they are. Until then, production has
> [Last Mile](partner-last-mile.md) only, with its current `deliver`, `return` and `retry`.

Read this page first. It explains **one driver's round**, from the depot to the customer and
back, and which of the three APIs you call at each moment. Each API then has its own page
with every field and error.

| API | One sentence | Page |
| --- | --- | --- |
| **Last Mile** `POST /partner_last_mile` | The **sheet**: which parcels a driver takes, and when he leaves with them. | [Last Mile](partner-last-mile.md) |
| **Mission Status** `POST /partner_mission_status` | What the **driver records at the customer**: delivered, unreachable, postponed, refused… | [Mission Status](partner-mission-status.md) |
| **Validation** `POST /partner_validation` | **Where a parcel goes** (to the customer again, or back to the merchant), and the driver **coming back** to the depot. | [Validation](partner-validation.md) |

All three use the same API key, the same `x-signature`, the same order ids and the same
`employeeId`. You never send your partner id: we read it from your key.

## Contents

- [1. Who does what](#1-who-does-what)
- [2. The round, step by step](#2-the-round-step-by-step)
- [3. After each driver result: what comes next](#3-after-each-driver-result-what-comes-next)
- [4. The return to the depot: the `cancelled` scan button](#4-the-return-to-the-depot-the-cancelled-scan-button)
- [5. A whole day, call by call](#5-a-whole-day-call-by-call)
- [6. The parcel's status at each step](#6-the-parcels-status-at-each-step)
- [7. Rules that hold everywhere](#7-rules-that-hold-everywhere)
- [8. Before you go live](#8-before-you-go-live)

## 1. Who does what

Every call mirrors something a person does in your own tools. That person decides which API:

| Who | What they do | You call |
| --- | --- | --- |
| **Depot employee** (your `employeeId`) | Loads parcels for a driver, hands them over, takes back what he brings back. | Last Mile `create` / `add` / `remove` / `collect`, then Validation `return` |
| **Driver** (your `driverId`), in your driver app | At each stop, records what happened. | Mission Status, **right away** (or as soon as the app is back online) |
| **Customer service / quality control** (an employee) | Calls the customer or the merchant and decides where a parcel goes. | Validation `retry` or `return_to_merchant` |
| **The merchant** | Is told about every change, and asked what to do when a customer refuses. | Nothing: we send the merchant's webhooks and open the check for you |
| **The customer** | Gets an SMS on some results (per the contract). | Nothing: we send it |
| **Us (Mofavo)** | Keep each order's status, timeline, invoices and the merchant's settlement. | — |

## 2. The round, step by step

```
 IN YOUR DEPOT                     ON THE ROAD                              BACK IN THE DEPOT
 ─────────────                     ───────────                              ─────────────────
 Last Mile                         Mission Status (per stop)                Validation
 create ─► add ─► (remove) ─►      delivered ─────────────────────────────► (nothing: done)
 collect ─────────────────────►    unreachable / postponed / refused… ───► return
                                        ▲          │
                                        │          ▼
                                   Validation retry / return_to_merchant
                                   (customer service decides while the
                                    driver still has the parcel)
```

| # | When | Who | Call | Result |
| --- | --- | --- | --- | --- |
| 1 | You plan a driver's round (today or tomorrow) | Depot | Last Mile `create` | An empty sheet for one depot, one driver, one day. |
| 2 | You load parcels for him | Depot | Last Mile `add` (and `remove`) | The parcels wait in the depot, on the sheet. |
| 3 | He leaves with them | Depot | Last Mile `collect` | Every parcel on the sheet is **out for delivery** with him. |
| 4 | At each customer | Driver | **Mission Status** | `delivered` ends it. Any other result keeps the parcel with the driver. |
| 5 | (Optional) after calling the customer or merchant | Customer service | Validation `retry` / `return_to_merchant` | The parcel goes to the customer after all, or back to the merchant. |
| 6 | He is back with what he couldn't deliver | Depot | Validation `return` | Each parcel is checked back into the depot, and goes where its result says. |
| 7 | Every parcel is delivered or returned | — | nothing | The sheet **closes itself**. |

Then, for what came back: put it on a **new sheet** (a parcel still for the customer), or send
it back to the merchant with [Returns](partner-return.md).

## 3. After each driver result: what comes next

| Driver result (Mission Status) | The parcel now | What comes next |
| --- | --- | --- |
| `delivered` | Delivered. Counted in the merchant's settlement. | Nothing. (An **exchange** is swapped at the door: the driver brings the old item back, see [§4](#4-the-return-to-the-depot-the-cancelled-scan-button).) |
| `unreachable` | With the driver, still for the customer. A check opens (or the customer gets an SMS). | Another result later today, or back in the depot: `return`, then `add` it to a new sheet. |
| `postponed` with a `date` | With the driver, still for the customer, planned for that day. | Back in the depot: `return`, then `add` it to that day's sheet. |
| `postponed` without a `date` | Unchanged: it's a **comment** on the timeline. | Another result later. |
| `wrong_phone` | With the driver. A "wrong phone" check opens. | When fixed: another result. Otherwise `return`. |
| `declined` (refused) | With the driver, **still for the customer for now**. The merchant is asked what to do. | Customer service decides: `retry` (deliver after all) or `return_to_merchant`. If the driver comes back first: `return`, and the parcel **goes back to the merchant**. |
| `missing` | Marked missing, for your depot to look into. The delivery itself is still "out for delivery". | When found: send its real result (e.g. `delivered`, or `unreachable` then `return`). |
| `reset` | Back to plain "out for delivery". Pending checks close. | Undoes a mistake. Send the right result next. |

A parcel can get **several results in one day** (unreachable at 10:00, delivered at 16:00).
Each one is on its timeline; the last one is its current status.

## 4. The return to the depot: the `cancelled` scan button

Validation `return` is exactly your depot's check-in of a driver's parcels. Two rules:

1. **The driver's result comes first.** A parcel with no result (still plainly "out for
   delivery") is refused: `still_in_delivery`. Send its result to Mission Status, then return it.
2. **`cancelled` is the scan button, and must match the parcel as it is now:**

| The parcel is going… | Typical case | Send | Then it goes |
| --- | --- | --- | --- |
| **to the customer** | unreachable, postponed, wrong phone, **and refused** (the refusal doesn't change where it goes; the return does) | `"cancelled": false` ("Retour") | refused → **back to the merchant**; otherwise still for the customer |
| **to the merchant** | after Validation `return_to_merchant`, or the old item of an exchange swapped at the door | `"cancelled": true` ("annulé/échange") | back to the merchant |

The wrong button is refused (`should_be_return` / `should_be_merchant`) and nothing changes,
so you can't send a parcel the wrong way by mistake.

## 5. A whole day, call by call

Driver `42`, depot `3`, employee `7`, four parcels: `101` will be delivered, `102` is
unreachable, `103` is refused, `104` is an exchange.

**Morning, in the depot (Last Mile):**

```json
{ "action": "create", "reference": "LM-2026-001200", "depotId": 3, "driverId": 42, "date": "2026-11-20", "employeeId": 7, "orderIds": [101, 102, 103, 104] }
{ "action": "collect", "reference": "LM-2026-001200", "employeeId": 7 }
```

**On the road (Mission Status), as the driver records each stop:**

```json
{ "reference": "LM-2026-001200", "driverId": 42, "results": [
  { "orderId": 101, "status": "delivered", "at": "2026-11-20T10:05:00+01:00" },
  { "orderId": 102, "status": "unreachable", "at": "2026-11-20T10:40:00+01:00" },
  { "orderId": 103, "status": "declined", "reason": "cancelByClient", "at": "2026-11-20T11:15:00+01:00" },
  { "orderId": 104, "status": "delivered", "at": "2026-11-20T12:00:00+01:00" }
] }
```

`104` is an exchange: "delivered" swaps it, and the driver now carries the old item.

**Evening, the driver is back (Validation):**

```json
{ "action": "return", "reference": "LM-2026-001200", "employeeId": 7, "orders": [
  { "orderId": 102, "cancelled": false },
  { "orderId": 103, "cancelled": false },
  { "orderId": 104, "cancelled": true }
] }
```

- `102` is in the depot, still for the customer: `add` it to tomorrow's sheet.
- `103` is in the depot, going back to the merchant: send it with [Returns](partner-return.md)
  (or Validation `retry` if the merchant wants another attempt).
- `104`'s old item is in the depot, going back to the merchant.
- Every parcel is delivered or returned: the sheet **closes itself**.

## 6. The parcel's status at each step

What [Get Order](partner-get-order.md) and the merchant's webhooks show:

| After | `status` | Going to |
| --- | --- | --- |
| Last Mile `add` | `inDepot` | the customer |
| Last Mile `collect` | `inTransit` | the customer |
| Mission Status `delivered` | `delivered` | — |
| Mission Status `delivered`, exchange | `inTransit` (old item with the driver) | the merchant |
| Mission Status `unreachable` / `postponed` / `wrong_phone` / `declined` | `inTransit` | the customer |
| Mission Status `missing` | `missing` | the customer |
| Validation `return_to_merchant`, driver still has it | `inTransit` | the merchant |
| Validation `return`, after unreachable / postponed | `inDepot` | the customer |
| Validation `return`, after refused | `inDepot` | the merchant |
| Validation `retry` | unchanged (`inTransit` or `inDepot`) | the customer |

Your own calls don't come back to your webhook feed (you already know); the merchant's
webhooks and the customer SMS still go out.

## 7. Rules that hold everywhere

- **Every call can be sent again.** A lost answer or a timeout: send the same call. What's
  already done is answered `ok` (Last Mile, Validation) or `duplicate` (Mission Status) and
  never written twice.
- **Per-parcel answers.** A `200` with `"ok": false` means some parcels were refused (listed
  with a reason); **the others were applied**. Read the list.
- **Offline driver app.** Send each result with `at` (when it happened). Results are applied
  in the order of `at`; an older one arriving late is answered `late` and only added to the
  timeline.
- **One sheet per round, one `reference` per sheet**, the same on every call for it.
- **Never deliver a parcel going to the merchant.** Mission Status refuses it
  (`not_for_customer`): send Validation `retry` first.
- Log the **`x-request-id`** header of every answer: it's what we ask for if you need help.

## 8. Before you go live

- [ ] On **staging**: one full round with all three APIs, until the sheet closes by itself.
- [ ] Your driver app sends each result to Mission Status **as it happens**, with `at`.
- [ ] Every parcel the driver brings back gets Validation `return`, after its result, with
      the right `cancelled` button.
- [ ] Customer service decisions go to Validation `retry` / `return_to_merchant`.
- [ ] You read `ok`, and the per-parcel list when it's `false`.
- [ ] You no longer call Last Mile `deliver`, `return` or `retry` (they're replaced by the two
      new APIs; see [Moving from Last Mile](partner-validation.md#8-moving-from-last-mile)).

[← Back to Livra integration guide](README.md)
