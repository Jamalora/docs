# External API Order Callback

This Lambda powers the external API endpoint that third-party apps use to create
orders in Mofavo. Each app authenticates with its own API key and HMAC
signature, while the merchant and sender context comes from a Bearer token.

## Endpoint

- `POST /external`

## Authentication

Two checks are required on every request:

1) App authentication via headers:
   - `x-mofavo-api-key`
   - `x-mofavo-signature` (HMAC SHA256 base64 of the raw request body)
2) Merchant/sender authentication via Bearer token:
   - `Authorization: Bearer <jwt>`

The JWT must contain `merchantId` and `senderId` in its payload.

## Request body

```json
{
  "data": {
    "reference": "EXT-10001",
    "customer": {
      "name": "Jane Doe",
      "phone": "+2348012345678",
      "address": "12 Banana Island Road",
      "city": "Lagos",
      "state": "LA"
    },
    "cart": [
      {
        "name": "Premium T-Shirt",
        "quantity": 2,
        "pricePerUnit": 7500
      }
    ],
    "total": {
      "totalPrice": 15000
    }
  }
}
```

## Responses

- `201` created
- `401` missing/invalid signature or token
- `403` invalid API key
- `400` sender/merchant lookup failure
- `500` misconfigured service

## Environment

- `API_SECRET` for JWT verification
- DB connection vars from shared lambda environment

## Local testing

Example request lives in `services/external_api/callback/test.http` with payload
in `services/external_api/callback/test.json`.

To generate an app key/secret:

```bash
node scripts/external-api-generate-keys.cjs
```

To generate a signature for the test payload:

```bash
node scripts/external-api-signature.cjs \
  --secret <apiSecret> \
  --file services/external_api/callback/test.json
```
