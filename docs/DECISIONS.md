## Orchestration Model for Checkout Completion — 2026-04-10
**Context:** We needed to define how downstream actions (production push and email) are triggered after successful payment.
**Options considered:** Option A: choreography via `order.paid` only; Option B: centralized orchestrator/saga service.
**Decision:** Option A (choreography via `order.paid` only).
**Reason:** It keeps the architecture simpler while still meeting current requirements when combined with idempotent consumers and retries.

## Source of Truth for Order State — 2026-04-10
**Context:** We needed clear ownership of payment and fulfillment status across Portal, Checkout, and Production domains.
**Options considered:** Option A: single service owns all state; Option B: Checkout owns payment status, Production owns fulfillment status, Portal reads projections.
**Decision:** Option B.
**Reason:** It preserves bounded contexts, avoids cross-domain coupling, and keeps customer-facing reads in Portal through projection/read models.

## Event Delivery Guarantee Strategy — 2026-04-10
**Context:** We needed to choose reliability semantics for asynchronous integration through the message broker.
**Options considered:** Option A: at-least-once delivery with idempotent consumers; Option B: exactly-once style processing.
**Decision:** Option A.
**Reason:** It is operationally practical, broadly supported by broker ecosystems, and robust when consumers enforce idempotency keys.

## Checkout Payment Persistence Shape — 2026-04-10
**Context:** We needed database structures that support reliable payment processing and auditable PSP interactions.
**Options considered:** Option A: `payments` only; Option B: `payments` + `payment_attempts`; Option C: `payment_attempts` only.
**Decision:** Option B (`payments` + `payment_attempts`).
**Reason:** It provides simple current-state reads and a complete attempt history for retries, debugging, and audits.

## Idempotency Persistence Strategy — 2026-04-10
**Context:** We needed durable deduplication for checkout API requests and webhook processing under retries and duplicate delivery.
**Options considered:** Option A: dedicated `idempotency_keys` table; Option B: business-table unique constraints only; Option C: cache-only dedup.
**Decision:** Option A (dedicated `idempotency_keys` table).
**Reason:** It gives explicit, durable replay protection and supports deterministic response reuse.

## Outbox Table Design — 2026-04-10
**Context:** We needed reliable event publication coupled with payment status changes.
**Options considered:** Option A: single generic `outbox_events`; Option B: outbox table per event type; Option C: direct publish without outbox.
**Decision:** Option A (single generic `outbox_events`).
**Reason:** It is a standard transactional outbox pattern with lower maintenance overhead and enough flexibility for new events.

## Portal Read Model Ownership — 2026-04-10
**Context:** We needed a customer-facing status view without violating service ownership boundaries.
**Options considered:** Option A: Portal-owned projection tables; Option B: live fan-out calls to Checkout and Production; Option C: shared reporting database.
**Decision:** Option A (Portal-owned projection tables).
**Reason:** It keeps UI queries fast, reduces runtime coupling, and preserves bounded-context ownership.

## Internal Communication Style for Checkout Flow — 2026-04-10
**Context:** We needed to define protocol boundaries between services in the checkout architecture.
**Options considered:** Option A: Portal Backend calls Checkout via REST and downstream integrations are event-driven; Option B: service mesh only; Option C: direct frontend-to-checkout integration.
**Decision:** Option A.
**Reason:** It aligns with choreography via `order.paid`, keeps downstream decoupled, and preserves simple synchronous request handling only where needed.

## Internal REST API Scope — 2026-04-10
**Context:** We needed to define what to include in the initial internal API design artifact.
**Options considered:** Option A: cross-service endpoints only; Option B: cross-service plus admin/ops endpoints; Option C: include public portal endpoints as well.
**Decision:** Option A.
**Reason:** It keeps the API contract focused on service-to-service behavior in current architecture while avoiding premature operational API expansion.

## API Versioning Strategy — 2026-04-10
**Context:** We needed a stable versioning approach for internal REST endpoints.
**Options considered:** Option A: URI versioning (`/v1`); Option B: header versioning; Option C: no explicit version.
**Decision:** Option A.
**Reason:** URI versioning is explicit, easy to document, and straightforward for routing and compatibility control.

## Service-to-Service Authentication Strategy — 2026-04-10
**Context:** We needed secure authentication and authorization for internal REST endpoints.
**Options considered:** Option A: mTLS plus service JWT; Option B: shared API key; Option C: trusted network only.
**Decision:** Option A.
**Reason:** It provides strong transport and identity security with clear service-level authorization controls.

## Idempotency Policy for Write APIs — 2026-04-10
**Context:** We needed deterministic retry safety for internal command endpoints.
**Options considered:** Option A: require `Idempotency-Key` for all `POST` and `PATCH`; Option B: require only on checkout/payment writes; Option C: optional idempotency.
**Decision:** Option A.
**Reason:** Uniform idempotency requirements reduce duplicate side effects and simplify client retry behavior.

## Delivery Strategy for Implementation Plan — 2026-04-10
**Context:** We needed a delivery model to sequence implementation across multiple services while producing business value early.
**Options considered:** Option A: MVP-first vertical slices; Option B: service-by-service completion; Option C: parallel capability streams.
**Decision:** Option A (MVP-first vertical slices).
**Reason:** It delivers end-to-end customer value early and reduces integration surprises by validating real flow sooner.

## Planning Cadence for Delivery Execution — 2026-04-10
**Context:** We needed timeboxing guidance for story planning and progress tracking.
**Options considered:** Option A: 1-week iterations; Option B: 2-week iterations; Option C: milestone-only planning.
**Decision:** Option B (2-week iterations).
**Reason:** It balances planning overhead and delivery predictability for multi-service backend work.

## User Story Sizing Method — 2026-04-10
**Context:** We needed a consistent method to estimate effort across epics and stories.
**Options considered:** Option A: Fibonacci story points; Option B: T-shirt sizing; Option C: ideal engineering days.
**Decision:** Option A (Fibonacci story points).
**Reason:** It supports relative estimation well for uncertainty-heavy distributed-system work.

## Portal API Documentation Scope Expansion — 2026-04-10
**Context:** We needed to extend REST API documentation to cover Portal Backend endpoints requested from the architecture portal section.
**Options considered:** Option A: add customer-facing Portal APIs; Option B: internal Portal APIs only; Option C: both customer-facing and internal APIs.
**Decision:** Option A (add customer-facing Portal APIs).
**Reason:** It makes the primary user journey contract explicit (search orders, request checkout, view status) and complements existing internal service contracts.

## API Gateway Placement for Public Traffic — 2026-04-10
**Context:** We needed edge capabilities (Google SSO auth, rate limiting, load balancing) while preserving internal service boundaries.
**Options considered:** Option A: Frontend -> API Gateway -> Portal Backend -> Checkout Service; Option B: Frontend -> API Gateway -> Checkout Service direct; Option C: Frontend -> API Gateway -> BFF Gateway -> internal services.
**Decision:** Option A.
**Reason:** It provides required edge controls immediately and keeps Checkout Service internal, reducing security exposure and domain coupling.

## Post-SSO Token Strategy at API Edge — 2026-04-10
**Context:** We needed to define which token API Gateway validates for customer API requests after Google SSO login.
**Options considered:** Option A: validate Google `id_token` directly on every API call; Option B: exchange Google identity for first-party `portal-access-jwt`; Option C: mixed token acceptance.
**Decision:** Option B (first-party `portal-access-jwt`).
**Reason:** It gives stronger control over claims, expiry, authorization mapping, and session/security policies without coupling API contracts to provider token shape.

## PSP Webhook Ingress Path — 2026-04-10
**Context:** We needed to decide whether PSP webhook callbacks should go directly to Checkout Service or pass through API Gateway.
**Options considered:** Option A: PSP webhook through API Gateway then routed to Checkout; Option B: direct public webhook endpoint on Checkout; Option C: separate dedicated webhook ingress service.
**Decision:** Option A.
**Reason:** It centralizes edge controls (TLS, filtering, rate guards) while preserving Checkout signature verification and replay protection in the domain service.

## Server-Owned Checkout Idempotency Key — 2026-04-10
**Context:** We needed to guarantee repeated checkout clicks/retries for the same order map to one checkout/payment attempt, including uncertain PSP response scenarios.
**Options considered:** Option A: Portal Backend creates/stores stable checkout-attempt idempotency key; Option B: client-generated key persisted in browser; Option C: deterministic derived key from order/version.
**Decision:** Option A.
**Reason:** Server ownership gives strongest consistency across refresh/device/tab, ensures same key reuse for retries, and allows Checkout Service to reuse the same key at PSP provider API for end-to-end duplicate-charge protection.

## Downstream Scope Simplification — 2026-04-10
**Context:** Product scope was narrowed and only production push plus payment success email should remain downstream side effects.
**Options considered:** Option A: keep only Production and Email consumers; Option B: keep all previous downstream consumers; Option C: disable all downstream side effects temporarily.
**Decision:** Option A.
**Reason:** It matches current product intent, reduces implementation scope, and keeps post-payment flow focused on operationally essential outcomes.
