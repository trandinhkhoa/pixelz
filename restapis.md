# Pixelz Checkout Architecture - REST API Design

This file defines REST APIs for services in `README.md`.

## Scope

- Includes **customer-facing APIs exposed via API Gateway**.
- Includes **cross-service internal REST endpoints**.
- Excludes admin/ops endpoints.
- Downstream processing (`Production`, `Email`) is event-driven via `order.paid`, not REST.

## API Standards

- Public portal base path: `/api/v1`
- Internal service base path: `/internal/v1`
- Edge webhook base path (gateway): `/webhooks/psp`
- API Gateway is the edge entry point for all `/api/v1` requests.
- API Gateway responsibilities: Google SSO/OIDC token validation, rate limiting, load balancing, edge request filtering.
- Frontend uses first-party `portal-access-jwt` issued by Portal Backend after Google SSO callback.
- AuthN/AuthZ: `mTLS` between services + service JWT in `Authorization: Bearer <token>`
- Idempotency: required for all `POST`/`PATCH`; for `POST /api/v1/orders/{order_id}/checkout`, Portal Backend generates and reuses a server-owned checkout-attempt key.
- Content type: `application/json`
- Traceability headers:
  - `X-Request-Id`
  - `X-Correlation-Id`

### Webhook security policy (PSP)
- PSP webhook enters through API Gateway: `POST /webhooks/psp/{provider}`.
- No user JWT is required for webhook route.
- Gateway enforces path policy, IP allowlist/rate guard, TLS termination.
- Checkout Service must still verify provider signature and replay/idempotency.

## Portal Backend Public APIs (consumed by Portal UI)

### P.0 Google SSO Login Start

- **Method**: `GET`
- **Path**: `/auth/google/start`
- **Caller**: Portal UI / browser
- **Purpose**: Redirect user to Google SSO authorization flow.

#### Example query
- `GET /auth/google/start`

---

### P.0.1 Google SSO Callback

- **Method**: `GET`
- **Path**: `/auth/google/callback`
- **Caller**: Google Identity Provider
- **Purpose**: Receive authorization callback and establish user session.

#### Example query
- `GET /auth/google/callback?code=<oauth-code>&state=<opaque-state>`

---

### P.0.2 Create Order

- **Method**: `POST`
- **Path**: `/api/v1/orders`
- **Caller**: Portal UI / BFF client
- **Purpose**: Create a customer order before checkout.

#### Example request
- `POST /api/v1/orders`

#### Request headers
- `Authorization: Bearer <portal-access-jwt>`
- `Idempotency-Key: <unique-key>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Request body

```json
{
  "order_name": "Spring Campaign Batch A",
  "currency": "USD",
  "total_amount_cents": 129900
}
```

#### Success response (`201 Created`)

```json
{
  "order_id": 12345,
  "order_name": "Spring Campaign Batch A",
  "currency": "USD",
  "total_amount_cents": 129900,
  "payment_status": "NOT_STARTED",
  "fulfillment_status": "NOT_STARTED"
}
```

#### Error responses
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`

---

### P.1 Search Orders by Name

- **Method**: `GET`
- **Path**: `/api/v1/orders`
- **Caller**: Portal UI / BFF client
- **Purpose**: Search and filter customer-owned orders by name.

#### Query params
- `name` (optional, string)
- `page` (optional, default `1`)
- `page_size` (optional, default `20`, max `100`)

#### Example query
- `GET /api/v1/orders?name=spring&page=1&page_size=20`

#### Request headers
- `Authorization: Bearer <customer-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "items": [
    {
      "order_id": 12345,
      "order_name": "Spring Campaign Batch A",
      "currency": "USD",
      "total_amount_cents": 129900,
      "payment_status": "PENDING_PSP",
      "fulfillment_status": "NOT_STARTED",
      "updated_at": "2026-04-10T01:00:00Z"
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

#### Error responses
- `400 Bad Request` - invalid query params
- `401 Unauthorized`
- `403 Forbidden`

---

### P.2 Get Order Detail

- **Method**: `GET`
- **Path**: `/api/v1/orders/{order_id}`
- **Caller**: Portal UI / BFF client
- **Purpose**: Read order details and current projected status.

#### Example query
- `GET /api/v1/orders/12345`

#### Request headers
- `Authorization: Bearer <customer-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "order_id": 12345,
  "order_name": "Spring Campaign Batch A",
  "customer_id": 998,
  "currency": "USD",
  "total_amount_cents": 129900,
  "payment_status": "PENDING_PSP",
  "fulfillment_status": "NOT_STARTED",
  "created_at": "2026-04-09T12:00:00Z",
  "updated_at": "2026-04-10T01:00:00Z"
}
```

#### Error responses
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

---

### P.3 Request Checkout for Order

- **Method**: `POST`
- **Path**: `/api/v1/orders/{order_id}/checkout`
- **Caller**: Portal UI / BFF client
- **Purpose**: Start checkout request for order; Portal delegates to Checkout Service using server-owned checkout-attempt idempotency.

#### Example request
- `POST /api/v1/orders/12345/checkout`

#### Request headers
- `Authorization: Bearer <customer-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Idempotency behavior
- Portal Backend creates/stores `checkout_attempt_idempotency_key` for an open checkout attempt (`order_id` scope).
- Repeated clicks/retries for same open attempt reuse same key and return same checkout attempt/session state.
- Portal Backend forwards the key to Checkout Service (`Idempotency-Key`) and Checkout Service reuses it at PSP provider API.

#### Request body

```json
{
  "psp_provider": "STRIPE",
  "return_url": "https://portal.pixelz.example/checkout/return",
  "cancel_url": "https://portal.pixelz.example/checkout/cancel"
}
```

#### Success response (`201 Created`)

```json
{
  "order_id": 12345,
  "checkout_id": "7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7",
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "status": "PENDING_PSP",
  "redirect_url": "https://checkout.stripe.com/c/pay/cs_test_a1b2c3",
  "expires_at": "2026-04-10T01:00:00Z"
}
```

#### Success response (`200 OK`, existing open attempt reused)

```json
{
  "order_id": 12345,
  "checkout_id": "7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7",
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "status": "PENDING_PSP",
  "redirect_url": "https://checkout.stripe.com/c/pay/cs_test_a1b2c3",
  "expires_at": "2026-04-10T01:00:00Z"
}
```

#### Error responses
- `400 Bad Request`
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`
- `409 Conflict` - order already paid or active checkout exists
- `422 Unprocessable Entity` - order not eligible

---

### P.4 Get Order Checkout Status

- **Method**: `GET`
- **Path**: `/api/v1/orders/{order_id}/status`
- **Caller**: Portal UI / BFF client
- **Purpose**: Read latest projected payment and fulfillment state for status page.

#### Example query
- `GET /api/v1/orders/12345/status`

#### Request headers
- `Authorization: Bearer <customer-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "order_id": 12345,
  "payment_status": "PAID",
  "fulfillment_status": "QUEUED",
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "last_event_id": "6d61e86e-a7f7-4d3f-8f3d-0a36f2fe95db",
  "updated_at": "2026-04-10T01:05:00Z"
}
```

#### Error responses
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

## Involved Service-to-Service REST Calls

- `API Gateway` -> `Portal Backend` (public API routing)
- `API Gateway` -> `Checkout Service` (PSP webhook routing)
- `Portal Backend` -> `Checkout Service`

No other cross-service REST call is required by current architecture. `Checkout Service` publishes `order.paid` to broker; downstream services consume asynchronously.

---

## 1) Checkout Service APIs (consumed by Portal Backend)

### 1.1 Create Checkout Session

- **Method**: `POST`
- **Path**: `/internal/v1/checkouts`
- **Caller**: `Portal Backend`
- **Purpose**: Start checkout for an existing order and create PSP session/intent.

#### Example request
- `POST /internal/v1/checkouts`

#### Request headers
- `Authorization: Bearer <service-jwt>`
- `Idempotency-Key: <unique-key>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Request body

```json
{
  "order_id": 12345,
  "customer_id": 998,
  "amount_cents": 129900,
  "currency": "USD",
  "return_url": "https://portal.pixelz.example/checkout/return",
  "cancel_url": "https://portal.pixelz.example/checkout/cancel",
  "psp_provider": "STRIPE"
}
```

#### Success response (`201 Created`)

```json
{
  "checkout_id": "7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7",
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "order_id": 12345,
  "status": "PENDING_PSP",
  "psp_provider": "STRIPE",
  "psp_session_id": "cs_test_a1b2c3",
  "redirect_url": "https://checkout.stripe.com/c/pay/cs_test_a1b2c3",
  "expires_at": "2026-04-10T01:00:00Z",
  "created_at": "2026-04-10T00:42:00Z"
}
```

#### Error responses
- `400 Bad Request` - invalid payload
- `401 Unauthorized` - invalid service JWT
- `403 Forbidden` - caller not authorized for customer/order
- `404 Not Found` - order not found
- `409 Conflict` - order already paid or active checkout exists
- `422 Unprocessable Entity` - order not eligible for checkout

---

### 1.2 Get Checkout by Checkout ID

- **Method**: `GET`
- **Path**: `/internal/v1/checkouts/{checkout_id}`
- **Caller**: `Portal Backend`
- **Purpose**: Poll checkout/payment status.

#### Example query
- `GET /internal/v1/checkouts/7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7`

#### Request headers
- `Authorization: Bearer <service-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "checkout_id": "7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7",
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "order_id": 12345,
  "customer_id": 998,
  "status": "PAID",
  "psp_provider": "STRIPE",
  "paid_at": "2026-04-10T00:45:30Z",
  "updated_at": "2026-04-10T00:45:30Z"
}
```

#### Error responses
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

---

### 1.3 Get Checkout by Order ID

- **Method**: `GET`
- **Path**: `/internal/v1/orders/{order_id}/checkout`
- **Caller**: `Portal Backend`
- **Purpose**: Fetch latest checkout/payment state for order page rendering.

#### Example query
- `GET /internal/v1/orders/12345/checkout`

#### Request headers
- `Authorization: Bearer <service-jwt>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "order_id": 12345,
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "checkout_id": "7fbf88ec-c8d6-4ef0-9aa4-6de4627a3ac7",
  "status": "PENDING_PSP",
  "psp_provider": "STRIPE",
  "last_attempt_no": 1,
  "updated_at": "2026-04-10T00:43:10Z"
}
```

#### Error responses
- `401 Unauthorized`
- `403 Forbidden`
- `404 Not Found`

---

### 1.4 PSP Webhook Receiver

- **Method**: `POST`
- **Path**: `/internal/v1/psp/webhooks/{provider}`
- **Caller**: PSP provider
- **Purpose**: Receive asynchronous payment status callback from PSP.

#### Example request
- `POST /internal/v1/psp/webhooks/stripe`

#### Request headers
- `Content-Type: application/json`
- `X-PSP-Signature: <provider-signature>`
- `X-Request-Id: <uuid>`
- `X-Correlation-Id: <uuid>`

#### Success response (`200 OK`)

```json
{
  "received": true
}
```

#### Error responses
- `400 Bad Request` - invalid webhook payload
- `401 Unauthorized` - invalid signature
- `409 Conflict` - duplicate webhook already processed

---

## 2) Event-Driven (Non-REST) Integration Contracts

These are involved internal integrations in architecture, but not REST endpoints:

- `Checkout Service` -> Message Broker: publish `order.paid`
- `Production Service` <- Message Broker: consume `order.paid`
- `Email Service` <- Message Broker: consume `order.paid`

## Standard Error Shape

All REST error responses should use:

```json
{
  "error_code": "ORDER_ALREADY_PAID",
  "message": "Order 12345 already has a successful payment",
  "details": null,
  "request_id": "fd58a9c4-df68-4f8a-8b65-e43f502de4d2",
  "correlation_id": "b34bbfcd-4fc7-4f00-a4b2-91842595a724"
}
```

## Idempotency Rules

- `Idempotency-Key` must be unique per logical command.
- Same key + same payload returns same success response.
- Same key + different payload returns `409 Conflict`.
- Public checkout request (`POST /api/v1/orders/{order_id}/checkout`) uses server-owned idempotency (Portal Backend resolves/reuses key and forwards it internally).
- Recommended key format: `svc:<caller>:action:<resource>:v1`.
