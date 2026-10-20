# Scalability

> **TL;DR** — Scalability is a system's ability to absorb growing load — users, data, traffic — by adding resources while latency and cost per request stay acceptable. It's distinct from performance: a performance problem is slow for one user; a scalability problem is fast for one user but slow under load.

## How it works

- **Name the load parameter first** — what is growing: requests/s (QPS), read/write ratio, data volume, concurrent connections, or **fan-out** (downstream calls or writes per request). Each one saturates a different tier.
- **Latency** = time per request; **throughput** = requests per second. Past saturation, throughput flattens while queues form and latency climbs steeply.
- **Percentiles, not averages:** p50 is the typical request, p99/p99.9 the slowest 1%/0.1%. Fan-out amplifies tails: a page calling 100 backends that are each slow 1% of the time is slow on 1 − 0.99^100 ≈ 63% of loads.
- **Little's law:** in-flight requests = throughput × latency — 2,000 RPS × 50 ms = 100 concurrent. Size thread, connection and DB pools from it.

## Vertical vs horizontal

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | bigger machine: CPU, RAM, faster disk | more machines behind an LB, or data split across nodes |
| Limit | largest instance you can buy | effectively none, if the work partitions |
| Cost curve | superlinear at the high end | roughly linear, commodity nodes |
| SPOF | still one machine | redundancy built in (N+1) |
| Downtime | usually a restart to resize | add/remove nodes live |
| Complexity | none — no code change | statelessness, partitioning, partial failures, consistency |
| Fits | databases early on, stateful/legacy apps | stateless web/app tiers, workers, analytics |

## Scaling each tier

| Tier | Levers | Watch out for |
|---|---|---|
| App | stateless instances + LB + autoscaling | sessions, uploads or caches held in one node's memory |
| Reads | cache (in-process, Redis), CDN, read replicas | staleness; replica lag breaks read-your-writes |
| Writes | shard by key, async via queue, batching | queues smooth spikes but add no capacity; batching trades latency for throughput |
| Data volume | partition, archive/tier cold data, compress | cross-partition queries and reports |

## Advanced

- **Amdahl's law:** speedup on N nodes = 1 / ((1 − p) + p/N), p = parallelizable fraction. With 5% serial work the ceiling is 20×, however many nodes you add.
- **Universal Scalability Law (Gunther):** C(N) = N / (1 + α(N − 1) + βN(N − 1)). α = contention (queueing on a shared lock or resource), β = coherency (nodes exchanging state to stay consistent). With β > 0 throughput peaks at N ≈ √((1 − α)/β), then **falls** — adding nodes makes it slower.
- **AKF scale cube:** X = clone identical instances behind an LB; Y = split by function (services); Z = split by data (shard by customer/key). X is cheap; Y and Z cost real engineering.
- **Shared-nothing:** each node owns its CPU, memory and disk and talks to others only over the network, so nodes add capacity independently; shared-disk clusters (Oracle RAC) spend more on cross-node coordination.
- **Coordination is the enemy:** distributed locks, global counters/auto-increment, cross-shard transactions, global uniqueness checks and synchronous cross-region writes serialize work (USL's α and β). Make each operation touch one partition; generate IDs without coordination (UUIDv7 — `Guid.CreateVersion7()` in .NET 9 — or Snowflake-style); keep per-node counters and sum them asynchronously.
- **Hot keys / hot partitions:** sharding only helps if load spreads. A celebrity, a viral product or a monotonic key (timestamp, auto-increment) piles traffic onto one shard. Fixes: salt/split the key (`key#0..N`, scatter-gather on read), cache hot keys in-process, coalesce duplicate in-flight requests, hash-partition monotonic keys instead of range-partitioning them.
- **Fan-out-on-write vs on-read (feeds):** *push* copies each new post into every follower's timeline — cheap reads, but a celebrity with 50M followers means 50M writes per post. *Pull* merges followees' posts at read time — cheap writes, expensive reads. **Hybrid** (Twitter): push for normal accounts, pull celebrity posts at read time and merge.
- **Autoscaling:** *reactive* target tracking on CPU, RPS per instance, or queue depth/consumer lag; *predictive or scheduled* for known daily or event peaks. Cooldowns/stabilization windows prevent flapping. New instances need minutes to boot, JIT, warm caches and open pools — scale-out lags the spike, so keep headroom or a **warm pool** of pre-initialized instances.
- **Back-pressure and load shedding:** bound every queue; when full, push back (block the producer, or return 429/503 with `Retry-After`) instead of buffering without limit. Shed excess or low-priority work at the front door before spending resources on it, and drop requests whose client deadline has already passed — past saturation, timeouts plus retries collapse goodput.
- **Cell-based architecture:** N identical, independent copies of the whole stack (app + data), each serving a subset of customers, behind a thin router mapping customer → cell. Scale by adding cells; cell size is capped at a load-tested maximum, so you never run at untested scale, and a failure stays in one cell.
- **Back-of-the-envelope:** QPS = DAU × actions/day ÷ 86,400 (≈ 10⁵ s, so 1M/day ≈ 12 QPS) × peak factor (often 2–3×). Storage = writes/day × record size × retention × replicas. Bandwidth = QPS × payload size.

```text
10M DAU · 20 reads + 2 writes per user/day · 1 KB per write · 10 KB per response
reads    200M/day ÷ 86,400 ≈ 2,300 QPS avg → ~7,000 at 3× peak
writes    20M/day ÷ 86,400 ≈   230 QPS avg → ~700 at 3× peak
storage  20M × 1 KB = 20 GB/day ≈ 7.3 TB/yr → ~22 TB with 3 replicas
egress   7,000 QPS × 10 KB ≈ 70 MB/s ≈ 560 Mbit/s at peak
```

## Trade-offs

- ✅ Horizontal: near-linear capacity, built-in redundancy, scale in to cut cost.
- ❌ Horizontal: the distributed-systems tax — partial failures, network hops, consistency questions, more to operate.
- ✅ Vertical: no code change, strong consistency for free — scale the database up before sharding it.
- ❌ Vertical: hard ceiling, superlinear cost, still a SPOF, downtime to resize.
- ❌ Every cache, replica and queue buys scale with staleness or eventual consistency.

## In the wild

AWS Auto Scaling (target tracking, predictive scaling, warm pools), Azure VM Scale Sets, Kubernetes HPA + Cluster Autoscaler, KEDA (scales consumers on RabbitMQ queue length) · Postgres/MySQL read replicas, SQL Server Always On readable secondaries · Vitess (MySQL), Citus (Postgres), DynamoDB, Cassandra, MongoDB for sharding · Redis, Memcached, CloudFront/Cloudflare/Akamai · Node runs JS on one thread per process — use every core via the `cluster` module, PM2 or one container per core · ASP.NET Core rate-limiting middleware for shedding · Twitter's hybrid timeline fan-out · AWS and Slack cell-based architectures.

## Gotchas

- Estimate before sharding: one well-indexed Postgres/SQL Server primary plus read replicas covers many interview-scale workloads.
- More app instances can sink the database: 50 pods × a pool of 20 = 1,000 connections vs Postgres's default `max_connections` of 100 (each one a server process). Put PgBouncer/RDS Proxy in front and size pools with Little's law.
- Queues absorb bursts, not sustained overload — if producers outpace consumers on average, the backlog grows without bound.
- CPU-based autoscaling misses I/O-bound and event-loop services (Node); scale on latency, RPS or queue depth.
- Keep the app tier stateless from day one (cheap); add Y/Z splits (services, shards) only when measured load demands it — they're expensive to undo.

## Related

[Stateful vs Stateless](04-stateful-vs-stateless.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Caching](../05-caching/01-caching.md) · [Replication](../04-databases/04-replication.md) · [Database Sharding](../04-databases/06-database-sharding.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md)

## Read more

- [The Tail at Scale — Dean & Barroso (Google)](https://research.google/pubs/the-tail-at-scale/)
- [Using load shedding to avoid overload — AWS Builders' Library](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/)
- [The AKF Scale Cube (AKF Partners)](https://akfpartners.com/growth-blog/scale-cube)

<!-- nav -->

---

[Index](../README.md) · [Availability →](02-availability.md)
