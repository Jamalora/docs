# WooCommerce Stock API Documentation

## Overview

The WooCommerce Stock API allows you to query the current stock quantity for products by their reference codes. This API requires JWT Bearer token authentication and request signature verification for secure access.

**Endpoint:** `POST /woocommerce/stock`

## Authentication

### Bearer Token (JWT)

All requests must include a valid JWT Bearer token in the `Authorization` header.

**Header Format:**
```
Authorization: Bearer <your-jwt-token>
```

**Token Requirements:**
- The token must be a valid JWT signed with the API secret
- The token payload must contain:
  - `merchantId` (required): The merchant ID associated with the products
  - `jti` (required): Unique token identifier (UUID)
  - `senderId` (optional): Sender ID
  - `partnerId` (optional): Partner ID
  - `contractId` (optional): Contract ID

**Token Validation:**
- The token must exist in the `MerchantApiTokens` table
- The token status must be `active`
- The token must not be revoked (`revoked_at` must be null)
- Products are filtered by the token's `merchantId` to ensure proper access control

### Request Signature

All requests must include a valid signature to verify the request authenticity.

**Header Format:**
```
x-mofavo-signature: <base64-encoded-signature>
```

**Signature Generation:**
The signature is generated using HMAC-SHA256 with your client secret:
```javascript
const crypto = require('crypto');
const signature = crypto
  .createHmac('sha256', WOOCOMMERCE_CLIENT_SECRET)
  .update(JSON.stringify(requestBody), 'utf8')
  .digest('base64');
```

## Request Format

### Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token: `Bearer <jwt-token>` |
| `x-mofavo-signature` | Yes | Base64-encoded HMAC-SHA256 signature |
| `Content-Type` | Yes | Must be `application/json` |

### Request Body

```json
{
  "refs": ["product-ref-1", "product-ref-2", "product-ref-3"]
}
```

**Field Descriptions:**
- `refs` (required): An array of product reference strings. The API will return stock information for all products matching these references that belong to the authenticated merchant.

**Example:**
```json
{
  "refs": ["product-ref-1", "product-ref-2"]
}
```

## Response Format

### Success Response (200 OK)

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

**Response Fields:**
- `success` (boolean): Always `true` for successful requests
- `products` (array): Array of product objects with stock information
  - `ref` (string): Product reference code
  - `quantity` (number): Current stock quantity for this product

**Note:** Only products that match the provided refs AND belong to the authenticated merchant's `merchantId` will be returned. If a ref doesn't match any products or belongs to a different merchant, it will not appear in the response.

### Error Responses

#### 400 Bad Request - Invalid Payload
```json
{
  "success": false,
  "error": "Invalid request payload"
}
```
Occurs when the request body doesn't match the expected schema (e.g., missing `refs` field or invalid format).

#### 401 Unauthorized - Missing/Invalid Authorization Header
```json
{
  "message": "Missing or invalid Authorization header"
}
```

#### 401 Unauthorized - Invalid Token
```json
{
  "message": "Invalid token"
}
```
Occurs when the JWT token is malformed or cannot be verified.

#### 401 Unauthorized - Invalid Token Payload
```json
{
  "message": "Invalid token payload"
}
```
Occurs when the token is missing required fields (`merchantId` or `jti`).

#### 401 Unauthorized - Token Revoked or Not Found
```json
{
  "message": "Token revoked or not found"
}
```
Occurs when the token doesn't exist in the database, is inactive, or has been revoked.

#### 401 Unauthorized - Invalid Signature
```json
{
  "success": false,
  "error": "Invalid signature"
}
```
Occurs when the request signature doesn't match the request payload.

#### 404 Not Found - Product Not Found
```json
{
  "success": false,
  "error": "Product not found"
}
```
Occurs when no products are found matching the provided refs for the authenticated merchant.

#### 500 Internal Server Error
```json
{
  "success": false,
  "error": "Internal server error"
}
```
Occurs when an unexpected server error happens.

## Example Requests

### cURL Example

```bash
curl -X POST https://api.mofavo.com/woocommerce/stock \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-jwt-token>" \
  -H "x-mofavo-signature: <base64-encoded-signature>" \
  -d '{
    "refs": ["product-ref-1", "product-ref-2"]
  }'
```

### JavaScript/Node.js Example

```javascript
const crypto = require('crypto');
const axios = require('axios');

const requestBody = {
  refs: [
    "product-ref-1",
    "product-ref-2"
  ]
};

// Generate signature
const signature = crypto
  .createHmac('sha256', process.env.WOOCOMMERCE_CLIENT_SECRET)
  .update(JSON.stringify(requestBody), 'utf8')
  .digest('base64');

// Make request
const response = await axios.post(
  'https://api.mofavo.com/woocommerce/stock',
  requestBody,
  {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${jwtToken}`,
      'x-mofavo-signature': signature
    }
  }
);

console.log(response.data);
```

## Support

For issues or questions, please contact your API administrator or refer to the internal documentation.

