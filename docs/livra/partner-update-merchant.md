# Update Merchant — Partner API

[← Back to Livra integration guide](README.md)

> **Available on staging now, for testing.** Production comes after your tests: we will
> tell you when it is live. Until then, call the staging URL below.

This page is for **delivery partners**. It explains one endpoint: `POST /update_merchant`,
which updates a merchant you created with [Create Merchant](README.md#create-merchant):
its information, its sender and its contract (fees, sous-contrats per size, 3% retenue).

You send **the same body as Create Merchant**, plus the `merchantId` Create Merchant
returned. Each part (`merchant`, `sender`, `contract`) is optional: **only the parts you send
are updated**, the others are not touched.

## Contents

- [1. What you need before you start](#1-what-you-need-before-you-start)
- [2. The request](#2-the-request)
- [3. What each part updates](#3-what-each-part-updates)
- [4. How to build `x-signature`](#4-how-to-build-x-signature)
- [5. Copy-paste example](#5-copy-paste-example)
- [6. The success response](#6-the-success-response)
- [7. Every error, and what to do about it](#7-every-error-and-what-to-do-about-it)
- [8. Checklist before you go live](#8-checklist-before-you-go-live)

## 1. What you need before you start

- Your **delivery partner API key and secret**: the ones you use for the other partner APIs
  (Merchant Settlement, Last Mile…). Staging has its own key and secret (ask
  `ops@mofavo.com`). Your production key doesn't work on staging, and your staging key
  doesn't work in production.
- The **`merchantId`** of the merchant, as Create Merchant returned it.

> **You do not send your partner id to choose the merchant.** We read it from your API key.
> You can only ever update **your own** merchants: merchants with an **active** contract
> with you.

## 2. The request

- **URL (staging):** `https://staging.livra.mofavo.com/update_merchant`
- **URL (production, once live):** `https://livra.mofavo.com/update_merchant`
- **Method:** `POST`

### Headers

```
Content-Type: application/json
x-api-key: <your api key>
x-signature: <see section 4>
```

### Body

```json
{
  "merchantId": 123,
  "merchant": {
    "name": "Example Merchant LLC",
    "state": "Tunis",
    "city": "Tunis",
    "street": "Example Street 1",
    "phoneNumber": "+21620000000",
    "zipcode": "1000",
    "TRN": "1234567A",
    "CIN": "01234567"
  },
  "sender": {
    "name": "Example Sender LLC",
    "state": "Tunis",
    "city": "Tunis",
    "street": "Example Street 2",
    "phoneNumber": "+21620000001"
  },
  "contract": {
    "deliveryPartnerId": 77,
    "deliveryFee": 7.5,
    "exchangeFee": 5,
    "cancellationFee": 4,
    "hasRetenue": true,
    "sizeFees": {
      "large": { "deliveryFee": 10.5, "exchangeFee": 8, "cancellationFee": 4 }
    }
  }
}
```

| Field | Required | What it is |
|---|---|---|
| `merchantId` | yes | The merchant to update: the id Create Merchant returned. |
| `merchant` | no | The merchant's information. Same fields and rules as Create Merchant. |
| `sender` | no | The merchant's sender. Same fields and rules as Create Merchant. |
| `contract` | no | Your contract with the merchant. Same fields and rules as Create Merchant, plus `hasRetenue`. |

Send **at least one** of `merchant`, `sender` and `contract`.

## 3. What each part updates

**Each part you send is checked exactly like in Create Merchant** (same required fields,
same CIN format, non-negative fees, allowed sizes) and gets the same error messages. A part
you send must be complete: to change one fee, send the whole `contract` part.

### `merchant`

- Replaces the merchant's information with the one you send.
- `TRN` and `CIN`: one you **leave out keeps its current value**; one you send as `null` is
  **cleared**. As in Create Merchant, the part must contain at least one of them.
- A **new** TRN or CIN that another merchant already uses is refused
  (`merchant_trn_already_exists`, `merchant_cin_already_exists`).

### `sender`

- Updates the sender created with the merchant.

### `contract`

- `deliveryPartnerId` must be **your** delivery partner id.
- The main fees (`deliveryFee`, `exchangeFee`, `cancellationFee`) are replaced by the ones
  you send.
- `sizeFees`: each size you send (`medium`, `large`, `extra_large`) is updated, or added when
  the contract has no price for it yet. **A size you leave out keeps its price.**
- **A fee change only applies to orders created from now on.** Orders created before keep
  the price they were created with. Sending the same fees again changes nothing.
- `hasRetenue`: whether the 3% retenue à la source applies to this merchant's orders.

| `hasRetenue` | Effect |
|---|---|
| `true` | The 3% retenue applies. |
| `false` | No retenue. |
| `null` | Decided automatically: the retenue applies when the merchant has no matricule fiscal (a TRN with letters, such as `1935742/N/N/M/000`). |
| left out | Unchanged. |
| anything else (`"yes"`, `"true"`, `1`) | Refused with a **400**. |

**Everything is saved together.** If one part is refused, nothing is changed, not even the
parts that were valid.

## 4. How to build `x-signature`

`x-signature` is an HMAC-SHA256 of the **request body**, keyed with your **API secret**,
written as lowercase hex (`sha256=<hex>` is also accepted).

```
x-signature = HMAC_SHA256( raw request body , apiSecret )  →  lowercase hex
```

Sign the **exact bytes you send**: build the JSON string once, sign that string, and send
that same string as the body.

## 5. Copy-paste example

Change only the contract's fees and the retenue:

```bash
API_KEY="your-staging-api-key"
API_SECRET="your-staging-api-secret"
BODY='{"merchantId":123,"contract":{"deliveryPartnerId":77,"deliveryFee":7.5,"exchangeFee":5,"cancellationFee":4,"hasRetenue":true}}'
SIGNATURE=$(printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" -hex | sed 's/^.* //')

curl -s -X POST "https://staging.livra.mofavo.com/update_merchant" \
  -H "Content-Type: application/json" \
  -H "x-api-key: $API_KEY" \
  -H "x-signature: $SIGNATURE" \
  --data "$BODY"
```

## 6. The success response

**200**

```json
{ "merchantId": 123 }
```

## 7. Every error, and what to do about it

Every error is `{ "ok": false, "error": "<code or message>" }`. Nothing is changed.

| Status | `error` | What it means | What to do |
|---|---|---|---|
| 400 | a sentence, e.g. `Invalid field in contract: hasRetenue (must be true, false or null)` | A field is missing or wrong. Same messages as Create Merchant. | Fix the field named in the message. |
| 400 | `Nothing to update: send at least one of merchant, sender or contract` | Only `merchantId` was sent. | Send the part you want to change. |
| 400 | `merchantId must be a positive integer` | `merchantId` is missing or not a number. | Send the id Create Merchant returned, as a number. |
| 400 | `merchant_trn_already_exists` / `merchant_cin_already_exists` | Another merchant already uses this new TRN / CIN. | Check the value with the merchant. |
| 400 | `contract_partner_mismatch` | `contract.deliveryPartnerId` is not your partner id. | Send your own partner id. |
| 401 | `Missing authentication headers` / `Invalid api key` / `Invalid signature` | Authentication failed. | Check the key, and sign the exact body you send (section 4). |
| 403 | `partner_scope_not_configured` | This key is not linked to a delivery partner. | Use your delivery partner key, or ask `ops@mofavo.com`. |
| 404 | `merchant_not_found` | No such merchant, or it has no active contract with you. | Check `merchantId`. |
| 500 | `internal_error` | Our side failed. Nothing was changed. | Retry later; contact us if it persists. |

Sending the same update twice is safe: the second call changes nothing.

## 8. Checklist before you go live

- [ ] Update one part at a time on staging (`merchant`, then `sender`, then `contract`) and
      check the other parts did not change.
- [ ] Change a fee, then create an order: it uses the new price; an order created before
      keeps the old one.
- [ ] Try `hasRetenue` `true`, `false`, `null`, and leave it out.
- [ ] Handle every error in section 7, especially the 400 messages (show them to your user).
