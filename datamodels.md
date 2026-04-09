# Pixelz Checkout System - Data Models and Database Schema

This document defines data models and SQL schema for each service in the proposed architecture.

## Confirmed Design Decisions

- Checkout uses `payments` + `payment_attempts`.
- Checkout uses dedicated `idempotency_keys` table.
- Checkout uses generic `outbox_events` table.
- Portal Backend stores server-owned checkout attempt idempotency state.
- Portal owns projection/read model tables.
- Delivery is at-least-once; all consumers must be idempotent.

## Service Ownership Overview

- Portal Backend owns customer-facing order data and read projections.
- Checkout Service owns payment lifecycle, payment attempts, idempotency, outbox.
- Production Service owns fulfillment lifecycle.
- Email Service owns email delivery records.

---

## 1) Portal Backend (Service: `portal-backend`)

### Data models
- `orders`: customer-created order metadata.
- `checkout_attempts`: server-owned checkout attempt and idempotency key lifecycle per order.
- `order_status_projection`: consolidated read status for UI.

### SQL schema (PostgreSQL)

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    order_name VARCHAR(255) NOT NULL,
    currency CHAR(3) NOT NULL,
    total_amount_cents BIGINT NOT NULL CHECK (total_amount_cents >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_order_name_trgm ON orders USING gin (order_name gin_trgm_ops);

CREATE TABLE checkout_attempts (
    id UUID PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    customer_id BIGINT NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    status VARCHAR(32) NOT NULL,
    checkout_id UUID,
    payment_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(idempotency_key)
);

CREATE INDEX idx_checkout_attempts_order_id ON checkout_attempts(order_id);
CREATE INDEX idx_checkout_attempts_status ON checkout_attempts(status);
CREATE UNIQUE INDEX uq_checkout_attempt_open_order
    ON checkout_attempts(order_id)
    WHERE status = 'OPEN';

CREATE TABLE order_status_projection (
    order_id BIGINT PRIMARY KEY,
    payment_status VARCHAR(32) NOT NULL,
    fulfillment_status VARCHAR(32) NOT NULL,
    payment_id UUID,
    last_event_id UUID,
    projection_version BIGINT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_order_status_projection_payment_status ON order_status_projection(payment_status);
CREATE INDEX idx_order_status_projection_fulfillment_status ON order_status_projection(fulfillment_status);
```

---

## 2) Checkout Service (Service: `checkout-service`)

### Data models
- `payments`: one row per order payment lifecycle.
- `payment_attempts`: each PSP attempt/callback history row.
- `idempotency_keys`: deduplicate API and webhook processing.
- `outbox_events`: transactional event publishing.

### Status enums
- `payments.status`: `CREATED`, `PENDING_PSP`, `PAID`, `FAILED`, `CANCELLED`.
- `payment_attempts.status`: `INITIATED`, `PSP_PENDING`, `SUCCEEDED`, `FAILED`, `EXPIRED`.
- `outbox_events.status`: `PENDING`, `PUBLISHED`, `FAILED`.

### SQL schema (PostgreSQL)

```sql
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    order_id BIGINT NOT NULL,
    customer_id BIGINT NOT NULL,
    amount_cents BIGINT NOT NULL CHECK (amount_cents >= 0),
    currency CHAR(3) NOT NULL,
    status VARCHAR(32) NOT NULL,
    psp_provider VARCHAR(64) NOT NULL,
    psp_payment_ref VARCHAR(255),
    paid_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(order_id)
);

CREATE INDEX idx_payments_customer_id ON payments(customer_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_psp_payment_ref ON payments(psp_payment_ref);

CREATE TABLE payment_attempts (
    id UUID PRIMARY KEY,
    payment_id UUID NOT NULL REFERENCES payments(id),
    attempt_no INT NOT NULL CHECK (attempt_no > 0),
    status VARCHAR(32) NOT NULL,
    psp_provider VARCHAR(64) NOT NULL,
    psp_session_id VARCHAR(255),
    psp_event_id VARCHAR(255),
    error_code VARCHAR(128),
    error_message TEXT,
    request_payload JSONB,
    response_payload JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(payment_id, attempt_no),
    UNIQUE(psp_provider, psp_event_id)
);

CREATE INDEX idx_payment_attempts_payment_id ON payment_attempts(payment_id);
CREATE INDEX idx_payment_attempts_status ON payment_attempts(status);

CREATE TABLE idempotency_keys (
    id UUID PRIMARY KEY,
    scope VARCHAR(64) NOT NULL,
    idempotency_key VARCHAR(255) NOT NULL,
    request_hash VARCHAR(128) NOT NULL,
    resource_type VARCHAR(64),
    resource_id VARCHAR(255),
    response_code INT,
    response_body JSONB,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    UNIQUE(scope, idempotency_key)
);

CREATE INDEX idx_idempotency_expires_at ON idempotency_keys(expires_at);

CREATE TABLE outbox_events (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(64) NOT NULL,
    aggregate_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(128) NOT NULL,
    event_version INT NOT NULL,
    payload JSONB NOT NULL,
    headers JSONB,
    status VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    published_at TIMESTAMPTZ,
    retry_count INT NOT NULL DEFAULT 0,
    next_retry_at TIMESTAMPTZ,
    last_error TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON outbox_events(status, created_at);
CREATE INDEX idx_outbox_aggregate ON outbox_events(aggregate_type, aggregate_id);
```

---

## 3) Production Service (Service: `production-service`)

### Data models
- `production_orders`: fulfillment state for paid orders.
- `production_event_dedup`: deduplicate incoming broker events.

### Status enums
- `production_orders.status`: `RECEIVED`, `QUEUED`, `PROCESSING`, `COMPLETED`, `FAILED`.

### SQL schema (PostgreSQL)

```sql
CREATE TABLE production_orders (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL UNIQUE,
    payment_id UUID NOT NULL,
    status VARCHAR(32) NOT NULL,
    external_production_ref VARCHAR(255),
    failure_reason TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_production_orders_status ON production_orders(status);

CREATE TABLE production_event_dedup (
    event_id UUID PRIMARY KEY,
    event_type VARCHAR(128) NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---


## Event Contracts (Minimum)

### `order.paid` (Checkout -> Broker -> Consumers)
Required fields:
- `event_id` (UUID)
- `event_type` (`order.paid`)
- `event_version` (int)
- `occurred_at` (ISO timestamp)
- `order_id` (bigint)
- `payment_id` (UUID)
- `customer_id` (bigint)
- `amount_cents` (bigint)
- `currency` (char(3))
- `idempotency_key` (string)
- `trace_id` (string)

Example payload:

```json
{
  "event_id": "6d61e86e-a7f7-4d3f-8f3d-0a36f2fe95db",
  "event_type": "order.paid",
  "event_version": 1,
  "occurred_at": "2026-04-10T00:00:00Z",
  "order_id": 12345,
  "payment_id": "b4f7bba8-d5de-4d45-b93c-a7bb2f1a091c",
  "customer_id": 998,
  "amount_cents": 129900,
  "currency": "USD",
  "idempotency_key": "checkout:order:12345:v1",
  "trace_id": "a613b9e98f78490f"
}
```

## Operational Notes

- At-least-once delivery means duplicate events will happen by design.
- Consumer dedup table keyed by `event_id` is mandatory.
- Outbox publisher should retry with exponential backoff and dead-letter on max retries.
- Projection updates in Portal should be monotonic by event time/version to avoid stale overwrite.
