# System Design Prereq

40 concepts to know before designing (or interviewing on) distributed systems — one short, dense note each, from basics to advanced.

Read top to bottom; each section builds on the previous one. Every note follows the same layout: **TL;DR → How it works → Advanced → Trade-offs → In the wild → Gotchas → Related → Read more**, with previous/next links at the bottom.

## 1. Foundations

| # | Topic | In one line |
|---|---|---|
| 1 | [Scalability](01-foundations/01-scalability.md) | Grow capacity with load: scale up vs out, and what breaks first |
| 2 | [Availability](01-foundations/02-availability.md) | Nines, SLOs and the math of redundancy, failover and MTTR |
| 3 | [Single Point of Failure (SPOF)](01-foundations/03-spof.md) | Find and remove components whose failure takes everything down |
| 4 | [Stateful vs Stateless](01-foundations/04-stateful-vs-stateless.md) | Where state lives decides how easily you scale and recover |
| 5 | [Concurrency vs Parallelism](01-foundations/05-concurrency-vs-parallelism.md) | Juggling many tasks vs running them simultaneously on many cores |

## 2. APIs & Communication

| # | Topic | In one line |
|---|---|---|
| 6 | [APIs](02-apis-and-communication/01-apis.md) | Service contracts: styles, HTTP semantics, versioning, pagination, errors |
| 7 | [REST vs GraphQL](02-apis-and-communication/02-rest-vs-graphql.md) | Resource endpoints vs client-shaped queries, and when each wins |
| 8 | [API Gateways](02-apis-and-communication/03-api-gateways.md) | One front door for auth, routing, rate limits and aggregation |
| 9 | [JSON Web Tokens (JWTs)](02-apis-and-communication/04-jwts.md) | Signed self-contained tokens: verification, key rotation, revocation, pitfalls |
| 10 | [Webhooks](02-apis-and-communication/05-webhooks.md) | Server-to-server event callbacks: signatures, retries, idempotent receivers |
| 11 | [Long Polling vs WebSockets](02-apis-and-communication/06-long-polling-vs-websockets.md) | Pushing server updates: polling, long polling, SSE and WebSockets |

## 3. Traffic & Edge

| # | Topic | In one line |
|---|---|---|
| 12 | [Proxy vs Reverse Proxy](03-traffic-and-edge/01-proxy-vs-reverse-proxy.md) | Acting for clients vs acting for servers, and why it matters |
| 13 | [Load Balancing](03-traffic-and-edge/02-load-balancing.md) | Spread traffic over healthy servers: L4/L7, algorithms, health checks |
| 14 | [Content Delivery Network (CDN)](03-traffic-and-edge/03-cdn.md) | Edge caches near users: cache headers, invalidation, origin shielding |
| 15 | [Rate Limiting](03-traffic-and-edge/04-rate-limiting.md) | Cap request rates: token bucket, sliding windows, distributed limiters |

## 4. Databases

| # | Topic | In one line |
|---|---|---|
| 16 | [SQL vs NoSQL](04-databases/01-sql-vs-nosql.md) | Relational vs key-value, document, wide-column and graph stores |
| 17 | [ACID Transactions](04-databases/02-acid-transactions.md) | Atomicity, consistency, isolation levels, durability and their anomalies |
| 18 | [Database Indexes](04-databases/03-database-indexes.md) | B+trees, composite and covering indexes, and why queries ignore them |
| 19 | [Replication](04-databases/04-replication.md) | Leader-follower, multi-leader and leaderless copies; lag and failover |
| 20 | [CAP Theorem](04-databases/05-cap-theorem.md) | During a partition pick consistency or availability; PACELC beyond |
| 21 | [Database Sharding](04-databases/06-database-sharding.md) | Split data across nodes by shard key, and live with the consequences |
| 22 | [Consistent Hashing](04-databases/07-consistent-hashing.md) | A hash ring that moves only ~1/N of keys when nodes change |
| 23 | [Change Data Capture (CDC)](04-databases/08-cdc.md) | Stream every committed row change from the DB log to other systems |
| 24 | [Geohashing](04-databases/09-geohashing.md) | Encode lat/long as prefix-searchable strings for proximity search |

## 5. Caching

| # | Topic | In one line |
|---|---|---|
| 25 | [Caching](05-caching/01-caching.md) | Faster copies of hot data: layers, hit ratio, stampedes, penetration |
| 26 | [Caching Strategies](05-caching/02-caching-strategies.md) | Cache-aside, read/write-through, write-behind, and keeping caches consistent |
| 27 | [Cache Eviction Policies](05-caching/03-cache-eviction-policies.md) | LRU, LFU, ARC, W-TinyLFU: what to drop when the cache is full |
| 28 | [Bloom Filters](05-caching/04-bloom-filters.md) | Tiny probabilistic set: definitely not or maybe, never a false negative |

## 6. Async & Events

| # | Topic | In one line |
|---|---|---|
| 29 | [Message Queues](06-async-and-events/01-message-queues.md) | Decouple producers and consumers: delivery semantics, ordering, DLQs |
| 30 | [Idempotency](06-async-and-events/02-idempotency.md) | Make retries safe: idempotency keys, dedupe, conditional writes |
| 31 | [Batch vs Stream Processing](06-async-and-events/03-batch-vs-stream-processing.md) | Scheduled bounded jobs vs continuous processing with windows and state |
| 32 | [Event Sourcing](06-async-and-events/04-event-sourcing.md) | Store every change as an immutable event; state is a replay |
| 33 | [Command Query Responsibility Segregation (CQRS)](06-async-and-events/05-cqrs.md) | Separate write model from read models, synced asynchronously |

## 7. Distributed Coordination

| # | Topic | In one line |
|---|---|---|
| 34 | [Distributed Locks](07-distributed-coordination/01-distributed-locks.md) | Mutual exclusion across machines: leases, Redlock debate, fencing tokens |
| 35 | [Consensus Algorithms](07-distributed-coordination/02-consensus-algorithms.md) | Raft and Paxos: majority agreement for leaders and replicated logs |
| 36 | [Two-Phase Commit (2PC)](07-distributed-coordination/03-two-phase-commit.md) | Atomic commit across participants, and why it blocks |
| 37 | [Saga Pattern](07-distributed-coordination/04-saga-pattern.md) | Local transactions plus compensations instead of distributed ACID |

## 8. Microservices & Reliability

| # | Topic | In one line |
|---|---|---|
| 38 | [Circuit Breaker](08-microservices-and-reliability/01-circuit-breaker.md) | Fail fast on a sick dependency to stop cascading failures |
| 39 | [Service Mesh](08-microservices-and-reliability/02-service-mesh.md) | Sidecar proxies for mTLS, retries and telemetry between services |
| 40 | [Observability](08-microservices-and-reliability/03-observability.md) | Logs, metrics and traces: understand production from its outputs |

---

[← Back to all study notes](../README.md)
