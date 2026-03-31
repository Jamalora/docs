# Livra Integration Guide (Production)

This document describes how to call Livra integration endpoints from your app.

## Shared authentication headers

Use these headers for both endpoints:

- `Content-Type: application/json`
- `x-api-key: <apiKey>`
- `x-signature: <hexHmac>`

`x-signature` must be `HMAC-SHA256(rawRequestBody, apiSecret)` encoded as lowercase hex (optionally prefixed with `sha256=`).

## Create Merchant

- **URL:** `https://livra.mofavo.com/create_merchant`
- **Method:** `POST`

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
    "CIN": 12345678
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
    "cancellationFee": 3.0
  }
}
```

### Rules

- `merchant.CIN` must be an 8-digit integer.
- All merchant/sender string fields must be non-empty.
- `contract.deliveryPartnerId` must be a positive integer.
- Contract fees must be non-negative numbers.

### Success

- **201**: `{ "merchantId": <number> }`

### Errors

- **400** invalid payload
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
  "isFragile": false
}
```

### Rules

- `products` is required and must be a non-empty array.
- Each `products[]` item needs non-empty `name`, positive integer `quantity`, optional non-negative `price`.
- `merchantId` and `deliveryPartnerId` must be positive integers.
- `primaryName`, `primaryPhone`, `primaryCity`, `primaryState` must be non-empty.
- `amount` must be non-negative.
- `allowOpen`, `isExchange`, `isFragile` must be booleans.
- If `isExchange=true`, `productsToRetrieve` is expected. Invalid/missing entries still proceed with fallback name `"unknown"`.

### Success

- **201**: `{ "orderId": <number> }`

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

### Request body

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
  "isFragile": false
}
```

### Constraints

- `orderId` must exist.
- Existing order status must still be `readyForPickUp`; otherwise update is rejected.
- Missing fields keep their current DB value.
- If a provided field has the same value, it is ignored (no rewrite).

### Success

- **200**: `{ "orderId": <number> }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_update_not_permitted`
  - plus all create-order 400 errors
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
      "oldValue": "+971500000000",
      "newValue": "+971511111111"
    }
  ],
  "comment": "Customer requested phone correction",
  "makeRegular": false
}
```

### Rules

- `orderId` must be a positive integer.
- `changes` must be a non-empty array.
- Each change requires `type` and must be one of:
  - `PHONE_CHANGE`
  - `PHONE2_CHANGE`
  - `AMOUNT_CHANGE`
  - `ADDRESS_CHANGE`
  - `ALLOW_OPEN_CHANGE`
  - `DELIVERY_DATE_CHANGE`
- `oldValue` and `newValue` are optional and can be any JSON value.
- `comment` is optional string.
- `makeRegular` is optional boolean.
- Order must currently be in `inDepot` or `inTransit` status.
- If the order is exchange-linked and `deliveryDate` is set (exchange already happened), request creation is blocked.

### Success

- **201**: `{ "ok": true }`

### Errors

- **400** one of:
  - `order_not_found`
  - `order_status_not_eligible_for_change_request`
  - `exchange_already_completed_change_request_not_allowed`
  - `no_changes_provided`
  - validation errors
- **401** missing/invalid auth headers/signature
- **500** internal error
