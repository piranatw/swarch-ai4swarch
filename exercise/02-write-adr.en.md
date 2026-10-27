**Language:** [ภาษาไทย](02-write-adr.md) | English

# Task 2: Write an ADR for a new requirement (approximately 20 min)

> **Lecture link:** T3 (GPT-4 zero-shot was coherent but *less complete than humans*; few-shot closed much of the gap), AICH3 (context), W10 (message brokers).
> **Question:** how much does an AI-written ADR improve when you give it an example and the real context?

Read [`docs/scenario.md`](../docs/scenario.md) first: requirement **R-12**, what happens after an order is placed.

For each run, start a **new chat or session** and send the message below. Replace `<paste R-12 here>` with the quoted R-12 block from the scenario.

**Run a: zero-shot** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
```

**Run b: example + context** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
```

> Want the ADR in Thai? Add a line `language: Thai` to both messages (template headings and technical names stay in English). **Use the same language in Run a and Run b**, so the comparison is fair.

> In Run a, check that the agent really read nothing besides the template. If it opened `docs/scenario.md` anyway, note that in your prompt log, because it changes the comparison.

## Compare a and b

| Criterion | Run a | Run b |
|---|---|---|
| Uses the scenario's numbers (latency, peak, MailFast limits)? | No. It names no number and lists "latency target, peak orders/s, MailFast limits" under Open questions. (It is not a clean zero-shot: the agent had seen the numbers table, but kept them out.) | Yes: p95 ≤ 500 ms (about 300 ms today), 200 orders/s, MailFast p95 1.5 s / 0.5 % failures / 50 emails/s, loyalty within 1 minute. It computes that MailFast's 1.5 s alone is three times the latency budget and that 200/s is 4× the 50/s limit. |
| Options considered (list them) | 1. Synchronous calls; 2. Message broker with `OrderPlaced` event; 3. In-process or DB-backed job queue | 1. Synchronous gRPC; 2. Goroutine fire-and-forget; 3. RabbitMQ; 4. Kafka |
| Handles "exactly once" loyalty points and MailFast's rate limit? How? | Partly. Loyalty: idempotent consumer by order id. MailFast: only "retries with backoff and a dead-letter queue", because the limit is unknown. No rate limiting. | Yes. Loyalty: unique constraint on `order_id`. MailFast: a separate email queue consumed at a fixed rate below 50/s, retries with backoff, dead-letter queue; separate queues so the email backlog cannot delay loyalty. |
| Anything invented (a number, tool or fact not in the scenario)? | No numbers. But it recommends a "transactional outbox" without noticing that no order store exists, and picks "a broker" without a product. | A few assumptions, mostly flagged: a 10-minute peak (marked as assumption), "a few ms" publish cost (unmeasured), 40/s email rate (a design choice). It also says no one on the team is noted as knowing Kafka (labelled "Not evidenced"). It found a real fact from `main.go` that Run a could not: `checkoutservice` has no database or order store. |

## Take ownership: your final ADR

You are the architect. Copy `outputs/2b-adr.md` to `docs/adr/0002-<short-title>.md` and edit it **yourself** until you would sign it. Don't ask the agent to do this part.

- Fix anything wrong or invented, and add what is missing.
- Make the final call yourself. You may disagree with the AI.

You will defend this ADR in front of a review board in Part B, so make sure you can explain every decision in it.

| # | What you changed from the AI's draft | Why |
|---|---|---|
| 1 | Deciders: set to the student architect, reviewed by the group's board. Added an "Out of scope" paragraph (loyalty reversal on refunds, adding a durable order store, the future SMS / fraud / recommendation consumers). | The draft said "not given" and mixed open questions with things that this decision does not try to solve. A board needs to know what it is **not** being asked to approve. |
| 2 | Email consumer: fixed rate 40/s (not just "below 50/s"), dead-letter after 5 failures, and a record of already-sent `order_id`s kept by the consumer to avoid duplicate emails. Analytics: at-most-once accepted. | MailFast's idempotency is unknown, so I won't rely on it. 40/s leaves 20 % headroom for retries under the 50/s limit. A lost analytics event is less harmful than slowing checkout, so I don't pay for exactly-once there. |
| 3 | Lost-event risk: replaced "log it" with a concrete policy: 200 ms publish timeout + 1 retry, return the order anyway, log `order_event_unpublished` with the order id and payment transaction id, metric + alert on any non-zero count, manual replay from the payment record. The outbox is a stated follow-up. | `checkoutservice` has no database, so an outbox would add a new store, which is a bigger decision than this ADR. The draft hid this gap behind "log it"; my version says clearly that between charge and publish we can lose an event and how operations will notice. |
