Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`). Files read for this review: `docs/adr/0002-publish-orderplaced-event-to-rabbitmq.md`, `docs/scenario.md`.

# ATAM-style review: ADR-0002 (RabbitMQ `OrderPlaced` event)

## 1. Quality-attribute scenarios

| # | Source · Stimulus · Environment · Response · Measure |
|---|---|
| S1 Performance | Shopper · places an order · 11.11 peak, 200 orders/s, MailFast at p95 1.5 s · `PlaceOrder` returns without waiting for MailFast, loyalty or analytics · p95 ≤ 500 ms |
| S2 Availability | MailFast, `loyaltyservice` or RabbitMQ is down · orders arrive · peak · orders complete, post-order work is deferred or flagged · 0 orders fail because of post-order actions |
| S3 Correctness | Broker redelivers an `OrderPlaced` event · normal operation · loyalty consumer processes it again · points credited once · points appear ≤ 1 minute after the order, exactly once |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Moving MailFast off the checkout path (S1) | MailFast p95 1.5 s alone is three times the 500 ms budget; the decision removes it from the critical path |
| Non-risk | Separate queues per consumer | An email backlog cannot delay loyalty points, which have a 1-minute deadline |
| Non-risk | Loyalty idempotency by `order_id` with a unique constraint | Directly addresses "exactly once" for redelivery |
| Sensitivity point | Email consumer rate (40/s) vs MailFast limit (50/s) | Raise it and MailFast rate-limits you; lower it and the backlog grows (at 200 orders/s the queue grows by 160/s) |
| Sensitivity point | Publish timeout (200 ms) and retry count | Publishing is on the checkout path; with a 300 ms baseline, a slow broker uses most of the 200 ms budget |
| Trade-off point | Eventual consistency for email and points vs checkout latency and availability | Fast, resilient checkout in exchange for emails that may arrive late and points that are not instant |
| Trade-off point | "Never fail an order" vs "never lose an event" | When the publish fails after payment, the ADR returns success and relies on an alert and manual replay: availability wins over completeness of loyalty and email |
| Risk | The event can be lost between charging the card and the publish (no outbox, no order store) | The customer pays but receives no email or points, and recovery is manual. At 200 orders/s even a small publish failure rate means real affected orders |
| Risk | Email backlog at peak | The ADR itself computes a queue of tens of thousands of messages; customers may wait about 40 minutes or more for a confirmation, and nothing sets a target for email delay |
| Risk | RabbitMQ becomes a single point of failure for post-order work | HA, node count and capacity are only listed as open questions, yet the whole design now depends on it |
| Risk | MailFast duplicate/idempotency unresolved | The consumer's "already sent" record has no stated store, and a crash between send and record can still produce a duplicate email |

## 3. Missing or weak

- **No failure handling for loyalty if `loyaltyservice` is down for longer than 1 minute:** the 1-minute requirement has no consumer-side alert or backlog target in the ADR except the Validation line.
- **Unsupported figures:** "a few ms" for the publish and the 10-minute peak length are assumptions; the 200 ms publish timeout is not justified.
- **Options not considered:** a transactional outbox with a database, or MailFast called through a dedicated async email worker without a general broker. Both are mentioned only as follow-ups.
- **Capacity:** no sizing for RabbitMQ at 200 events/s plus retries, no queue length limit or memory/disk alarm policy.
- **Ownership and on-call** for the broker and consumers are not assigned.
- **Privacy:** the event carries the customer's email address; the retention and access rules are not given.
- **"Loyalty reversal on refund" is out of scope** though points awarded for refunded orders are a likely business problem.

## 4. Verdict

**Accept with changes.** The direction is sound for latency and availability. The three most important changes:

1. Close the lost-event gap: add a durable order record and outbox (or state clearly who accepts the risk and how the manual replay works).
2. Set an explicit target for confirmation-email delay at peak, and size the consumer rate and backlog to meet it.
3. Specify RabbitMQ high availability and capacity, and name the owner who operates it.
