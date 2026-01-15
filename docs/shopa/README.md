# Shopa Callback API

This document describes the Shopa callback endpoint for creating orders and
logging comments as history events.

## Endpoint

`POST /shopa/callback`

## Headers

- `Content-Type: application/json`
- `x-mofavo-api-key: <api key>`
- `x-mofavo-signature: <signature>`
- `Authorization: Bearer <jwt>`

## Authentication

Two layers are required:

1. API key/secret
   - The API key is sent in `x-mofavo-api-key`.
   - The signature is an HMAC SHA256 of the raw request body using the API
     secret, then base64-encoded.
   - Send the signature in `x-mofavo-signature`.
2. JWT
   - Send the JWT in `Authorization: Bearer <jwt>`.
   - The token must include `merchantId` and `senderId`.

## Request Body

```json
{
  "ref": "shopa-1732",
  "name": "Tedst Test",
  "phone": "+21690909090",
  "comments": [
    {
      "content": "First comment",
      "time": "2026-01-06T11:19:13+01:00"
    }
  ]
}
```

### Fields

- `ref` (string, required): External reference for the order.
- `name` (string, required): Customer name.
- `phone` (string, required): Customer phone number.
- `comments` (array, required): List of comments.
  - `content` (string, required)
  - `time` (datetime, required): ISO 8601 timestamp.

## Responses

### 201 Created

```json
{
  "id": 12345
}
```

### 401 Unauthorized

Returned when:

- Missing API key.
- Invalid signature.
- Missing or invalid JWT.
- Token payload does not match the merchant.

### 403 Forbidden

Returned when the API key is not recognized or is inactive.

### 500 Internal Server Error

Returned for unexpected errors.

## Behavior Notes

- The order is created if the `ref` is new.
- If an order already exists with the same `ref`, the API does not create a
  duplicate order.
- Comments are stored as history events. Only new comments are inserted.
  A comment is considered duplicate when both the `content` and `time` match.

## Example Curl

```bash
curl -X POST "https://api.mofavo.com/shopa/callback" \
  -H "Content-Type: application/json" \
  -H "x-mofavo-api-key: $SHOPA_API_KEY" \
  -H "x-mofavo-signature: $SHOPA_SIGNATURE" \
  -H "Authorization: Bearer $SHOPA_JWT" \
  -d '{
    "ref": "shopa-1732",
    "name": "Tedst Test",
    "phone": "+21690909090",
    "comments": [
      {
        "content": "First comment",
        "time": "2026-01-06T11:19:13+01:00"
      }
    ]
  }'
```
