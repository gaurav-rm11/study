# Command Query Responsibility Segregation (CQRS)

> **TL;DR** — Use one model to change state (commands) and a separate model to read it (queries), so each can be shaped, stored and scaled for its own job. Worth it where reads and writes differ sharply in shape or load; overkill for plain CRUD.

## How it works

```text
          ┌─► PlaceOrder (command) ─► handler ─► Write model / DB
          │      ◄── ack: id + version           (normalised, invariants)
 Client ──┤                                        │ events via outbox / CDC
          │                                        ▼ async → projection lag
          └─► GetOrders (query) ◄─────────────── Read model(s) / DB
                                                 (denormalised, one per screen)
```

- **Commands** express intent (`PlaceOrder`, `CancelOrder`). The handler validates, loads the aggregate, enforces invariants, persists, and returns an ack (id, version) — not data. Commands can be rejected.
- **Queries** return DTOs shaped for one screen or endpoint, have no side effects, and can bypass the domain model entirely (raw SQL, a view, a document).
- **Roots:** Bertrand Meyer's **CQS** (Eiffel) — a *method* either changes state or returns data, never both. Greg Young lifted the split from methods to whole models, optionally with separate stores.

## Levels

| Level | Write side | Read side | Sync | Read consistency |
|---|---|---|---|---|
| 1. Separate models, one DB | EF Core aggregates + handlers | Dapper / raw SQL / views on the same tables | none | strong |
| 2. Read replicas | primary | same schema on replicas | DB replication | eventual (replica lag) |
| 3. Separate read stores | relational DB or event store | denormalised tables, Elasticsearch, Redis, document DB | events via outbox, CDC or event-sourcing projections | eventual (projection lag) |

Level 1 is already CQRS; promote a read model to its own store only when a query workload demands it.

## Advanced

- **Sync mechanisms:** *outbox* — the write transaction also inserts the event into an outbox table; a relay publishes it to RabbitMQ/Kafka and projectors update the read stores. *CDC* — Debezium or SQL Server CDC tails the write DB's log: no app changes, but row diffs instead of intent. *Event-sourcing projections* — the event store is the write model. Never dual-write (save, then publish): a crash in between silently desyncs the sides.
- **Living with lag:** the command returns the new version plus the few fields the UI needs, so the client renders without querying; optimistic UI; **read-your-writes** by routing that user's reads to the write side for a few seconds; or the query takes a `minVersion` and waits until the projection's checkpoint reaches it (or times out and the client retries); push "updated" over WebSockets/SignalR.
- **Independent scaling:** reads usually dominate — scale replicas, caches and search clusters out while the write side stays small and strongly consistent; heavy reports stop contending with writes.
- **Many read models:** one per use case, each owned by its projector — order list (SQL table), search (Elasticsearch), counters (Redis), analytics (warehouse).
- **Rebuilding:** read models are disposable — replay events (ES) or backfill from the write DB (snapshot + CDC). Build v2 beside v1, switch queries once it has caught up, drop v1.
- **Command handling:** validate shape at the edge; enforce invariants in the aggregate; use the **command id** as an idempotency key (dedupe row in the same transaction) so a retry doesn't place two orders; send the **expected version** the user saw (ETag/`If-Match`, SQL Server `rowversion`) → 409 instead of a lost update.
- **Task-based vs CRUD UI:** a CRUD form sends the whole entity (`UpdateCustomer`) and the intent is lost; a task-based UI sends `ChangeShippingAddress` or `UpgradeToPremium`, which map to business rules and meaningful events. CQRS pays off with task-based UIs.
- **CQRS + event sourcing (ES):** a natural pair, but independent. ES practically needs CQRS (an event store can only load streams); CQRS doesn't need ES (all three levels work on plain tables).
- **Operating it:** alert on projection lag (head position − checkpoint, or age of the last applied event); park poison events in a dead-letter queue instead of stalling a projection; carry a correlation id from command → event → projector.
- **Fowler's warning:** for most systems CQRS adds risky complexity — apply it to the specific bounded contexts that need it, never to a whole system.

## Trade-offs

- ✅ Each side fits its job: a small, invariant-focused write model; fast, pre-joined read models.
- ✅ Scale and store reads independently with the best tech per view; add a view without touching the write side.
- ❌ More moving parts: outbox relay, broker, projectors and extra stores to run, monitor and pay for.
- ❌ Duplicated data and code (DTOs, mappings), eventual consistency the UX must absorb, and debugging "why is the list stale?" means tracing outbox → broker → projector.

## In the wild

.NET: MediatR-style command/query handlers (MediatR itself went commercial in 2025) · Wolverine (command bus + messaging) with Marten (PostgreSQL document/event store with projections) · EF Core for writes + Dapper for reads, as in the ordering service of Microsoft's eShopOnContainers · SQL Server indexed views / Postgres materialized views as cheap same-DB read models · Debezium → Kafka → Elasticsearch for search views · MassTransit / NServiceBus over RabbitMQ for commands and events · Java: Axon Framework.

## Gotchas

- **CQRS ≠ two databases ≠ eventual consistency:** level 1 is CQRS with strong consistency.
- **CQRS on simple CRUD** doubles the code for nothing.
- **Commands returning big payloads** — or queries with side effects (a GET that marks messages read) — blur the split and break caching and safe retries; return id, version, status.
- **Shared-schema coupling:** projectors reading the write tables directly, or command handlers reading the read store, mean neither side can evolve alone.
- **Never check invariants against the read model:** a uniqueness check on a lagging projection lets duplicates in; enforce it with a write-side constraint.
- **Projections must tolerate duplicates and reordering:** upsert by key; ignore events with a version ≤ the stored one.

## Related

[Event Sourcing](04-event-sourcing.md) · [Message Queues](01-message-queues.md) · [Idempotency](02-idempotency.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Replication](../04-databases/04-replication.md) · [Caching](../05-caching/01-caching.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md)

## Read more

- [CQRS — Martin Fowler](https://martinfowler.com/bliki/CQRS.html)
- [Apply simplified CQRS and DDD patterns in a microservice — Microsoft .NET docs](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/apply-simplified-microservice-cqrs-ddd-patterns)
- [CQRS Documents — Greg Young (PDF)](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)

<!-- nav -->

---

[← Event Sourcing](04-event-sourcing.md) · [Index](../README.md) · [Distributed Locks →](../07-distributed-coordination/01-distributed-locks.md)
