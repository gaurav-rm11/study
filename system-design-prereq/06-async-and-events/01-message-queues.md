# Message Queues

> **TL;DR** — A message broker stores messages from producers until consumers process them, so the two sides don't have to be up, fast or scaled at the same time. It's how you run async work, absorb spikes, retry failures and fan events out to many services.

## How it works

```text
Producer ─► exchange/topic ─► queue [m3 m2 m1] ─┬─► Consumer A ─ ack ─► deleted
                                         │      └─► Consumer B ─ nack ─► redelivered
                                         └─ failed N times ─► DLQ ─► alert · fix · replay
```

- **Decoupling** in time (the consumer can be down; messages wait), space (the producer doesn't know who consumes) and scale (each side scales on its own). **Load levelling**: a burst lands in the queue and consumers drain it at a rate the database survives.
- **Competing consumers (point-to-point)** — N consumers share one queue; each message goes to one of them → work distribution. **Publish/subscribe** — each subscriber (its own queue, subscription or consumer group) gets a copy → event fan-out.
- **Ack / nack** — the consumer acks after processing → the broker deletes the message (Kafka: the consumer commits its offset). Nack, crash, closed connection or timeout → redelivery.
- **Prefetch** — cap on unacked messages per consumer (RabbitMQ `basic.qos`): low = fair dispatch of slow jobs, high = throughput, unlimited = one consumer hoards the backlog.
- **Dead-letter queue (DLQ)** — where a message goes after N failed attempts, rejection or expiry, instead of looping forever.

## Brokers compared

| | RabbitMQ | Kafka | SQS / SNS | Azure Service Bus |
|---|---|---|---|---|
| Model | queue broker (AMQP), routing via exchanges | partitioned, replicated append-only log | SQS queue + SNS pub/sub topic | queues + topics/subscriptions |
| Delivery | push, throttled by prefetch | pull (poll) | pull (long poll ≤ 20 s) | pull; SDK processor pumps messages |
| Ordering | per queue; single active consumer keeps it | per partition (key → partition) | Standard: best-effort; FIFO: per message group | per session (`SessionId`) |
| Retention / replay | deleted on ack; **streams** keep a replayable log | by time/size (default 7 days) or compaction; replay by resetting offsets | ≤ 14 days (default 4); no replay, only DLQ redrive | deleted on complete; no replay |
| Throughput | high; a queue lives on one node → shard hot queues | millions msg/s per cluster; parallelism = partitions | Standard ≈ unlimited; FIFO capped | Premium scales by messaging units |
| Routing | exchanges: direct (exact key), topic (`orders.*.eu`, `#`), fanout, headers | none: topic + key; consumers filter | SNS filter policies → SQS, Lambda, HTTP | SQL / correlation filters per subscription |
| Delay / priority | delayed-message plugin or TTL + DLX; priority queues | neither | delay ≤ 15 min; no priority | scheduled messages; no priority |
| Notable | publisher confirms, quorum queues | idempotent producer, transactions, Connect | FIFO dedupe id, visibility timeout | sessions, duplicate detection, transactions |

## Advanced

- **Delivery semantics:** *at-most-once* = ack before processing (auto-ack) → a crash loses the message. *At-least-once* = ack after → a crash between side effect and ack redelivers → consumers must be idempotent. *Exactly-once* exists only inside one system: Kafka's idempotent producer (producer id + per-partition sequence numbers) plus transactions commit consume → produce → offsets atomically for `read_committed` readers. An external effect (DB row, email, HTTP call) is back to at-least-once + dedupe.
- **Ordering** holds only per queue/partition with one consumer at a time — competing consumers, retries and redelivery reorder. Pin related messages to one lane: Kafka partition key (`orderId`), RabbitMQ single active consumer, SQS FIFO `MessageGroupId`, Service Bus sessions. Adding Kafka partitions remaps keys. Or tolerate disorder: carry a version, drop stale messages.
- **Poison messages:** one message that always fails hot-loops or blocks its lane. Ladder: a few immediate retries → delayed retries with exponential backoff (SQS visibility timeout, RabbitMQ TTL + dead-letter exchange, Kafka retry topics like `orders.retry.5m`) → DLQ after N attempts (SQS `maxReceiveCount`, Service Bus `MaxDeliveryCount` default 10, quorum-queue delivery limit) → alert on DLQ depth > 0 → fix → replay. Retry topics give up per-key order.
- **Visibility vs ack timeouts:** SQS hides a received message for the visibility timeout (default 30 s, max 12 h); not deleted in time → visible again → processed twice. Set it above p99 processing time and extend it (`ChangeMessageVisibility`) for long jobs. RabbitMQ redelivers when the channel closes, and closes channels holding an unacked delivery past `consumer_timeout` (default 30 min). Service Bus: renewable peek-lock. Kafka has no per-message ack: miss `max.poll.interval.ms` (default 5 min) → evicted → rebalance → reprocess from the last committed offset.
- **Backpressure & autoscaling:** scale consumers and alert on queue depth, **age of oldest message** (SQS `ApproximateAgeOfOldestMessage`) and Kafka **consumer lag** (log-end offset − committed offset), e.g. with KEDA. Bound queues (RabbitMQ `x-max-length` + `reject-publish`, TTLs); RabbitMQ blocks publishers on memory/disk alarms. Pull consumers set their own pace; push relies on prefetch. Kafka caps a group at one consumer per partition.
- **Kafka internals:** partitions are replicated (leader + in-sync replicas; `acks=all` + `min.insync.replicas` for durability). Each consumer-group member owns some partitions and commits offsets; a join, leave or crash triggers a **rebalance** that pauses consumption (cooperative-sticky assignment and static membership shrink it). **Log compaction** keeps the latest record per key → the topic becomes a changelog/table.
- **RabbitMQ internals:** **publisher confirms** ack a publish once it's persisted (quorum queue: on a majority) — without them a publish can vanish silently. **Quorum queues** replicate with Raft (classic mirrored queues are gone in 4.0). **Streams** add an append-only, replayable log.
- **Queue vs log:** a queue deletes on ack and tracks each message → per-message retry, DLQ, delay, priority; ideal for jobs. A log keeps messages and each reader tracks an offset → many independent readers, replay, CDC, reprocessing; but a stuck message blocks its partition.
- **Transactional outbox / inbox:** a DB write plus a broker publish isn't atomic (crash between → lost event, or an event for a rolled-back change). Outbox: insert the message into an `outbox` table in the same transaction as the business change; a relay (poller, or CDC such as Debezium) publishes it. Inbox: store each processed message id in the same transaction as the consumer's side effects; skip ids already seen.
- **Message design:** small (KBs), versioned schema with additive changes (Avro/Protobuf + schema registry), unique message id, correlation/causation ids for tracing, type and timestamp. Events are past-tense facts with many consumers (`OrderPlaced`); commands have one handler (`ChargeCard`). **Claim check**: put large payloads in S3/Blob Storage and send a pointer (the SQS Extended Client does this).
- **Fan-out:** SNS topic → one SQS queue per service, so each gets its own copy, pace, retries and DLQ. Same shape: RabbitMQ fanout/topic exchange → per-service queues; Service Bus topic → subscriptions; Kafka → one consumer group per service.
- **Request-reply:** the request carries `reply_to` + `correlation_id`; the responder publishes to that queue; the caller matches the id and times out (RabbitMQ direct reply-to, MassTransit request client).

## Trade-offs

- ✅ Services deploy, fail and scale independently; spikes are absorbed; failures become retries instead of user errors; one event reaches many consumers.
- ❌ Eventual consistency, duplicates and reordering become your problem; tracing a flow across hops needs correlation ids.
- ❌ Another stateful system to run, monitor and pay for; a growing backlog hides an outage until latency explodes.
- ❌ Wrong tool when the caller needs the result now (reads, validation). At low volume a jobs table polled with `FOR UPDATE SKIP LOCKED` (Postgres) or `READPAST` (SQL Server), Hangfire or BullMQ is simpler.

## In the wild

RabbitMQ · Apache Kafka (Confluent, Amazon MSK, Redpanda) · Amazon SQS + SNS, EventBridge · Azure Service Bus, Event Hubs (Kafka-compatible log) · Google Pub/Sub · NATS JetStream · Apache Pulsar · Redis Streams · .NET: MassTransit (v8 open source, v9 commercial), NServiceBus — retries, outbox/inbox and sagas over RabbitMQ / Service Bus / SQS · Node: amqplib, KafkaJS / Confluent's JS client, BullMQ (Redis) · SQL Server Service Broker.

## Gotchas

- **Auto-ack is at-most-once.** Ack after the side effect commits, never on receive.
- RabbitMQ `nack` with `requeue=true` hands a failing message straight back for immediate redelivery → hot loop; dead-letter it instead.
- Kafka consumers beyond the partition count sit idle — size partitions for peak parallelism up front; adding them later remaps keys.
- Publishing inside an open DB transaction lets consumers act on rows that roll back or aren't visible yet → outbox.
- RabbitMQ is happiest with short queues; a backlog of millions strains memory and recovery. Need retention or replay → use a log.
- TTL + DLX delays: RabbitMQ expires messages only from the head of a queue, so a 1-hour message blocks 1-minute ones behind it → one wait queue per delay.

## Related

[Idempotency](02-idempotency.md) · [Batch vs Stream Processing](03-batch-vs-stream-processing.md) · [Event Sourcing](04-event-sourcing.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [Webhooks](../02-apis-and-communication/05-webhooks.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [Consumer Acknowledgements and Publisher Confirms — RabbitMQ docs](https://www.rabbitmq.com/docs/confirms)
- [Apache Kafka documentation — Design (log, partitions, delivery semantics)](https://kafka.apache.org/documentation/#design)
- [Transactional outbox — microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)

<!-- nav -->

---

[← Bloom Filters](../05-caching/04-bloom-filters.md) · [Index](../README.md) · [Idempotency →](02-idempotency.md)
