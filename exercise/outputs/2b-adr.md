Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`). Run b (example + context): read `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`.

# ADR-0002: Publish an `OrderPlaced` event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not given (see Open questions)

## Context and problem

Requirement R-12 adds three post-order actions: confirmation email through **MailFast** (replacing the mock `emailservice` logic), loyalty points through a new `loyaltyservice`, and an "order placed" analytics event. Marketing expects more actions next year (SMS, fraud scoring, a recommendations update).

Today `checkoutservice.PlaceOrder` (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`) runs synchronously: prepare cart items and shipping quote, `chargeCard` (`paymentservice`), `shipOrder` (`shippingservice`), `emptyUserCart`, then `sendOrderConfirmation` (`emailservice`). If the email call fails, only a warning is logged and the order still succeeds. The code has **no database, queue or order store**: the order id is a UUID created per request, and nothing is persisted.

Facts from `docs/scenario.md`:

- `PlaceOrder` p95 must stay **≤ 500 ms** (today approximately 300 ms), so the budget left is about **200 ms**.
- Peak load is **200 orders/s** during the 11.11 sale.
- MailFast: p95 **1.5 s**, p99 **8 s**, fails approximately **0.5 %** of calls, rate limit **50 emails/s**.
- Loyalty points: within **1 minute**, **exactly once**.
- An order must **never fail** because email, loyalty or analytics is down.
- Team: 4 developers, strong in Go and gRPC; two have used **RabbitMQ**. Everything runs on Kubernetes; the company prefers **open-source, self-hosted** components.

Calling MailFast synchronously would add p95 ≈ 1.5 s to every checkout, three times the whole latency budget. At 200 orders/s the demand is 4× MailFast's limit of 50 emails/s.

## Decision drivers

1. **Availability:** an order never fails because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s; no slow external call on the checkout path.
3. **Correctness:** loyalty points awarded exactly once, within 1 minute.
4. **Modifiability:** SMS, fraud scoring and recommendations updates should be added without changing `checkoutservice` each time.
5. **Operability and cost:** open-source, self-hosted, on Kubernetes, and learnable by a team of 4 (two know RabbitMQ).

## Considered options

1. Synchronous gRPC calls from `checkoutservice` to MailFast, `loyaltyservice` and the analytics sink
2. Fire-and-forget calls (goroutines) from `checkoutservice`, no broker
3. RabbitMQ (self-hosted): `checkoutservice` publishes one `OrderPlaced` event; email, loyalty and analytics consume from separate queues
4. Kafka (self-hosted) with the same event design

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | Simplest; same style as today; no new component | MailFast p95 1.5 s breaks the 500 ms p95 (about 300 ms + 1.5 s); 200 orders/s exceeds the 50 emails/s limit; a dependency being down can fail or stall the order; every new action edits `checkoutservice` |
| 2. Goroutine fire-and-forget | No new infrastructure; checkout doesn't wait | Work is lost if the pod restarts; no retry or backlog for the 150 emails/s over MailFast's limit; no way to add consumers without code changes; hard to observe |
| 3. RabbitMQ | Durable queues absorb the 11.11 burst; per-consumer rate limit (≤ 50/s for MailFast); retries and dead-letter queues; new action = new queue and consumer; open-source, self-hosted; two developers already know it | New stateful component to run and monitor; at-least-once delivery means consumers must be idempotent; the publish step itself can fail (see Risks) |
| 4. Kafka | Replayable log; very high throughput | Heavier to operate than needed for 200 events/s; nobody on the team is noted as having Kafka experience (Not evidenced); per-message retry and rate limiting are more work than in RabbitMQ |

## Decision

Use **RabbitMQ**. After a successful charge and shipment, `checkoutservice` publishes one `OrderPlaced` event (order id, user id, email, items, totals) with **publisher confirms** and a short timeout, and returns the order to the user. Three durable queues, bound to the event, feed independent consumers:

- **email consumer:** calls MailFast at a **fixed rate below 50/s** (for example 40/s for headroom), retries with backoff, and uses a dead-letter queue after N failures;
- **loyalty consumer:** `loyaltyservice` credits points keyed by `order_id` with a unique constraint, so a redelivered event cannot give points twice;
- **analytics consumer:** writes the event to the marketing sink.

Separate queues mean the email backlog cannot delay loyalty points. Future actions (SMS, fraud scoring, recommendations update) bind new queues to the same event.

Why not the others: option 1 and 2 break the latency and availability drivers, and option 4 is more than the load needs for a team that already knows RabbitMQ.

## Consequences

- **Positive:** the checkout path gains one broker publish (a few ms, to be measured) instead of a 1.5 s MailFast call (quality attribute: performance). A down MailFast, `loyaltyservice` or analytics sink no longer affects orders (availability). New post-order actions don't change `checkoutservice` (modifiability).
- **Positive:** the emails above MailFast's limit become a queue backlog, not errors. For an assumed 10-minute peak at 200 orders/s, about 96,000 messages queue up (160/s net growth × 600 s) and drain in about 40 minutes at 40/s. The 10-minute peak length is my assumption. Whether emails that late are acceptable is a product question (see Open questions) (reliability).
- **Negative:** one more stateful component to run, upgrade and monitor, plus `loyaltyservice` to build (operability, cost). Email and points are eventually consistent: they arrive after the response (consistency).
- **Risks and mitigations:**
  - *Lost event:* the order has no store today, so a transactional outbox is not possible without adding a database. If the publish fails after payment and shipping, the user is charged but no event exists. Mitigation: publish with confirms and a retry, accept the order anyway (never fail an order), and log failed publishes with the order id for reconciliation. A durable order store or outbox is a follow-up decision.
  - *Duplicate events:* consumers are idempotent by `order_id` (loyalty enforces it with a unique key). MailFast idempotency is not known, so a duplicate email is possible.
  - *Broker outage:* publishing fails fast and orders still complete; loyalty then misses its 1-minute target for the affected orders, which is an accepted risk to be reconciled.
  - *Poison messages:* dead-letter queue and alert.
  - *Sensitive data:* the event carries the user's email; restrict queue access.

## Validation

- Load-test checkout at **200 orders/s** (extend the Locust `loadgenerator`) and check `PlaceOrder` p95 ≤ 500 ms with the broker on; repeat with RabbitMQ stopped and with MailFast stubbed at p95 1.5 s / 0.5 % errors, and confirm that no order fails.
- Replay the same `OrderPlaced` event twice and check that points are credited once.
- Measure email queue depth and age during the load test, and alert when the oldest loyalty message is older than 30 s.
- Review after the 11.11 sale.

## Open questions

- Who are the deciders, and who owns RabbitMQ and the consumers in production?
- Is there (or will there be) a durable order store? Without one, a lost event cannot be recovered.
- How long does the 11.11 peak last, and how late may a confirmation email be during it (the backlog could take tens of minutes to drain)? Is the 40/s email rate acceptable?
- Does MailFast support an idempotency key, to avoid duplicate emails on retry?
- How should loyalty points handle refunds or cancelled orders (not in the scenario)?
- What are the fields, retention and privacy rules for the analytics event?
- How many RabbitMQ nodes (HA) are needed, and what is the budget?
- Does the team accept the failure case where payment succeeds but the event is not published?
