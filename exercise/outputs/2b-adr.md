Agent: GitHub Copilot using Copilot SDK in VS Code; underlying model not exposed.

# ADR-0002: Use RabbitMQ for asynchronous post-order actions

- **Status:** Proposed
- **Date:** 2026-10-21
- **Deciders:** Online Boutique architecture team

## Context and problem

Requirement R-12 requires Online Boutique to perform three actions after an order is placed:

1. send the order-confirmation email through MailFast, replacing the mock `emailservice` logic;
2. award loyalty points through a new `loyaltyservice`; and
3. record an `"order placed"` analytics event for the marketing team.

Marketing expects more post-order actions next year, including SMS, fraud scoring, and a recommendations update.

The current flow has `checkoutservice.PlaceOrder` call `paymentservice`, `shippingservice`, and then `emailservice` synchronously over gRPC, in sequence. If the email call fails, checkout logs a warning and the order still succeeds.

The new design must keep `PlaceOrder` p95 at or below 500 ms, compared with approximately 300 ms today, at a peak of 200 orders/s. MailFast has p95 latency of 1.5 s, p99 latency of 8 s, an approximate failure rate of 0.5% of calls, and a rate limit of 50 emails/s. Loyalty points must be awarded within 1 minute and exactly once. An order must never fail because email, loyalty, or analytics is unavailable. The system runs on Kubernetes, the company prefers open-source self-hosted components, and the four-person team is strong in Go and gRPC; two developers have used RabbitMQ.

## Decision drivers

- **Availability:** post-order action failures must not fail order placement.
- **Performance:** `PlaceOrder` p95 must remain at or below 500 ms at 200 orders/s.
- **Correctness:** loyalty points must be awarded within 1 minute and exactly once.
- **Modifiability:** future post-order actions should be addable without changing the checkout flow.
- **Operability and cost:** the solution must fit Kubernetes and the preference for open-source, self-hosted components.
- **Team fit:** the design should use skills available to the team.

## Considered options

1. Publish a post-order event to a self-hosted RabbitMQ exchange, with a durable queue for each action.
2. Keep post-order integrations as synchronous calls from checkout, using bounded timeouts and retries.
3. Use a self-hosted Kafka event log with independent consumers for each action.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. RabbitMQ event exchange and action queues | Removes MailFast latency from the checkout request (performance); isolates email, loyalty, and analytics failures from order placement (availability); separate queues can independently retry actions and enforce MailFast's 50 emails/s limit (operability); future consumers can be added without changing checkout (modifiability); RabbitMQ is familiar to two developers and is open-source/self-hosted (team fit, cost). | Adds a broker to operate (operability and cost); redelivery requires idempotent consumers and a stable event ID, especially for loyalty (correctness); a durable handoff is needed so an order event is not lost between order success and publication (reliability). |
| 2. Synchronous calls from checkout | Simple request/response interactions and no broker to operate (cost, team fit); bounded calls are straightforward to expose and monitor (operability). | MailFast's 1.5 s p95 and 8 s p99 exceed the checkout latency budget if awaited (performance); downstream failures and rate limits increase coupling to checkout (availability); every future action adds more coupling and latency (modifiability); exactly-once loyalty still needs idempotency or coordination (correctness). |
| 3. Kafka event log and action consumers | Durable replay and independent consumers support future actions (reliability and modifiability); partitioning can support high event throughput (performance). | It adds more operational complexity than the stated requirement needs (operability and cost); the stated team experience is with RabbitMQ, not Kafka (team fit); rate limiting and exactly-once loyalty still require consumer logic (correctness). |

## Decision

Choose **Option 1: use a self-hosted RabbitMQ exchange with independent durable queues for post-order actions**.

After the order is durably accepted, `checkoutservice` should publish one post-order event containing a stable order/event identifier and the data required by the consumers. Separate consumers should process the email, loyalty, and analytics actions. The MailFast consumer should limit calls to 50 emails/s and retry transient failures. Every consumer should be idempotent using the stable event/order identifier. `loyaltyservice` must persist a processed-event record or equivalent deduplication state so redelivery cannot award points twice.

The implementation must provide a durable handoff between successful order placement and event publication, such as an outbox owned by `checkoutservice` or an equivalent durable mechanism. The event must not exist only in memory after returning the order response, because a process failure could otherwise lose the post-order actions.

The email action should replace the current mock email behavior with a MailFast consumer. Checkout should not wait for MailFast, `loyaltyservice`, analytics, or future action consumers before returning a successful order response.

## Consequences

- **Positive:** `PlaceOrder` is not on the critical path for MailFast's slow or failed calls (performance and availability).
- **Positive:** Independent queues isolate MailFast's rate limit and retries from loyalty and analytics (operability and availability).
- **Positive:** New actions such as SMS, fraud scoring, and recommendation updates can consume the event without adding calls to checkout (modifiability).
- **Positive:** Stable event IDs and idempotent consumers provide a path to exactly-once loyalty outcomes despite redelivery (correctness).
- **Positive:** RabbitMQ fits the open-source, self-hosted preference and existing team experience (cost and team fit).
- **Negative:** RabbitMQ adds infrastructure, monitoring, backup, and failure-recovery responsibilities (operability and cost).
- **Negative:** Email and other actions become eventually consistent and may complete after the order response (user experience and observability).
- **Negative:** Durable event publication and loyalty deduplication add implementation and storage complexity (reliability and modifiability).
- **Risks and mitigations:** RabbitMQ does not by itself guarantee exactly-once processing. Use durable queues, acknowledgements, retry and dead-letter handling, stable event IDs, and idempotent consumers. Monitor queue age and consumer lag to ensure loyalty completion within 1 minute (correctness and operability).

## Validation

- Run a load test at 200 orders/s and verify `PlaceOrder` p95 is at most 500 ms while MailFast exhibits its stated 1.5 s p95 and 8 s p99 latency (performance).
- Make MailFast, loyalty, and analytics unavailable during order placement and verify that orders still succeed; restore consumers and verify queued actions drain (availability).
- Inject duplicate events and restart the loyalty consumer, then verify each order receives points exactly once and within 1 minute (correctness).
- Verify the MailFast consumer emits no more than 50 calls/s and handles the stated 0.5% call-failure rate with retry and dead-letter behavior (operability).
- Add a representative future consumer without changing the checkout workflow and verify it receives the post-order event (modifiability).
- Test RabbitMQ failure and recovery with durable queues and the chosen handoff mechanism, verifying that accepted orders do not lose their post-order events (reliability).

## Open questions

- Which durable storage technology should implement the `checkoutservice` outbox and the loyalty deduplication record?
- What exact MailFast API, authentication, payload, timeout, and retry contract should the email consumer use?
- What analytics destination, schema, retention period, and delivery guarantee are required?
- Which fields belong in the post-order event, and what personal or payment data must be excluded or protected?
- What RabbitMQ topology, replication, durable-storage, dead-letter, backup, and operational-ownership requirements apply?
- How is an order defined as “durably accepted” relative to the current checkout transaction boundary?
