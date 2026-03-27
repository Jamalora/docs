# Create merchant — integration guide

This document describes how to call the **create merchant** API from your application.

---

## Endpoint (production)

| Item | Value |
|------|--------|
| **URL** | `https://livra.mofavo.com/create_merchant` |
| **Method** | `POST` |

---

## Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Content-Type` | Yes | Must be `application/json`. |
| `x-api-key` | Yes | API key issued for your integration. |
| `x-signature` | Yes | HMAC-SHA256 of the **exact raw JSON body** (the same bytes you send in the request body), using your **API secret** as the HMAC key. Send the digest as a **lowercase hex string** (optionally prefixed with `sha256=`). |

**Signing (conceptually):**

- Build the JSON body string exactly as you will send it (same ordering and spacing if you care about byte-for-byte identity — typically you serialize once and reuse that string for both signing and the body).
- Compute `HMAC-SHA256(body, api_secret)`.
- Encode the result as **hex** and put it in `x-signature`.

---

## Request body (JSON)

The body is a single JSON object with three nested objects: `merchant`, `sender`, and `contract`.

### `merchant`

| Field | Type | Notes |
|-------|------|--------|
| `name` | string | Non-empty after trim. |
| `state` | string | Non-empty after trim. |
| `city` | string | Non-empty after trim. |
| `street` | string | Non-empty after trim. |
| `phoneNumber` | string | Non-empty after trim. |
| `zipcode` | string | Non-empty after trim. |
| `TRN` | string | Non-empty after trim. |
| `CIN` | number | Integer, **8 digits** (e.g. `10000000`–`99999999`). |

### `sender`

| Field | Type | Notes |
|-------|------|--------|
| `name` | string | Non-empty after trim. |
| `state` | string | Non-empty after trim. |
| `city` | string | Non-empty after trim. |
| `street` | string | Non-empty after trim. |
| `phoneNumber` | string | Non-empty after trim. |

### `contract`

| Field | Type | Notes |
|-------|------|--------|
| `deliveryPartnerId` | number | Positive integer. |
| `deliveryFee` | number | Non-negative; decimal values allowed. |
| `exchangeFee` | number | Non-negative; decimal values allowed. |
| `cancellationFee` | number | Non-negative; decimal values allowed. |

### Example (shape only)

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

---

## Successful response

| HTTP status | Body |
|-------------|------|
| **201 Created** | `{ "merchantId": <number> }` |

`merchantId` is the identifier of the created merchant in the system.

---

## Error responses

All error bodies are JSON. Typical patterns:

| HTTP status | When | Example body |
|-------------|------|----------------|
| **400 Bad Request** | Invalid or incomplete JSON body; missing/invalid fields in `merchant`, `sender`, or `contract`. | `{ "ok": false, "error": "<message>" }` — `error` describes the validation problem (e.g. missing field, invalid `CIN`, invalid fees). |
| **401 Unauthorized** | Missing `x-api-key` / `x-signature`; unknown or inactive API key; or signature does not match the body + secret. | `{ "ok": false, "error": "Missing authentication headers" }` or `"Invalid api key"` or `"Invalid signature"` |
| **500 Internal Server Error** | Unexpected server-side failure. | `{ "ok": false, "error": "internal_error" }` |

On **401**, do not retry with the same signature on a changed body — recompute the signature from the exact new body.

---

## Credentials

You will receive:

- **API key** — sent as `x-api-key`.
- **API secret** — used only to compute `x-signature` (never sent in headers or body).

Keep the secret confidential.
