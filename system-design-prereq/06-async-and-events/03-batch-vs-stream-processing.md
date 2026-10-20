# Batch vs Stream Processing

> **TL;DR** — Batch processing runs over a bounded, stored dataset on a schedule (huge throughput, minutes-to-hours latency); stream processing handles an unbounded flow of events continuously as they arrive (milliseconds to seconds). Pick by how fresh the answer must be — and how much correctness complexity you can afford.

## How it works

```text
Batch:   sources ──(nightly)──► job: read all → transform → write ──► warehouse / report

Stream:  producers ──► log (Kafka partitions) ──► processor (keyed state + windows) ──► sink
                         ▲ replay from offset          │ periodic checkpoints
```

- **Batch:** a job reads a complete, fixed input (files, tables, a day of events), computes, writes output, exits. Failure → rerun the job.
- **Stream:** a long-running processor consumes events from a log, keeps state per key, emits results continuously. Failure → restore the last checkpoint and replay from its offsets.

## Batch vs stream

| | Batch | Stream |
|---|---|---|
| Input | bounded (files, tables, a day's data) | unbounded event log (Kafka, Kinesis, CDC) |
| Trigger | schedule or on demand | every event / tiny micro-batches |
| Latency | minutes–hours | ms–seconds |
| State | recomputed each run | long-lived, checkpointed |
| Correctness | easy: complete data, rerun on failure | hard: late and out-of-order events, exactly-once |
| Reprocessing | rerun the job | replay the log from an offset |
| Use cases | ETL/ELT, reports, billing, ML training, backfills | fraud, alerting, live dashboards, CDC sync, recommendations |
| Tools | Spark, Hadoop MapReduce, Hive, dbt/warehouse SQL | Flink, Kafka Streams, Spark Structured Streaming, Kinesis, Dataflow/Beam |

## Advanced

- **Event time vs processing time:** event time = when it happened (in the event); processing time = when the system sees it. Offline clients, retries and backlogs deliver events late and out of order → aggregate on event time.
- **Windows:** tumbling (fixed, non-overlapping: per minute), hopping/sliding (fixed size, overlapping: 5 min every 1 min), session (per key, closed by an inactivity gap).
- **Watermarks:** the processor's estimate that "no events older than T are still coming"; a window fires when the watermark passes its end. **Allowed lateness** keeps window state to update results for stragglers; anything later goes to a side output.
- **Triggers & accumulation:** emit early speculative results, then refined/final ones (the Beam model: *what* is computed, *where* in event time, *when* in processing time, *how* refinements relate).
- **State & checkpoints:** keyed state lives locally (memory or RocksDB) for speed; Flink snapshots it consistently via checkpoint barriers (Chandy–Lamport style) to durable storage, then restores state + source offsets on failure.
- **Exactly-once:** checkpointed state + replayable source + transactional (two-phase-commit) or idempotent sinks. Kafka Streams: `processing.guarantee=exactly_once_v2`. Effects outside the pipeline (emails, HTTP calls) still need idempotency.
- **Stream–table duality:** a table is the latest value per key of a changelog stream; a stream is a table's change log (Kafka compacted topics, KTables).
- **Joins:** stream–stream joins need a time window (buffer both sides); stream–table joins enrich each event with the current value (e.g. user profile).
- **Backpressure:** slow sinks must slow the sources (Flink credit-based flow control) or lag grows unbounded; watch consumer lag.
- **Lambda vs Kappa:** Lambda = batch layer (accurate, recomputed) + speed layer (approximate, real-time), merged at query time — the same logic written twice. Kappa = one streaming pipeline; reprocess by replaying the log (needs long retention or tiered storage).
- **Micro-batch vs record-at-a-time:** Spark Structured Streaming runs small batches (latency ~100 ms and up); Flink processes each record (lower latency). Both support exactly-once.
- **Skew & hot keys:** keying gives ordering and local state, but one hot key (a celebrity, a giant tenant) pins one task → salting + two-stage aggregation.
- **Converging models:** Beam/Dataflow and Flink treat batch as a bounded stream with the same API; lakehouse table formats (Delta Lake, Apache Iceberg, Apache Hudi) let streaming jobs write tables that batch jobs query.

## When to use which

- **Batch:** answers can be hours old (daily reports, invoices, ML training), you need complete data (month-end close), or heavy recomputation/backfills — cheapest and simplest to get right.
- **Stream:** value decays in seconds (fraud, alerting, live pricing/inventory), continuous sync (CDC → search index/cache), event-driven features.
- **Both:** stream for fresh, approximate numbers; batch to reconcile and correct them (typical for billing and analytics).

## In the wild

Apache Spark (batch + Structured Streaming) · Apache Flink (Alibaba, Uber, Netflix) · Kafka Streams / ksqlDB · Amazon Kinesis + Managed Service for Apache Flink · Google Dataflow (Apache Beam) · Azure Stream Analytics · dbt on Snowflake/BigQuery for batch ELT · Airflow/Dagster for scheduling · .NET: Hangfire / Quartz.NET for batch jobs, Kafka or RabbitMQ consumers for streaming.

## Gotchas

- Aggregating on processing time silently produces wrong numbers after any outage or backlog replay.
- A stream processor's "exactly-once" stops at its boundaries — external side effects still need idempotency.
- Streaming isn't automatically better: always-on clusters, state management and late-data handling cost far more than a cron job.
- Replay only works while the source log still has the data (retention) and the logic is deterministic.
- Hot keys and unbounded state (session windows that never close, join buffers) are the usual causes of streaming outages.

## Related

[Message Queues](01-message-queues.md) · [Event Sourcing](04-event-sourcing.md) · [CQRS](05-cqrs.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Idempotency](02-idempotency.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [Tyler Akidau — Streaming 101: The world beyond batch](https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/)
- [Apache Flink — Stateful stream processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Jay Kreps — Questioning the Lambda Architecture](https://www.oreilly.com/radar/questioning-the-lambda-architecture/)

<!-- nav -->

---

[← Idempotency](02-idempotency.md) · [Index](../README.md) · [Event Sourcing →](04-event-sourcing.md)
