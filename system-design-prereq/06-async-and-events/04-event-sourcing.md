# Event Sourcing

> **TL;DR** — Store every state change as an immutable event in an append-only log instead of overwriting current state; current state is a fold over those events. You get an audit trail by construction, state as of any moment, and new read models built from full history.

## How it works

```text
Stream "account-42"  (append-only, versioned per aggregate)
  v1  AccountOpened   {owner: "Ana"}   ┐
  v2  MoneyDeposited  {amount: 100}    ├─ fold ─► Account {balance: 70, version: 3}
  v3  MoneyWithdrawn  {amount: 30}     ┘
        │ subscriptions, in global-position order
        ├─► projections: balances table, statements   (pure, safe to replay)
        └─► reactors: fraud check, emails             (side effects, run once)
```

- **Events are facts:** past-tense, immutable, business-level. `MoneyWithdrawn` records what happened; `WithdrawMoney` was the command that asked for it. The log is the source of truth; any current-state table is a derived cache.
- **One stream per aggregate** (`account-42`), versioned 1…n. The store also stamps each event with a global position so subscribers can ask for "everything after P".
- **Write path:** load the stream → fold (`state = apply(state, event)`) → check the command against that state (enough balance?) → append the new events with the expected version. Nothing is updated or deleted.
- **Read path:** queries hit projections built from the events, never a replay per request.
- **Analogies:** an accountant's ledger (entries are never erased; mistakes get a reversing entry), git (commits = events, checkout = state at any point), a database WAL (the log is the truth; tables are a cache of it).

## Event sourcing vs look-alikes

| | What it is | Events carry | Source of truth |
|---|---|---|---|
| Event sourcing | persistence model inside one service | fine-grained domain facts, kept forever | the event log |
| Event-driven architecture | services integrate by publishing and consuming events | integration events (notifications or state transfer) as public contracts | each service's own DB, often plain CRUD |
| CDC | tail a DB's log (WAL/binlog), publish row changes | row before/after: *what* changed, not *why* | the tables |
| Event streaming | durable partitioned log for transport and processing (Kafka, Kinesis) | whatever producers write; retention-limited by default | usually not — it's the pipe |

They combine: an event-sourced service publishes translated integration events (EDA) over Kafka (streaming); a CRUD service can emit a similar feed with CDC.

## Advanced

- **Optimistic concurrency:** append with expected version 3; if another writer already added v4, the store rejects the append and the command reloads, re-checks invariants and retries. That is what stops two concurrent withdrawals overdrawing the account, without locks. In SQL it's just `UNIQUE (stream_id, version)`.
- **Snapshots:** every N events, persist the folded state with its version; load = latest snapshot + later events. A disposable optimisation — rebuild it when the aggregate's shape changes. Huge streams are usually a modelling smell: prefer short-lived streams or "closing the books" (a new stream per period, opened with a carried-forward balance).
- **Projections (read models):** subscribers fold events into query-shaped stores (SQL tables, Elasticsearch, Redis); rebuild any time by dropping and replaying from position 0. That is CQRS, and event sourcing practically requires it. Async projections are eventually consistent: a read right after a command can be stale. Every cross-aggregate query ("all overdrawn accounts") needs one, because an event store is built to load streams by id, not to run ad-hoc queries.
- **Idempotent projections + checkpoints:** delivery is at-least-once. Save the last processed global position in the same transaction as the read-model update, or skip events whose version ≤ the version already stored on the row.
- **Projections vs reactors:** projections only write read models, so replaying them is safe. Reactors and integration handlers (send email, charge card, notify other services) must run once — separate subscriptions with their own checkpoints, excluded from rebuilds.
- **Position gaps in SQL-backed stores:** sequence values are assigned at insert, not commit, so position 101 can become visible before 100; a subscriber that checkpoints 101 skips 100 forever. Wait for gaps to fill or time out (Marten's high-water mark).
- **Schema evolution:** events are kept forever and never rewritten. Additive changes + tolerant reader (ignore unknown fields, default missing ones); a new event type/version when meaning changes; **upcasting** (convert old events to the current shape on read); copy-and-transform into a new stream only as a last resort. Never change what an existing event means.
- **Granularity:** model domain intent — `AddressCorrected` and `CustomerRelocated` trigger different rules — not CRUD (`CustomerUpdated`, `FieldChanged`), which loses the "why" and leaves an expensive audit log.
- **GDPR / right to be forgotten:** keep PII out of events (store a reference to a deletable PII record), or **crypto-shredding** — encrypt each person's PII with a per-person key and delete the key to erase it everywhere, backups included.
- **Corrections:** never edit history; append a compensating event (`DepositReversed`). A bug in the fold is fixed in code and replayed — the events were right.
- **Time travel:** fold up to a version or timestamp for "balance on 31 March"; replay production streams locally to reproduce a bug; run a new report over all past data on day one.
- **Kafka as the event store:** good transport, weak store — no expected-version append per entity (so no per-stream optimistic concurrency), loading one aggregate means scanning a partition (topic-per-entity doesn't scale), and default retention (7 days) or compaction discards history. Usual setup: an event store or Postgres holds the truth and publishes to Kafka.

## Trade-offs

- ✅ Audit trail by construction, state at any point in time, debugging by replay, new read models from existing history.
- ✅ Append-only writes: no update-in-place contention, and a ready-made event feed for integration.
- ❌ Every query needs a projection, projections lag, and the learning curve is steep.
- ❌ Versioning forever, GDPR handling, growing storage, and fixing bad data with compensating events instead of an `UPDATE`.
- ✅ Fits ledgers, payments, accounting, audit-critical or regulated domains, and domains with temporal rules (claims, bookings, logistics).
- ❌ Overkill for simple CRUD where nobody asks "how did we get here?" — use a normal table plus history (SQL Server temporal tables, triggers or CDC).

## In the wild

EventStoreDB (now KurrentDB): streams, expected-version appends, catch-up and persistent subscriptions, server-side projections · Marten (.NET; event store + projections on PostgreSQL) · Eventuous (.NET) · Axon Framework / Axon Server (Java) · DIY: an `events` table in Postgres or SQL Server with `UNIQUE (stream_id, version)` plus a global sequence · LMAX's exchange keeps all state in memory and rebuilds it from snapshots plus its journal of input events.

## Gotchas

- **ES ≠ event-driven architecture:** don't publish internal domain events as public integration contracts — translate them, or every internal refactor breaks consumers.
- **Rebuilding with reactors attached** re-sends emails and re-charges cards.
- **Folds must be deterministic:** no `DateTime.Now`, randomness or remote calls in `apply`; record the timestamp, price or FX rate in the event itself.
- **Events can't be rejected:** validation happens in the command, before the append; a consumer that disagrees must react with a new event of its own.
- **Don't validate against a projection** (e.g. unique usernames) — it's stale; enforce on the write side with a reservation stream or a unique-constraint table.
- **A queue is not an event store:** a RabbitMQ queue (unlike a RabbitMQ stream) deletes messages on ack — no replay, no per-entity reads.

## Related

[CQRS](05-cqrs.md) · [Idempotency](02-idempotency.md) · [Message Queues](01-message-queues.md) · [Batch vs Stream Processing](03-batch-vs-stream-processing.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [ACID Transactions](../04-databases/02-acid-transactions.md)

## Read more

- [Event Sourcing — Martin Fowler](https://martinfowler.com/eaaDev/EventSourcing.html)
- [What do you mean by "Event-Driven"? — Martin Fowler](https://martinfowler.com/articles/201701-event-driven.html)
- [Versioning in an Event Sourced System — Greg Young](https://leanpub.com/esversioning)

<!-- nav -->

---

[← Batch vs Stream Processing](03-batch-vs-stream-processing.md) · [Index](../README.md) · [CQRS →](05-cqrs.md)
