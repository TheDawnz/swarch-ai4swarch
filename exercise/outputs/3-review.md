Agent: GitHub Copilot using Copilot SDK in VS Code; underlying model not exposed.

# ATAM-style review of ADR-0002

## Quality-attribute scenarios

| Quality attribute | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| Performance and availability | Customer placing an order during the 11.11 sale | 200 orders/s arrive while MailFast has 1.5 s p95 and 8 s p99 latency | Production peak load; post-order consumers may be slow or unavailable | `checkoutservice` returns the order without waiting for MailFast, loyalty, or analytics | `PlaceOrder` p95 remains ≤ 500 ms; orders do not fail because a post-order dependency is down |
| Correctness | Loyalty program and marketing team | RabbitMQ redelivers an order event or the loyalty consumer restarts | At-least-once broker delivery and consumer failure | `loyaltyservice` deduplicates using a stable event/order ID and retries unfinished work | Each order receives points exactly once and within 1 minute |
| Availability and operability | MailFast consumer/operator | MailFast fails approximately 0.5% of calls and accepts at most 50 emails/s | Peak order rate of 200 orders/s | The email queue absorbs bursts; the consumer rate-limits, retries transient errors, and dead-letters unrecoverable work without affecting checkout | No more than 50 MailFast calls/s; failed actions are observable and recoverable; checkout remains available |

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Moving MailFast, loyalty, and analytics off the synchronous `PlaceOrder` path is a strong fit for the ≤500 ms latency and “order must never fail” goals. | MailFast's stated p95 and p99 are already greater than the checkout latency budget, so synchronous integration would directly threaten performance and availability. |
| Non-risk | Separate durable queues and independent consumers are a good extension point for SMS, fraud scoring, and recommendation updates. | Future actions can be added without increasing checkout coupling or changing its critical path. |
| Risk | The ADR requires a durable handoff/outbox but does not decide where the order and outbox are stored or how they share a transaction boundary. | Without an atomic order-plus-event handoff, a crash after order success can lose the event, violating all three R-12 actions while making recovery difficult. |
| Risk | “Exactly once” is described as a stable event ID plus consumer deduplication, but the ADR does not define the loyalty points write and deduplication operation as atomic. | A consumer can acknowledge or update one record and fail before the other; a retry can then either double-award points or incorrectly suppress them. The requirement is an exactly-once outcome, not merely an event identifier. |
| Sensitivity point | RabbitMQ queue age, consumer throughput, retry backoff, and MailFast's 50 calls/s limit determine whether loyalty completes within 1 minute. | At 200 orders/s, post-order work can accumulate faster than the MailFast limit, and retry traffic can consume capacity. The one-minute loyalty SLO needs explicit capacity and alert thresholds. |
| Sensitivity point | The decision assumes RabbitMQ can remain available or that the outbox can buffer publication during broker failure. | The “order never fails” goal depends on outbox capacity, replay behavior, and recovery time; these are not specified. |
| Trade-off point | Durable queues, retries, and an outbox improve reliability and availability but add infrastructure, storage, operational work, and potentially delayed actions. | The four-person team gets resilience and extensibility at the cost of more failure modes and operational burden. |
| Trade-off point | Rate-limiting MailFast protects provider compliance but increases email completion time during bursts. | At 200 orders/s and a 50 emails/s limit, email work can build up even when checkout remains healthy; queue policy and user expectations must distinguish email delay from order success. |

## Missing or weak

- **Missing option:** A transactional outbox plus a simpler self-hosted worker or queue should be compared explicitly with RabbitMQ. The ADR mentions an outbox as an implementation detail, but does not evaluate the combined alternative or its operational cost.
- **Missing option:** A managed external queue or email-delivery integration is not considered. The scenario prefers self-hosting, so rejecting it may be reasonable, but the cost and availability trade-off should be stated.
- **Weak decision boundary:** “After the order is durably accepted” is not defined. The ADR needs the order persistence system, transaction boundary, and behavior when the outbox is full or unavailable.
- **Weak exactly-once claim:** Idempotency can prevent duplicate effects only if the points update and deduplication marker have a safe atomic or transactional design. The ADR should specify that design and how lost acknowledgements are handled.
- **Weak capacity analysis:** The validation plan tests 200 orders/s but gives no queue depth, consumer throughput, retry budget, or one-minute backlog threshold for loyalty. The email rate limit is explicit, but its effect on queue growth is not quantified.
- **Weak MailFast integration:** The ADR does not define authentication, timeout, idempotency key, provider response handling, or whether an email is retried after an ambiguous timeout. These affect duplicate emails and recovery.
- **Weak failure policy:** Dead-letter handling is named but not defined: retention, alerting, replay ownership, and the distinction between transient and permanent errors are missing.
- **Unsupported system claim:** The ADR refers to the current `EMAIL_SERVICE_ADDR` and gRPC contracts as evidence, but this review has not independently verified those repository files. Those claims should remain explicitly tied to the cited files and be rechecked during implementation.

## Verdict

**Accept with changes.** The RabbitMQ direction is well aligned with the stated latency, availability, extensibility, Kubernetes, and team constraints, but the ADR is not ready to implement safely without resolving the reliability and correctness boundaries.

The three most important changes are:

1. Specify the durable order/event handoff: storage, transaction boundary, outbox retry/replay, broker outage behavior, and capacity limits.
2. Define an exactly-once loyalty outcome, including the atomic relationship between points balance update and processed-event deduplication, plus recovery from ambiguous failures.
3. Add capacity and operations targets for the one-minute loyalty SLO and MailFast's 50 calls/s limit: queue-depth thresholds, consumer rates, retry budgets, dead-letter/replay policy, and alerts.
