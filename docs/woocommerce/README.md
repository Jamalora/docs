# WooCommerce APIs Documentation

## Quick Start

**Base URL:** `https://api.mofavo.com`

**Client Secret:** Set the `WOOCOMMERCE_CLIENT_SECRET` variable (value: `Mofavo.ftw2025`)

**Available APIs:**
- **Stock API** (`POST /woocommerce/stock`) - Get product stock quantities
- **Status API** (`POST /woocommerce/status`) - Get order status and destination

Both APIs use the same authentication method.

## Table of Contents

1. [Quick Start](#quick-start)
2. [Authentication](#authentication)
3. [Helper Function](#helper-function)
4. [Stock API](#stock-api)
5. [Status API](#status-api)
6. [Common Error Responses](#common-error-responses)
7. [Support](#support)

## Authentication

All requests require two things:

### 1. JWT Bearer Token

Include in the `Authorization` header:
```
Authorization: Bearer <your-jwt-token>
```

**Token must contain:**
- `merchantId` (required) - Your merchant ID
- `jti` (required) - Unique token identifier (UUID)
- `senderId`, `partnerId`, `contractId` (optional)

### 2. Request Signature

Include in the `x-mofavo-signature` header:
```
x-mofavo-signature: <base64-encoded-signature>
```

**Generate signature using HMAC-SHA256:**
```javascript
const crypto = require('crypto');
const signature = crypto
  .createHmac('sha256', WOOCOMMERCE_CLIENT_SECRET)
  .update(JSON.stringify(requestBody), 'utf8')
  .digest('base64');
```

### Required Headers

| Header | Value |
|--------|-------|
| `Authorization` | `Bearer <your-jwt-token>` |
| `x-mofavo-signature` | `<base64-encoded-signature>` |
| `Content-Type` | `application/json` |

## Helper Function

Use this reusable function for all API requests:

```javascript
const crypto = require('crypto');
const axios = require('axios');

// Set this to your client secret (e.g., from environment variable)
const WOOCOMMERCE_CLIENT_SECRET = process.env.WOOCOMMERCE_CLIENT_SECRET;
const BASE_URL = 'https://api.mofavo.com';

async function makeWooCommerceRequest(endpoint, requestBody, jwtToken) {
  // Generate signature
  const signature = crypto
    .createHmac('sha256', WOOCOMMERCE_CLIENT_SECRET)
    .update(JSON.stringify(requestBody), 'utf8')
    .digest('base64');

  // Make request
  const response = await axios.post(
    `${BASE_URL}${endpoint}`,
    requestBody,
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${jwtToken}`,
        'x-mofavo-signature': signature
      }
    }
  );

  return response.data;
}
```

## Stock API

**Endpoint:** `POST /woocommerce/stock`

Get stock quantities for products by their reference codes.

### Request

```json
{
  "refs": ["product-ref-1", "product-ref-2"]
}
```

### Response

```json
{
  "success": true,
  "products": [
    {
      "ref": "product-ref-1",
      "quantity": 10
    },
    {
      "ref": "product-ref-2",
      "quantity": 5
    }
  ]
}
```

**Note:** Only products that match the refs AND belong to your merchant will be returned.

### Example

```javascript
const stockData = await makeWooCommerceRequest(
  '/woocommerce/stock',
  {
    refs: ['product-ref-1', 'product-ref-2']
  },
  jwtToken
);

console.log(stockData);
// {
//   success: true,
//   products: [
//     { ref: 'product-ref-1', quantity: 10 },
//     { ref: 'product-ref-2', quantity: 5 }
//   ]
// }
```

### cURL Example

```bash
curl -X POST https://api.mofavo.com/woocommerce/stock \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-jwt-token>" \
  -H "x-mofavo-signature: <base64-encoded-signature>" \
  -d '{"refs": ["product-ref-1", "product-ref-2"]}'
```

## Status API

**Endpoint:** `POST /woocommerce/status`

Get order status and destination information by their WooCommerce order reference codes.

### Request

```json
{
  "refs": ["ref-order1", "ref-order2"]
}
```

### Response

```json
{
  "success": true,
  "orders": [
    {
      "ref": "ref-order1",
      "status": "draft",
      "finalDestination": "address-1"
    },
    {
      "ref": "ref-order2",
      "status": "readyForPickUp",
      "finalDestination": "address-2"
    }
  ]
}
```

**Response fields:**
- `ref` - Order's woocomerce reference
- `status` - Current order status (e.g., "draft", "readyForPickUp", "inTransit")
- `finalDestination` - Delivery address

**Note:** Only orders that match the refs AND belong to your merchant will be returned.

### Example

```javascript
const statusData = await makeWooCommerceRequest(
  '/woocommerce/status',
  {
    refs: ['order-ref-1', 'order-ref-2']
  },
  jwtToken
);

console.log(statusData);
// {
//   success: true,
//   orders: [
//     { ref: 'ref-order1', status: 'draft', finalDestination: 'address-1' },
//     { ref: 'ref-order2', status: 'readyForPickUp', finalDestination: 'address-2' }
//   ]
// }
```

### cURL Example

```bash
curl -X POST https://api.mofavo.com/woocommerce/status \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-jwt-token>" \
  -H "x-mofavo-signature: <base64-encoded-signature>" \
  -d '{"refs": ["order-ref-1", "order-ref-2"]}'
```

## Common Error Responses

Both APIs return the same error responses:

| Status | Error | Description |
|--------|-------|-------------|
| 400 | `Invalid request payload` | Missing `refs` field or invalid format |
| 401 | `Missing or invalid Authorization header` | Authorization header missing or malformed |
| 401 | `Invalid token` | JWT token is malformed or cannot be verified |
| 401 | `Invalid token payload` | Token missing required fields (`merchantId` or `jti`) |
| 401 | `Token revoked or not found` | Token doesn't exist, is inactive, or revoked |
| 401 | `Invalid signature` | Request signature doesn't match payload |
| 404 | `Product not found` / `Order not found` | No items found matching the refs |
| 500 | `Internal server error` | Unexpected server error |

**Error Response Format:**
```json
{
  "success": false,
  "error": "Error message"
}
```

Or for authorization errors:
```json
{
  "message": "Error message"
}
```

## Support

For issues or questions, please contact your API administrator or refer to the internal documentation.
