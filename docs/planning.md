# Pixelz Checkout Delivery Plan

## Planning Approach

- Delivery strategy: **MVP-first vertical slices**
- Iteration cadence: **2-week iterations**
- Story sizing scale: **Fibonacci story points** (`1, 2, 3, 5, 8, 13`)
- Assumming 1 point = 1 day, as a starting point

## Delivery Sequence (High Level)

1. Platform foundation and service scaffolding
2. MVP checkout flow (search -> checkout -> paid -> production push)
3. Notification and invoicing completion
4. Reliability, idempotency, and duplicate prevention hardening
5. Observability, reconciliation, and go-live readiness

---

## Epic 1: Platform Foundation and Core Contracts

### User Story 1.1 - Provision service and database skeletons
- **Description:** As a backend engineer, I want all core services and databases scaffolded so feature teams can build in parallel safely.
- **Acceptance Criteria:**
  - Repositories/modules exist for `portal-backend`, `checkout-service`, `production-service`, `email-service`.
  - PostgreSQL schemas for Portal, Checkout, and Production are created from `datamodels.md`.
  - Services start in local environment with health endpoints.
  - Message broker is available locally with required topic/queue for `order.paid`.
- **Value:** Enables parallel implementation and reduces integration setup risk.
- **Size:** `5`
- **Tasks:**
  - Create service templates and baseline configs.
  - Provision DB migrations for initial tables.
  - Configure broker topic and local docker compose.

### User Story 1.2 - Define internal API contract for checkout
- **Description:** As a portal service, I need stable checkout endpoints to create and query checkout sessions.
- **Acceptance Criteria:**
  - `POST /internal/v1/checkouts` implemented according to `restapis.md`.
  - `GET /internal/v1/checkouts/{checkout_id}` implemented.
  - `GET /internal/v1/orders/{order_id}/checkout` implemented.
  - Contract tests verify request/response schema and status codes.
- **Value:** Locks integration surface early and prevents contract drift.
- **Size:** `5`

### User Story 1.3 - Implement service-to-service security baseline
- **Description:** As a platform team, we need secure internal communication between services.
- **Acceptance Criteria:**
  - mTLS is enabled for internal REST traffic.
  - Service JWT validation is enforced by Checkout APIs.
  - Unauthorized requests return `401`, invalid permissions return `403`.
- **Value:** Prevents unauthorized internal access and reduces security risk.
- **Size:** `5`

---

## Epic 2: MVP Checkout to Production Push (Core Business Value)

### User Story 2.1 - Search orders by name in Portal
- **Description:** As an Art Director, I want to find an existing order by name so I can check out quickly.
- **Acceptance Criteria:**
  - Portal order list supports filtering by `order_name`.
  - Search is case-insensitive and returns only customer-owned orders.
  - Query latency remains acceptable on seeded test data.
- **Value:** Enables the first step in the user journey.
- **Size:** `3`

### User Story 2.2 - Create checkout session from Portal
- **Description:** As an Art Director, I want to initiate checkout for an order so payment can start.
- **Acceptance Criteria:**
  - Portal Backend resolves/creates open checkout attempt and forwards server-owned `Idempotency-Key` to Checkout create endpoint.
  - Checkout validates order eligibility and returns PSP redirect/session data.
  - Duplicate create requests with same idempotency key return same result.
- **Value:** Starts monetization flow and prepares payment capture.
- **Size:** `5`

### User Story 2.3 - Process PSP webhook and mark payment success
- **Description:** As Checkout Service, I need to process PSP callbacks to confirm payment result reliably.
- **Acceptance Criteria:**
  - PSP webhook endpoint validates provider signature.
  - Successful payment transitions order payment state to `PAID`.
  - Duplicate webhook events do not create duplicate payment effects.
  - Payment attempts are recorded in `payment_attempts`.
- **Value:** Converts initiated checkout into confirmed revenue event.
- **Size:** `8`
- **Tasks:**
  - Implement webhook signature verification.
  - Persist webhook event IDs for deduplication.
  - Add state transition guardrails.

### User Story 2.4 - Publish `order.paid` through transactional outbox
- **Description:** As Checkout Service, I need guaranteed event publication after payment success.
- **Acceptance Criteria:**
  - Payment status update and outbox insert happen in one DB transaction.
  - Outbox publisher retries transient failures.
  - Event published includes minimum contract fields from `datamodels.md`.
- **Value:** Ensures downstream systems react reliably after payment.
- **Size:** `8`

### User Story 2.5 - Consume `order.paid` and create Production order
- **Description:** As Production Service, I need to process paid orders exactly once so fulfillment can begin.
- **Acceptance Criteria:**
  - Production consumes `order.paid` and creates/updates `production_orders`.
  - Duplicate events are ignored via dedup table.
  - Production status reaches `RECEIVED` or `QUEUED` for successful intake.
- **Value:** Delivers primary business outcome: paid orders enter production.
- **Size:** `5`

---

## Epic 3: Customer Communication Completion

### User Story 3.1 - Send payment success email from `order.paid`
- **Description:** As a customer, I want confirmation email after successful payment.
- **Acceptance Criteria:**
  - Email service consumes `order.paid` and sends success email.
  - Email sending is idempotent by event or `(order_id, template)`.
  - Delivery status is recorded for troubleshooting.
- **Value:** Improves customer trust and reduces support contacts.
- **Size:** `5`

## Epic 4: Reliability and Duplicate Prevention Hardening

### User Story 4.1 - Enforce idempotency on all write APIs
- **Description:** As platform reliability, I need all write operations to be retry-safe.
- **Acceptance Criteria:**
  - Internal Checkout write endpoints require `Idempotency-Key`.
  - Public checkout start (`POST /api/v1/orders/{order_id}/checkout`) uses server-owned idempotency key resolved by Portal Backend.
  - Same key + same payload replays original response.
  - Same key + different payload returns `409`.
- **Value:** Prevents duplicate charges and inconsistent retries.
- **Size:** `5`

### User Story 4.2 - Add consumer dedup in downstream services
- **Description:** As downstream services, we need exactly-once effect on top of at-least-once delivery.
- **Acceptance Criteria:**
  - Production and Email consumers persist processed `event_id`.
  - Duplicate broker deliveries are no-op.
  - Metrics track dedup hits.
- **Value:** Prevents duplicate processing and notification errors.
- **Size:** `5`

### User Story 4.3 - Retry, DLQ, and poison message handling
- **Description:** As operations, we need controlled failure handling for async processing.
- **Acceptance Criteria:**
  - Exponential backoff retries configured for publisher and consumers.
  - Dead-letter queue configured for permanent failures.
  - Failed events include reason and correlation ID.
- **Value:** Improves resilience and recoverability under integration failures.
- **Size:** `8`

---

## Epic 5: Status Projection, Observability, and Go-Live Readiness

### User Story 5.1 - Build portal status projection updater
- **Description:** As Portal, I need a combined payment/fulfillment status projection for customer UI.
- **Acceptance Criteria:**
  - Projection table `order_status_projection` is updated from payment and production events.
  - Projection updates are monotonic by version/time.
  - Portal can render payment + fulfillment state from projection only.
- **Value:** Gives users reliable and fast order status visibility.
- **Size:** `8`

### User Story 5.2 - End-to-end integration test for happy path
- **Description:** As release owner, I need confidence that checkout reaches production with all side effects.
- **Acceptance Criteria:**
  - Integration test covers order search -> checkout -> payment success -> `order.paid` -> production status update.
  - Integration test validates email side effects.
  - Test is repeatable in CI and stable.
- **Value:** Reduces go-live risk and regression risk.
- **Size:** `8`

### User Story 5.3 - Add observability and runbook essentials
- **Description:** As operations team, we need enough telemetry to detect and resolve failures quickly.
- **Acceptance Criteria:**
  - Logs include `request_id`, `correlation_id`, `event_id` where applicable.
  - Metrics and alerts exist for webhook failures, outbox backlog, consumer failures, and DLQ depth.
  - Basic runbook documents incident triage and replay procedure.
- **Value:** Improves production support speed and system reliability.
- **Size:** `5`

---

## Suggested Iteration Cut

### Iteration 1 (Weeks 1-2)
- Epic 1 (all stories)
- Story 2.1, 2.2

### Iteration 2 (Weeks 3-4)
- Stories 2.3, 2.4, 2.5

### Iteration 3 (Weeks 5-6)
- Epic 3 (all stories)
- Story 4.1

### Iteration 4 (Weeks 7-8)
- Stories 4.2, 4.3
- Stories 5.1, 5.2, 5.3

## Definition of Done (Cross-Story)

- Code merged with peer review.
- API and event contracts documented and versioned.
- Automated tests added and passing in CI.
- Security checks for service authn/authz validated.
- Monitoring and alerting updated for new runtime behaviors.
