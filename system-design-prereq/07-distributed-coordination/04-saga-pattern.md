# Saga Pattern

> **TL;DR** — A saga runs a business transaction that spans services as a sequence of local transactions, each committing and triggering the next; if a step fails, compensating transactions undo the completed steps in reverse. It trades 2PC's locks and isolation for availability and eventual consistency.

## How it works

```text
CreateOrder ──► ReserveStock ──► ChargePayment ──► ShipOrder ──► CONFIRMED
 (PENDING)                          (pivot)       (retriable)
                                       │ ✗ card declined
CancelOrder ◄── ReleaseStock ◄─────────┘
 (REJECTED)     compensations run in reverse for the steps that committed
```

- Each step is a **local ACID transaction** in one service; it commits, then publishes an event or reply that triggers the next step.
- On failure, **compensating transactions** run for the already-committed steps, in reverse. They are semantic undos (cancel order, release stock, refund), not rollbacks — those commits were already visible to others.
- Origin: Garcia-Molina & Salem (1987), for long-lived transactions inside one database. With sub-transactions T1…Tn and compensations C1…Cn, a saga ends as either T1…Tn or T1…Tj, Cj…C1.

## Choreography vs orchestration

| | Choreography | Orchestration |
|---|---|---|
| Mechanism | each service reacts to events and emits the next (`OrderCreated` → Inventory → `StockReserved` → Payment) | a central orchestrator (state machine) sends commands and handles replies |
| Coupling | services subscribe to each other's events; cyclic dependencies creep in | participants know only their own commands; the orchestrator owns the flow |
| Visibility | flow is implicit, spread across services — "where is order 42?" is hard | state and flow in one place; easy to query, test and monitor |
| Complexity | trivial for 2–4 steps, then event spaghetti | one more component, but scales to many steps, branches and timeouts |
| Failure handling | each service listens for failure events and compensates itself | the orchestrator drives retries, timeouts and compensation centrally |
| Use when | few steps, linear flow, teams wanting autonomy | many steps, branching, deadlines, audit needs — most critical flows |

## Saga vs 2PC

| | 2PC | Saga |
|---|---|---|
| Guarantees | atomic and isolated (locks held until commit) | atomic eventually, via compensation; no isolation |
| Failure mode | blocking — a coordinator crash leaves participants in doubt, holding locks | each service commits alone and stays available |
| Needs | every participant supports prepare (XA); RabbitMQ, most NoSQL stores and HTTP APIs can't | a compensation per step, messaging, idempotency |
| Choose when | few XA-capable participants, short transactions, strong consistency required | microservices, third-party APIs, long-running or human steps |

## Advanced

- **Step types:** **compensatable** steps come first and each has an undo; the **pivot** is the go/no-go — once it commits, the saga must finish (usually the last step that can fail for business reasons; here, payment); **retriable** steps follow and can only fail transiently, so they're retried until they succeed. Put irreversible actions (shipping, emails, third-party calls with no undo) at or after the pivot; read-only steps need no compensation.
- **Compensations are business logic:** idempotent and retriable — a compensation can't "fail", only retry or escalate to a human. Some effects can't be undone: an email was sent → send a correction.
- **Uncertain outcomes:** after a timeout you don't know whether the step ran. Compensations must be safe for steps that never happened (refunding an unknown payment → no-op), and participants must remember cancellations so a late original request (reserve arriving after release) is rejected.
- **No isolation (ACD, not ACID):** others see intermediate state → **lost updates** (one saga overwrites another's change), **dirty reads** (reading stock a saga reserved and later released), **fuzzy reads** (two steps of one saga read different values).
- **Countermeasures:** *semantic lock* (`PENDING` states others must respect: wait, fail or treat as tentative); *commutative updates* (debit/credit, not "set balance"); *pessimistic view* (reorder steps so updates others might act on happen in retriable steps); *reread value* (verify unchanged before updating — an optimistic lock); *version file* (record operations so out-of-order ones can be reordered, e.g. a cancel arriving before its authorise); *by value* (pick saga or distributed transaction per request, by business risk).
- **Transactional outbox:** a step's state change and outgoing message must be atomic — insert the message into an outbox table in the same local transaction; a relay or CDC publishes it. Otherwise a crash between commit and publish stalls the saga.
- **At-least-once delivery:** brokers redeliver, so every step and compensation handler is idempotent (dedupe by message id, recorded in the same transaction).
- **Timeouts and deadlines:** each step gets a timeout (a scheduled message — MassTransit `Schedule`, NServiceBus `RequestTimeout`) and the saga an overall deadline (cancel orders unpaid after N minutes); on expiry, retry or compensate.
- **Persistent state, crash recovery:** the orchestrator saves saga state (step, data, version) on every transition, in the same transaction as consuming the message and writing the outbox, so any instance can resume after a crash. Concurrent messages for one saga → optimistic concurrency on the saga row.
- **Observability:** a saga/correlation id on every message, log line and trace; a saga log of each step and outcome; alerts on stuck sagas and on failed compensations (route those to a manual-fix queue).
- **Durable execution vs library sagas:** engines like Temporal and Azure Durable Functions persist each step's result and replay the history to rebuild state after a crash, so workflow code must be deterministic; library sagas (MassTransit, NServiceBus) are message-driven state machines persisted in your own DB.

## Trade-offs

- ✅ No global locks or blocking coordinator: services keep their own databases and stay available; works across third-party APIs and long-running or human steps.
- ✅ Failures become explicit business outcomes (cancellation, refund) that can be audited.
- ❌ No isolation — you design countermeasures and a "pending" UX.
- ❌ Every step needs a compensation, idempotency, timeouts and an outbox; failure paths are hard to test.
- ❌ Eventually consistent, and debugging spans services.

## In the wild

.NET: MassTransit state-machine sagas (persisted via EF Core, Redis, MongoDB…) and NServiceBus sagas, typically over RabbitMQ or Azure Service Bus · Temporal (and Cadence, from Uber), Azure Durable Functions, AWS Step Functions (`Retry`/`Catch` per state), Camunda / Zeebe (BPMN with compensation events) · Java: Axon Framework, Eventuate Tram Sagas (Chris Richardson) · Classic flows: e-commerce checkout, travel booking (flight + hotel + car), payments and refunds.

## Gotchas

- **Compensation ≠ rollback:** others may already have acted on the intermediate state (read the stock count, received the email).
- **Payments:** prefer authorise-then-capture — the authorisation acts as a semantic lock on funds, voiding it is a cleaner compensation than a refund, and capture becomes a retriable step after shipping.
- **Choreography sprawl:** past a handful of steps, events chain into implicit loops nobody owns — switch to orchestration.
- **Don't block an HTTP request on a saga:** return 202 with the `PENDING` order, then notify (webhook, WebSocket, polling).
- **Interview trap:** "use a saga" isn't an answer — name the pivot, each compensation, how isolation anomalies are handled, and where saga state lives.

## Related

[Two-Phase Commit (2PC)](03-two-phase-commit.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [ACID Transactions](../04-databases/02-acid-transactions.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Event Sourcing](../06-async-and-events/04-event-sourcing.md) · [Observability](../08-microservices-and-reliability/03-observability.md)

## Read more

- [Sagas — Garcia-Molina & Salem, 1987 (PDF)](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)
- [Pattern: Saga — microservices.io (Chris Richardson)](https://microservices.io/patterns/data/saga.html)
- [Saga distributed transactions pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/saga)

<!-- nav -->

---

[← Two-Phase Commit](03-two-phase-commit.md) · [Index](../README.md) · [Circuit Breaker →](../08-microservices-and-reliability/01-circuit-breaker.md)
