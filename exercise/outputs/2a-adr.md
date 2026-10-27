Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`). Run a (zero-shot): written from the requirement text and `docs/adr/template.md` only.

# ADR-0002: Decouple post-order actions from checkout with asynchronous messaging

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Not given (see Open questions)

## Context and problem

Requirement R-12 says that after an order is placed Online Boutique must (1) send the order confirmation email through MailFast, a new external email provider that replaces the mock `emailservice` logic; (2) award loyalty points through a new `loyaltyservice` (to be built); (3) record an "order placed" analytics event for the marketing team. Marketing expects more post-order actions next year (SMS, fraud scoring, a recommendations update).

The three actions have different dependencies (an external provider, a new internal service, an analytics sink) and the list will grow. We must decide how `checkoutservice` triggers them.

## Decision drivers

- Checkout must stay fast and must not fail because of a post-order action (reasoning from the requirement; no numbers are given in the requirement).
- Modifiability: adding SMS, fraud scoring or a recommendations update next year should not require changing `checkoutservice` each time.
- Reliability: loyalty points should not be lost or duplicated **[Assumption: the requirement does not say so]**.
- An external provider (MailFast) can be slow or unavailable.
- Team skills and operational constraints: Not given.

## Considered options

1. **Synchronous calls from `checkoutservice`** to MailFast, `loyaltyservice` and the analytics sink.
2. **Asynchronous events via a message broker**: `checkoutservice` publishes an `OrderPlaced` event; independent consumers handle email, loyalty and analytics.
3. **Background job queue / worker inside `checkoutservice`** (in-process or database-backed) that retries the three actions.

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous calls | Simplest; no new infrastructure; easy to reason about | Checkout latency grows with the slowest dependency; a failing dependency can fail or slow the order; every new action changes `checkoutservice` |
| 2. Message broker, `OrderPlaced` event | Checkout only publishes one event; consumers scale and fail independently; new consumers (SMS, fraud scoring) need no checkout change; retries and dead-letter queues are possible | New component to run; eventual consistency; at-least-once delivery means consumers must be idempotent; publish can fail or be lost unless done carefully |
| 3. In-process or DB-backed job queue | Fewer moving parts than a broker; keeps retries near the order data | Couples the jobs to checkout; harder to add independent consumers; scaling and ownership remain with checkout |

## Decision

Choose **option 2**: publish an `OrderPlaced` event to a message broker after the order is saved, and have the email, loyalty and analytics consumers process it independently. This best satisfies modifiability (new actions are new consumers) and keeps checkout from depending on MailFast or `loyaltyservice` availability.

## Consequences

- **Positive:** Checkout latency no longer includes MailFast or loyalty calls (quality attribute: performance). Failures in one consumer don't affect others or the order (availability). New post-order actions are added as new consumers (modifiability).
- **Negative:** One more component to operate and monitor (operability, cost). Email and points arrive slightly after the order (eventual consistency).
- **Risks and mitigations:** Duplicate events can double-award points → make the loyalty consumer idempotent by order id. A lost event can mean no email or points → publish reliably (for example a transactional outbox) and monitor the backlog. MailFast limits are unknown → add retries with backoff and a dead-letter queue.

## Validation

Load-test checkout with the broker enabled and compare latency with the current baseline; kill each consumer and confirm that orders still succeed; replay the same event twice and confirm that points are awarded once. Review after the first month in production.

## Open questions

- Checkout latency target and expected peak orders per second (no numbers given).
- MailFast latency, rate limits and failure rate.
- How quickly must loyalty points be awarded, and is "exactly once" required?
- Which broker is acceptable (self-hosted or managed) and what does the team already know?
- What is the analytics sink, and what data may it receive?
- Should an order be allowed to succeed when the event cannot be published?
- Who owns the new broker and consumers operationally?
- Deciders and review date.
