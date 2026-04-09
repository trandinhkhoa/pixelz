## System Diagram
Green(s) are the new components/flows

https://www.figma.com/board/gLn9AKIDUTix2d0EKHOBS1/Welcome-to-FigJam?node-id=0-1&t=p9x6uddWyK3kyxoD-1

---

## 0) **API Gateway**

**What it does**

* Entry point for frontend traffic
* Validate Google SSO/OIDC token at edge
* Apply rate limiting and request throttling
* Perform load balancing to backend services
* Enforce edge security controls (WAF-style checks)
* Route public API paths to **Portal Backend**
* Route PSP webhook path to **Checkout Service** with dedicated webhook policy

**Owns**

* edge traffic policy
* edge authentication and request filtering
* public API routing to internal services

---

## 1) **Portal Backend**

**What it does**

* Serve customer portal
* Order CRUD
* Search / filter orders by name
* Show checkout/payment status
* Read payment + fulfillment projections for customer UI

**Owns**

* customer-facing order operations
* order browsing and checkout entry point
* read model for combined order status

---

## 2) **Checkout Service**

**What it does**

* Receive checkout request
* Validate order can be checked out
* Create payment session / intent
* Route payment request through PSP adapter
* Handle PSP webhooks
* Mark payment result
* Prevent duplicate payment / duplicate processing
* Write outbox in same transaction as payment status update
* Publish payment events to message broker

**Modules inside Checkout Service**

* checkout orchestration
* PSP adapter layer
* webhook handling
* idempotency handling
* outbox publisher

**Owns**

* checkout lifecycle
* payment state
* PSP interaction state

**Database**

* checkout / payment table
* outbox table

---

## 3) **Message Broker**

**What it does**

* Carry async domain events from Checkout Service
* Decouple payment success from downstream systems
* Deliver events with at-least-once semantics

**Why **
- Once the payment is processed, we cannot expect the users to sit there and wait for the order to be fulfilled. So we need to display the payment result right away (synchronously) and then have Checkout service send the order asynchronously to Production service for it to be fulfilled in the background. The same goes for Email service.
- RabbitMQ is used because the current usecase in the exercise is simple. 1 queue for each downstream service.

**Main event**

* `order.paid`

**Delivery rule**

* Consumers must be idempotent (safe on duplicate `order.paid`)

---

## 4) **Production Service**

**What it does**

* Consume `order.paid`
* Push paid order into internal production/fulfillment flow
* Ensure order is not processed twice
* Retry if temporary failure happens

**Owns**

* production fulfillment lifecycle
* production acceptance / processing state

---

## 5) **Email Service**

**What it does**

* Consume `order.paid`
* Send successful payment / checkout email to customer
* Handle retries / duplicate event safety

---

# End-to-end flow

## What is outbox and why use it

### What it is

**Outbox** means Checkout Service writes business state change and event record in same DB transaction.

### Why it matters (what happens without it)

If service updates DB and publishes event as separate steps, failures create broken state:

* DB commit succeeds, event publish fails -> payment is `PAID` but downstream never knows (no production push/email)
* Event publish succeeds, DB commit fails/rolls back -> downstream acts on payment that is not actually committed
* Retry after partial failure can create duplicate publish without clear dedup source

### How this design achieves it

In this architecture, one DB commit does two writes:

* payment status updated to `PAID`
* `order.paid` inserted into outbox table

Then separate outbox publisher reads outbox table and publishes to **Message Broker**.

With outbox + idempotent consumers, system can safely retry publish and keep eventual consistency.

## What is idempotency and why use it

### What it is

**Idempotency** means the same command/event can be processed multiple times, but final business effect happens once.

### How it can happen
- Scenario 1: what if a customer clicks the “pay” button quickly twice?
- Scenario 2: The payment is successfully processed by the PSP, but the response fails to reach our payment system due to network errors. Then the user clicks the “pay” again.
### Why it matters (what happens without it)

Without idempotency, retries and duplicate deliveries can cause real business damage:

* one order can create multiple payment attempts or duplicate charges
* same paid event can trigger duplicate production jobs
* customer can receive duplicate success emails

### How this design achieves it

This architecture enforces idempotency at multiple layers:

* Portal Backend creates and stores a server-owned
`checkout_attempt_idempotency_key` . it is basically the id of the shopping
cart. It can be unique using database unique key constraint + auto
increment for the id of the shopping cart.
* repeated user clicks/retries for same open attempt reuse that same stored key
* Portal Backend forwards the same key to Checkout Service as `Idempotency-Key`
* Checkout Service reuses the same key when calling PSP (provider idempotency key), so duplicate external charge creation is prevented
- example scenarios:
    - Scenario 1: what if a customer clicks the “pay” button quickly twice?
        - when a user clicks “pay,” an idempotency key, which is the id of
the shopping cart, is sent to the `Checkout service`
        - For the second request, it’s treated as a retry because the
payment system has already seen the idempotency key. When we
include a previously specified idempotency key in the request
header, the payment system returns the latest status of the
previous request.
    - Scenario 2: The payment is successfully processed by the PSP, but the
response fails to reach our payment system due to network errors. Then
the user clicks the “pay” again.
        - the payment service sends the PSP a `Idempotency-Key` and the PSP
returns a corresponding token. The nonce uniquely represents the
payment order, and the token uniquely maps to the
`Idempotency-Key`. Therefore, the token uniquely maps to the
payment order.
        - When the user clicks the “pay” button again, the payment order is
the same, so the token sent to the PSP is the same. Because the
token is used as the idempotency key on the PSP side, it is able to
identify the double payment and return the status of the previous
execution.

## Dataflow when user requests checkout
I know this can be a sequence diagram but this would work just as fine.

https://www.figma.com/board/gLn9AKIDUTix2d0EKHOBS1/Welcome-to-FigJam?node-id=0-1&t=p9x6uddWyK3kyxoD-1

1. User starts Google sign-in via API Gateway `GET /auth/google/start`.
2. Google redirects to API Gateway callback `GET /auth/google/callback`; gateway routes to Portal Backend.
3. Portal Backend validates Google identity and issues first-party `portal-access-jwt`.
4. User creates order through API Gateway `POST /api/v1/orders` with `Authorization: Bearer <portal-access-jwt>`.
5. API Gateway validates JWT, applies rate limits, and routes to Portal Backend `POST /api/v1/orders`.
6. Portal Backend enforces authorization and persists order.
7. User searches orders through API Gateway `GET /api/v1/orders?name=<query>&page=<n>&page_size=<n>`.
8. API Gateway validates JWT/rate limits and routes to Portal Backend `GET /api/v1/orders?name=<query>&page=<n>&page_size=<n>`.
9. User opens order detail through API Gateway `GET /api/v1/orders/{order_id}`.
10. API Gateway routes to Portal Backend `GET /api/v1/orders/{order_id}`.
11. Portal Backend enforces ownership authorization.
12. User starts checkout through API Gateway `POST /api/v1/orders/{order_id}/checkout`.
13. API Gateway validates JWT/rate limits and routes to Portal Backend `POST /api/v1/orders/{order_id}/checkout`.
14. Portal Backend loads or creates an open checkout attempt for the order and resolves stable server-owned idempotency key for that attempt.
15. Portal Backend calls Checkout Service `POST /internal/v1/checkouts` over `mTLS` with service JWT and `Idempotency-Key: <checkout_attempt_idempotency_key>`.
16. PSP adapter resolves provider configuration (for example `provider = stripe`) and loads provider credentials from secure secret storage.
17. PSP adapter maps canonical request to provider-specific payload and headers (for Stripe: `POST /v1/checkout/sessions` or `POST /v1/payment_intents`) and includes same idempotency key to PSP.
18. PSP adapter sends HTTPS request with provider auth, timeout, and safe retry policy for transient failures.
19. PSP returns provider transaction identifiers and payment session data (`session_id`/`payment_intent_id`, hosted URL, expiry).
20. PSP adapter normalizes provider response into internal model (`provider_ref`, `redirect_url`, `expires_at`, `provider_status`) and returns it to checkout orchestration.
21. If provider response is uncertain (timeout/network), Checkout Service retries/query-by-idempotency before creating any new payment session.
22. Checkout Service persists provider refs + initial payment status (`PENDING`) and idempotency record in its database.
23. Checkout Service returns payment session details to Portal Backend (redirect URL, checkout ID, expiry).
24. Portal Backend returns checkout response to Portal UI.
25. User is redirected to PSP hosted page and completes payment (external flow, no Portal REST endpoint).
26. PSP sends webhook to API Gateway `POST /webhooks/psp/{provider}`.
27. API Gateway applies webhook policy and routes to Checkout Service `POST /internal/v1/psp/webhooks/{provider}`.
28. Checkout Service verifies PSP signature + replay protection, then marks payment `PAID` and writes outbox.
29. Checkout publishes `order.paid` to message broker (event-driven, no REST endpoint).
30. Production/Email consume `order.paid` (event-driven, no REST endpoint).
31. Duplicate events are handled through idempotency and dedup tables.
32. User checks latest status through API Gateway `GET /api/v1/orders/{order_id}/status`.
33. API Gateway validates JWT/rate limits and routes to Portal Backend `GET /api/v1/orders/{order_id}/status`.

## Why frontend does not call Checkout Service directly

### Security boundary

`Checkout Service` is internal-only and should not be exposed to browser traffic.
Frontend calls only public APIs through `API Gateway`.

### Authentication model

Internal checkout calls use `mTLS` + service JWT (`Portal Backend` -> `Checkout Service`).
Browser clients cannot safely hold service credentials.

### Business rule ownership

`Portal Backend` enforces customer authorization and checkout-attempt idempotency before calling checkout.
This prevents bypass of ownership checks and keeps customer-facing orchestration in one place.

### Contract stability

Frontend integrates with stable portal APIs, while checkout internals can evolve independently.
This reduces coupling between UI and payment domain internals.

---

Where:

* **Checkout Service** owns payment orchestration
* **Production Service** owns fulfillment of orders
* **Email** reacts asynchronously
* **Message Broker** connects them
* **Idempotency + outbox** protect against duplicates
* **API Gateway** handles edge auth/rate limit/load balancing
* **Portal Backend** CRUD app for Orders
