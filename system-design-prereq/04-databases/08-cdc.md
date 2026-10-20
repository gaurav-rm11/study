# Change Data Capture (CDC)

> **TL;DR** — CDC captures every committed row-level change (insert, update, delete) in a database — ideally by reading its transaction log — and publishes it as an ordered stream of events. It keeps caches, search indexes, warehouses, read models and other services in sync without fragile dual writes.

## How it works

```text
App ──► DB ──► WAL / binlog ──► Debezium ──► Kafka ──┬──► search index
                            (Kafka Connect)          ├──► cache invalidation
                                                     ├──► warehouse / lake
                                                     └──► read models, other services
```

- The database already writes every committed change to its log for crash recovery and replication. A log-based connector reads that log like a replica, turns each row change into an event, publishes it (typically one topic per table, keyed by primary key) and checkpoints its log position so it resumes where it stopped.
- The writing app doesn't change; each consumer reads at its own pace and can replay the stream.

## Capture methods

| Method | Mechanism | Pros | Cons |
|---|---|---|---|
| Log-based | Postgres WAL logical decoding (`pgoutput`), MySQL row-based binlog, SQL Server CDC (log reader fills change tables), MongoDB change streams (oplog), DynamoDB Streams | near-zero write overhead; every change incl. deletes; commit order; low latency | DB-specific setup and privileges; log retention to manage |
| Trigger-based | triggers copy each change into a shadow table that gets polled | works without log access; full old/new rows | an extra write inside every transaction; triggers to maintain |
| Query / polling | `WHERE updated_at > :last_seen` | simple; read access only | misses hard deletes and intermediate states; adds query load; rows committed late with an older `updated_at` get skipped |

## Advanced

- **The dual-write problem:** `SaveChanges(); Publish();` isn't atomic — a crash between them loses the event (or publishes one for a rolled-back write), and concurrent requests can publish out of order.
- **Transactional outbox:** in the *same* local transaction, write the business row and an `outbox` row (aggregate type and id, event type, payload). A relay publishes outbox rows: CDC tailing the log (Debezium's **outbox event router** routes by aggregate type and keys by aggregate id) or a polling publisher (MassTransit's EF Core outbox). With log-based CDC the outbox row can be deleted in the same transaction — the log already holds the insert.
- **Event anatomy:** key = primary key; value = `before` and `after` row images, `op` (`c`, `u`, `d`, `r` for snapshot reads), `source` (database, table, transaction id, log position — Postgres LSN, MySQL binlog file/position or GTID) and timestamps (commit vs processing → lag). A delete is followed by a **tombstone** (key, null value) so Kafka **log compaction** can drop the key; a compacted topic then holds the latest state of every row.
- **Snapshot, then stream:** a new connector snapshots existing rows, noting the log position at the start, then streams from that position — no gap, possibly duplicates. **Incremental snapshots** (Debezium, based on Netflix's DBLog watermark algorithm) read tables in primary-key chunks *while* streaming; low/high watermarks discard chunk rows superseded by concurrent changes, so you can backfill, re-snapshot or add tables without pausing.
- **Ordering:** only per Kafka partition — keying by primary key keeps each row's changes in order, but there's no order across rows or tables. A multi-row transaction arrives as separate events, possibly on several topics; Debezium can emit transaction metadata (BEGIN/END markers, transaction id) for consumers that must regroup it.
- **At-least-once:** after a crash the connector re-reads from its last committed offset. Consumers must be idempotent: upsert by key and skip events whose LSN/version is ≤ the one already applied.
- **Schema evolution:** DDL flows straight downstream — a dropped or renamed column or a type change can break consumers. Use a schema registry (Confluent, Apicurio) with Avro/Protobuf and compatibility rules; roll out changes expand → migrate → contract.
- **Coupling:** raw table CDC turns your internal schema into a public API. Fine inside one team's boundary (its cache, search index, warehouse); across services, publish domain events through an outbox.
- **Postgres ops:** a logical replication slot retains WAL until its consumer confirms it — a stalled or abandoned connector fills the primary's disk. Monitor slot lag (`pg_replication_slots`), cap it with `max_slot_wal_keep_size` (PG13+), and enable Debezium heartbeats so slots on quiet databases keep advancing. `before` holds at most the old primary key unless the table has `REPLICA IDENTITY FULL`. Slots didn't survive failover before PG17's failover slots.
- **MySQL and general ops:** needs a row-based binlog (`binlog_format=ROW`, the default since 5.7.7); a connector down longer than binlog retention loses its position → full re-snapshot. Kafka Connect stores offsets in an internal topic — lose them and you re-snapshot or skip changes. Alert on end-to-end lag (commit timestamp → consumer), not just connector health.
- **SQL Server — CDC vs Change Tracking:** CDC is asynchronous: a capture job (SQL Server Agent) reads the log into change tables with full before/after values (cleanup default: 3 days), and Debezium's SQL Server connector reads those tables. Change Tracking is synchronous and records only *which* primary keys changed plus a version — you re-query current values; no history.
- **Use cases:** cache invalidation, search indexing, CQRS read models, real-time analytics and ETL into a warehouse or lake, audit trails, and strangler-fig migrations (stream the legacy database into the new service's store until cutover).

## Trade-offs

- ✅ Near-real-time propagation with no dual writes and no app changes; captures deletes and every intermediate change; a replayable stream decouples consumers from the source database's load.
- ❌ Another distributed pipeline (Kafka, Connect, schema registry) to run, monitor and backfill; downstream is eventually consistent.
- ❌ Raw CDC couples consumers to table schemas; setup is database-specific, and log retention can hurt the primary.

## In the wild

Debezium (MySQL, Postgres, SQL Server, Oracle, MongoDB, Db2) on Kafka Connect, or Debezium Server for non-Kafka sinks (Kinesis, Pub/Sub, Event Hubs, Redis Streams, RabbitMQ) · AWS DMS (full load + ongoing replication) · Fivetran, Airbyte (CDC into warehouses) · Maxwell's Daemon (MySQL binlog → JSON) · Oracle GoldenGate · Google Datastream · DynamoDB Streams + Lambda · Cosmos DB change feed · MongoDB change streams · Postgres native logical replication · Netflix DBLog · .NET: SQL Server CDC / Change Tracking, MassTransit outbox.

## Gotchas

- **Polling `updated_at` isn't real CDC** — no hard deletes (use soft deletes), no intermediate states, and late-committing transactions slip past the watermark.
- **CDC isn't event sourcing** — events are row diffs derived from state, not intent-rich domain events; the table remains the source of truth.
- **An abandoned replication slot is a disk bomb** — drop the slot when you decommission a connector.
- **Initial snapshots of big tables** take hours and load the source; schedule them and prefer incremental snapshots.
- **Interview trap:** "write to the DB, then publish to RabbitMQ/Kafka" — the expected fix is outbox + CDC (or a polling relay) + idempotent consumers.

## Related

[Message Queues](../06-async-and-events/01-message-queues.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Event Sourcing](../06-async-and-events/04-event-sourcing.md) · [CQRS](../06-async-and-events/05-cqrs.md) · [Replication](04-replication.md) · [Caching Strategies](../05-caching/02-caching-strategies.md) · [Batch vs Stream Processing](../06-async-and-events/03-batch-vs-stream-processing.md)

## Read more

- [Debezium architecture — official docs](https://debezium.io/documentation/reference/stable/architecture.html)
- [Reliable microservices data exchange with the outbox pattern — Debezium blog](https://debezium.io/blog/2019/02/19/reliable-microservices-data-exchange-with-the-outbox-pattern/)
- [Turning the database inside-out — Martin Kleppmann](https://martin.kleppmann.com/2015/03/04/turning-the-database-inside-out.html)

<!-- nav -->

---

[← Consistent Hashing](07-consistent-hashing.md) · [Index](../README.md) · [Geohashing →](09-geohashing.md)
