# Mofavo e-commerce API v2

The API that e-commerce sites and CMS integrations use to create orders and read order status
and stock. Same request bodies and success responses as v1 (`api.mofavo.com`), but **every**
route now requires an API key, a signature and the merchant's token. v1 keeps working until
every client has moved; Mofavo will tell you the date it stops.

| | Base URL |
|---|---|
| Staging | `https://staging.external-api.mofavo.com` |
| Production | `https://external-api.mofavo.com` |

| v2 route | Replaces (v1) |
|---|---|
| `POST /create_order` | `POST https://api.mofavo.com/external` |
| `POST /order_status` | `POST https://api.mofavo.com/woocommerce/status` |
| `POST /product_stock` | `POST https://api.mofavo.com/woocommerce/stock` |
| `POST /auth_check` | — (checks your key and signature only, and echoes the body: use it to test your signing) |

## Authentication

Every v2 route checks, in this order, and answers the first failure:

| Header | What it is | Where it comes from |
|---|---|---|
| `x-mofavo-api-key` | Your application's key | Given by Mofavo with its secret (one per application, or per shop) |
| `x-mofavo-signature` | Base64 HMAC-SHA256 of the **exact raw body bytes**, keyed with your application's secret | Computed per request |
| `Authorization: Bearer <token>` | The merchant (store) you act for | The merchant's API token, given by Mofavo |

`auth_check` checks only the first two, so you can test your signing before you have a token.

### Errors

| Status | `error` | When |
|---|---|---|
| 403 | `Missing API key` || No `x-mofavo-api-key` |
| 403 | `Invalid API key` || Unknown or inactive key |
| 401 | `Missing signature` || No `x-mofavo-signature` |
| 401 | `Invalid signature` || The signature doesn't match the body (wrong secret, a hex digest, a body re-serialized after signing) |
| 401 | `Missing or invalid Authorization header` || No `Bearer` token |
| 401 | `Invalid token` || Not a token we issued, or claims missing |
| 401 | `Token revoked or not found` || Revoked, or not a token of that merchant |
| 500 | `Misconfigured service` || Our `API_SECRET` is empty (our problem) |
| 500 | `internal_error` || Anything else on our side |

Auth errors answer `{ "error": "<message>" }`.

### Signing

Sign the bytes you send, exactly. Serialize the body once, sign that string, send that string.

```js
// Node.js
const body = JSON.stringify(payload);
const signature = crypto.createHmac("sha256", apiSecret).update(body, "utf8").digest("base64");
await fetch(`${baseUrl}/order_status`, {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "x-mofavo-api-key": apiKey,
    "x-mofavo-signature": signature,
    authorization: `Bearer ${merchantToken}`,
  },
  body,
});
```

```php
// PHP (e.g. a WooCommerce plugin)
$body = wp_json_encode($payload);
$signature = base64_encode(hash_hmac('sha256', $body, $api_secret, true)); // raw bytes, then Base64
wp_remote_post("$base_url/order_status", [
  'headers' => [
    'Content-Type' => 'application/json',
    'x-mofavo-api-key' => $api_key,
    'x-mofavo-signature' => $signature,
    'Authorization' => "Bearer $merchant_token",
  ],
  'body' => $body,
]);
```

```bash
# Shell
printf '%s' "$BODY" | openssl dgst -sha256 -hmac "$API_SECRET" -binary | base64
```

## Routes

### `POST /create_order`

```json
{
  "data": {
    "status": "draft",
    "customer": { "name": "Amal", "phone": "20123456", "address": "12 Rue X", "city": "Tunis", "state": "Tunis" },
    "cart": [{ "name": "Shirt", "quantity": 2, "pricePerUnit": 30 }],
    "total": { "totalPrice": 60 }
  }
}
```

- `status`: `draft`, `readyForPackaging` or `readyForPickUp`. `city`, `state` optional.
- `cart`: at least one item; `quantity` a whole number > 0, `pricePerUnit` ≥ 0. Numbers may be
  strings (`"1,200"` → 1200), as in v1.
- The order is created for the token's sender and merchant. A non-draft order needs a token
  that carries a contract.

| Status | Body |
|---|---|
| 201 | `{ "id": 1001 }` |
| 400 | `{ "error": "Invalid payload", "issues": [{ "path": "data.cart", "message": "…" }] }` |
| 400 | `{ "error": "bad sender" }`: the token's sender doesn't exist |
| 401 | `{ "error": "Invalid token payload" }`: the sender isn't the token's merchant's, or a non-draft order with a token without a contract |

### `POST /order_status`

Body `{ "refs": ["WC-1001", "WC-1002"] }` (WooCommerce order references).

| Status | Body |
|---|---|
| 200 | `{ "success": true, "orders": [{ "ref": "WC-1001", "status": "delivered", "finalDestination": "primaryRecipient" }] }`. Refs that aren't the merchant's are absent; none → `orders: []` |
| 400 | `{ "success": false, "error": "Invalid request payload" }` |

### `POST /product_stock`

Body `{ "refs": ["SKU-1"] }` (product references).

| Status | Body |
|---|---|
| 200 | `{ "success": true, "products": [{ "ref": "SKU-1", "quantity": 7 }] }`: `quantity` is available stock (quantity minus reserved) |
| 404 | `{ "success": false, "error": "Product not found" }`: none of the refs is the merchant's |
| 400 | `{ "success": false, "error": "Invalid request payload" }` |

## Moving from v1

| You call today | Change |
|---|---|
| `POST /external` | Only the URL: `https://external-api.mofavo.com/create_order`. Your key, secret, signing code and token stay the same. |
| `POST /woocommerce/status`, `/woocommerce/stock` | The URLs (`/order_status`, `/product_stock`), **plus** a per-shop API key and secret: send `x-mofavo-api-key` and sign with that shop's secret instead of the shared WooCommerce secret. Ask Mofavo for the shop's key and secret. |

What else changes from v1:

1. Status and stock need an API key and a per-client signature.
2. `create_order` refuses revoked tokens.
3. A missing or unknown API key is **403**.
4. A server error answers `{ "error": "internal_error" }`.
5. `create_order` saves the order and its products together, or neither.

Test against staging first (`https://staging.external-api.mofavo.com`), starting with `POST /auth_check`.
