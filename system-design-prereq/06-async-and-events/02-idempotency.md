# Idempotency

> **TL;DR** — An operation is idempotent if doing it twice has the same effect as doing it once. Networks fail ambiguously (a timeout doesn't tell you whether the request ran), so anything that gets retried — HTTP calls, queue messages, webhooks — must be safe to repeat.

## How it works

```text
Client ──POST /payments  Idempotency-Key: 7f3c…──► Server
  │      (times out — did it charge?)              │ 1. INSERT key (unique) → claimed
  │                                                │ 2. charge, store response under key
  └──retry, SAME key──────────────────────────────►│ 3. key exists → return stored response
                                                   │    (no second charge)
```

- **Why it's needed:** retries after timeouts, at-least-once message delivery, webhook redelivery, double clicks, proxies and clients retrying automatically.
- **HTTP semantics (RFC 9110):** GET, HEAD, OPTIONS, PUT, DELETE are idempotent; POST isn't; PATCH isn't guaranteed.
- **Natural vs not:** `SET status = 'PAID'`, `PUT /users/42`, `DELETE` are naturally idempotent; `balance = balance + 100`, `INSERT` with a generated id and "send email" are not — make them idempotent with a key or constraint.

## Techniques

| Technique | How | Use for |
|---|---|---|
| Idempotency key | client sends a unique key per logical operation; server stores key → result | payment/order POSTs |
| Unique constraint / natural key | `UNIQUE(order_number)`; the duplicate insert fails or is ignored | creating business entities |
| Upsert | `INSERT … ON CONFLICT` (Postgres), `MERGE` (SQL Server), `ON DUPLICATE KEY UPDATE` (MySQL) | sync jobs, consumers |
| Conditional write | `UPDATE … WHERE version = 7`, `If-Match: <etag>` | state transitions, concurrent edits |
| Dedupe table (inbox) | record processed message ids in the same transaction as the effect | queue consumers |
| Absolute instead of relative | "set balance to 500" / "apply txn #123" instead of "+100" | counters, ledgers |

## Advanced

- **Idempotency-key protocol:** the client generates a UUID per *logical* operation and reuses it on every retry; the server stores `(key, request fingerprint, status, response)` and replays the stored response (same status and body). The IETF `Idempotency-Key` draft: missing key → 400, key reused with a different payload → 422, original still in flight → 409. Keys expire (Stripe prunes them after ≥ 24 h).
- **Claim first, atomically:** `INSERT` the key under a unique constraint *before* doing the work — the loser of a race gets a conflict instead of executing twice. Commit the result in the same DB transaction as the business change so "did the work" and "recorded the key" can't diverge.
- **External side effects:** a card charge or email can't join your DB transaction → split the request into atomic phases with recovery points (Stripe's approach), and pass your key downstream (payment providers accept one).
- **Consumers (inbox pattern):** queues deliver at-least-once → insert `message_id` into a processed table in the same transaction as the effect; duplicates hit the unique constraint and are acked/skipped. Keep entries longer than the broker's redelivery window.
- **Exactly-once = at-least-once + idempotency:** Kafka's idempotent producer (producer id + per-partition sequence numbers) only dedupes writes into Kafka; SQS FIFO dedupes by deduplication id within a 5-minute window. End to end, the consumer still has to be idempotent.
- **Retries and idempotency go together:** retry with timeouts + exponential backoff + jitter, and only auto-retry idempotent operations (client, proxy and gRPC retry policies included).
- **Idempotent ≠ same response:** a second DELETE may return 404 — the *state* matches, the response doesn't (unless you replay the stored one). **Idempotent ≠ safe:** DELETE changes state; GET is both.
- **Not a concurrency control:** idempotency handles the *same* request repeated; two *different* requests racing on one entity still need optimistic versions or locks.
- **Key scope & TTL:** namespace keys per client/tenant and operation; TTL must exceed the client's longest retry window.
- **Test it:** deliver every message twice in integration tests, replay requests in chaos tests, and assert exactly one side effect.

## Trade-offs

- ✅ Retries become safe → simpler error handling; at-least-once messaging is good enough; no double charges or duplicate orders.
- ❌ An extra write and storage per request for keys/dedupe rows, plus TTL cleanup.
- ❌ Atomicity around external calls is subtle; a check-then-act without a unique constraint still double-executes under concurrency.

## In the wild

Stripe, Adyen, Razorpay and PayPal idempotency keys · IETF `Idempotency-Key` header draft · AWS `ClientToken` / `ClientRequestToken` parameters · Kafka idempotent producer, SQS FIFO deduplication ids · .NET: MassTransit / NServiceBus outbox + inbox · Postgres `ON CONFLICT`, SQL Server `MERGE`, MySQL `ON DUPLICATE KEY UPDATE`.

## Gotchas

- "Check if it exists, then insert" without a unique constraint is a race — two retries both pass the check.
- A key generated per attempt (or on the server) defeats the purpose; it must be created once per logical operation, before the first attempt.
- Recording the key in a separate transaction after the side effect leaves a crash window that repeats the effect.
- At-least-once delivery + a non-idempotent consumer = duplicate emails and double inventory decrements.
- HTTP clients or proxies that auto-retry POSTs without keys create duplicates.

## Related

[Message Queues](01-message-queues.md) · [Webhooks](../02-apis-and-communication/05-webhooks.md) · [APIs](../02-apis-and-communication/01-apis.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Distributed Locks](../07-distributed-coordination/01-distributed-locks.md)

## Read more

- [Stripe — Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)
- [Brandur Leach — Implementing Stripe-like idempotency keys in Postgres](https://brandur.org/idempotency-keys)
- [AWS Builders' Library — Making retries safe with idempotent APIs](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)

<!-- nav -->

---

[← Message Queues](01-message-queues.md) · [Index](../README.md) · [Batch vs Stream Processing →](03-batch-vs-stream-processing.md)
