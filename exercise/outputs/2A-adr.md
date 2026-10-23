Agent: GitHub Copilot using Copilot SDK in VS Code; underlying model not exposed.

# ADR-0002: Use RabbitMQ for asynchronous post-order actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Online Boutique architecture team

## Context and problem

Requirement R-12 says that after an order is placed, Online Boutique must:

1. send the order-confirmation email through MailFast, replacing the mock `emailservice` logic;
2. award loyalty points through a new `loyaltyservice`; and
3. record an `"order placed"` analytics event for the marketing team.

Marketing expects more post-order actions next year, including SMS, fraud scoring, and a recommendations update.

Today, `checkoutservice.PlaceOrder` calls `paymentservice`, `shippingservice`, and then `emailservice` synchronously over gRPC and in sequence. If the email call fails, checkout logs a warning and the order still succeeds. The current deployment config exposes `EMAIL_SERVICE_ADDR=emailservice:5000` for `checkoutservice`, and the service contracts define `CheckoutService.PlaceOrder` and `EmailService.SendOrderConfirmation` in `system/online-boutique/protos/demo.proto`. This existing direct email path must be replaced for MailFast.

The post-order design must keep `PlaceOrder` p95 at or below 500 ms, compared with approximately 300 ms today, at a peak of 200 orders/s. MailFast has p95 latency of 1.5 s, p99 latency of 8 s, fails approximately 0.5% of calls, and permits only 50 emails/s. Loyalty points must be awarded within 1 minute and exactly once. An order must never fail because email, loyalty, or analytics is unavailable. The system runs on Kubernetes, the company prefers open-source self-hosted components, and the team has strong Go/gRPC skills with two developers experienced with RabbitMQ.

## Decision drivers

- **Availability:** order placement must not depend on email, loyalty, or analytics availability.
- **Performance:** `PlaceOrder` p95 must remain at or below 500 ms at 200 orders/s.
- **Correctness:** loyalty points must be awarded within 1 minute and exactly once.
- **Extensibility:** adding future post-order actions should not require changing the checkout workflow.
- **Operability and cost:** the solution must run on Kubernetes and favor open-source, self-hosted components.
- **Team fit:** the implementation should use skills already available in the team.

## Considered options

1. Publish a `PostOrder` event to a self-hosted RabbitMQ exchange, with durable queues and one consumer per action.
2. Keep synchronous gRPC calls from `checkoutservice` to MailFast, `loyaltyservice`, and an analytics endpoint, with bounded timeouts and retries.
3. Add a self-hosted event-log component such as Apache Kafka and have action consumers process the order event.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. RabbitMQ event exchange and action queues | Removes MailFast and other action latency from `PlaceOrder` (performance); a broker outage can be handled by retrying or buffering publication rather than failing the order (availability, subject to the publication design); separate queues allow MailFast to be rate-limited to 50 emails/s and retried independently (operability); supports adding SMS, fraud scoring, and recommendation consumers without changing checkout (modifiability); two developers already know RabbitMQ (team fit); open-source and self-hosted on Kubernetes (cost/operability). | Requires operating RabbitMQ (operability/cost); delivery is at-least-once in normal broker workflows, so consumers need durable event IDs and idempotency, especially for exactly-once loyalty points (correctness); an order-event publication/outbox mechanism is needed to avoid losing an event between order success and publish (reliability). |
| 2. Synchronous gRPC calls | Fits the team's Go/gRPC skills and is direct to implement (team fit/modifiability); no broker to operate (cost/operability); request/response errors are straightforward to observe (operability). | MailFast's 1.5 s p95 and 8 s p99 exceed the checkout latency budget if included in the request path (performance); downstream failures and rate limits complicate order availability (availability); future actions increase checkout coupling and latency (modifiability); exactly-once points require distributed coordination or idempotency anyway (correctness). |
| 3. Kafka event log and action consumers | Durable replayable event log supports independent consumers and future actions (modifiability/reliability); partitions can scale high-volume event processing (performance). | Adds a larger operational footprint than required for the stated workload (cost/operability); the team has documented RabbitMQ experience rather than Kafka experience (team fit); rate limiting MailFast and implementing exactly-once loyalty still require consumer logic (correctness); Kubernetes operation and monitoring are more complex (operability). |

## Decision

Choose **Option 1: publish a post-order event to a self-hosted RabbitMQ exchange and process it through independent action queues**.

`checkoutservice` should complete the order without waiting for MailFast, loyalty, analytics, or future action consumers. After the order is durably accepted, it should publish one event containing a stable order/event identifier and the data required by the post-order actions. The email, loyalty, and analytics consumers should each have their own durable queue. The MailFast consumer should limit outbound calls to 50 emails/s and retry transient failures. Every consumer should use the stable event/order identifier for idempotency; in particular, `loyaltyservice` must persist a processed-event record or equivalent deduplication state so redelivery cannot award points twice.

The implementation must include a durable handoff between successful order placement and event publication, such as an outbox owned by `checkoutservice` or an equivalent transactionally durable mechanism. The exact storage technology for that handoff is not evidenced by the provided system files and remains an open question. This handoff is necessary because an in-memory publish after the order response could lose the event if the process fails.

The current direct `checkoutservice` → `emailservice` dependency is therefore replaced by the event-driven MailFast consumer. The existing Kubernetes manifest currently configures `EMAIL_SERVICE_ADDR=emailservice:5000`; that setting must be removed or replaced as part of implementation rather than retained as a second email path.

## Consequences

- **Positive:** `PlaceOrder` no longer waits for MailFast's slow or failed calls (performance and availability).
- **Positive:** Independent queues isolate MailFast's 50 emails/s rate limit and retries from loyalty and analytics processing (availability and operability).
- **Positive:** New post-order actions can subscribe to the event without adding synchronous calls to checkout (modifiability).
- **Positive:** A stable event ID plus consumer deduplication can satisfy the exactly-once loyalty outcome despite broker redelivery (correctness).
- **Positive:** RabbitMQ is open-source, self-hostable on Kubernetes, and familiar to part of the team (cost, operability, and team fit).
- **Negative:** The system gains a RabbitMQ cluster and asynchronous failure modes to operate (operability and cost).
- **Negative:** Post-order actions become eventually consistent; email and analytics may complete after the order response (user experience and observability).
- **Negative:** The outbox or equivalent durable handoff adds storage and implementation complexity (reliability and modifiability).
- **Risks and mitigations:** RabbitMQ delivery is not by itself exactly-once; use durable queues, acknowledgements, retry/dead-letter handling, a stable event ID, and idempotent consumers (correctness and availability). Monitor queue age and consumer lag so loyalty completion stays within 1 minute (operability).

## Validation

- Load-test `PlaceOrder` at 200 orders/s and verify p95 latency is at most 500 ms while MailFast is configured with its 1.5 s p95 and 8 s p99 latency profile (performance).
- Stop MailFast, loyalty, and analytics consumers during order placement and verify orders still succeed; restore them and verify queued work drains (availability).
- Inject duplicate deliveries and consumer restarts, then verify each order receives loyalty points exactly once and within 1 minute after recovery (correctness).
- Apply the MailFast limit and verify no more than 50 email calls/s are emitted while failed calls are retried and eventually dead-lettered when appropriate (operability).
- Add a representative future consumer without changing `checkoutservice` and verify it receives the existing event (modifiability).
- Review RabbitMQ cluster sizing, durable storage, backup, monitoring, and failure recovery before production adoption (operability and cost).

## Open questions

- What database or durable storage is available for the `checkoutservice` outbox and loyalty deduplication record? Not evidenced in the provided `system/` files.
- What exact MailFast API, authentication, payload, and retry contract should the email consumer use? Not evidenced in the provided files.
- What analytics destination and event schema should receive the `"order placed"` event? Not evidenced in the provided files.
- What fields must the post-order event contain, and which fields may include payment or personal data? Not evidenced in the provided files.
- What RabbitMQ deployment topology, durability, retention, dead-letter policy, and operational ownership are required? Not evidenced in the provided files.
- How should an order be considered “durably accepted” relative to the existing checkout implementation? The provided files show the gRPC service contract and configured service addresses, but not the persistence transaction boundary.
