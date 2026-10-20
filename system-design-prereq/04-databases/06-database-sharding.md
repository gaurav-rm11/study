# Database Sharding

> **TL;DR** — Sharding splits one logical database's rows across many independent database servers (shards) by a **shard key**, so storage and write throughput scale past what a single primary can handle. It's also the costliest way to scale a database, so it comes last.

## How it works

```text
                 bucket = hash(customer_id) % 1024
App ──► Router ──┬──► Shard 1   buckets 0–341     ──► replicas
                 ├──► Shard 2   buckets 342–682   ──► replicas
                 └──► Shard 3   buckets 683–1023  ──► replicas
```

- **Horizontal partitioning:** every shard has the same schema and a disjoint subset of rows; each is a full database with its own CPU, RAM, disk and write capacity.
- **vs vertical partitioning:** splitting by tables or columns (blobs or `audit_log` to another DB, one DB per service) scales by function, but one huge table still lives on one box.
- **vs replication:** replicas copy the *same* data (read scaling, failover); shards hold *different* data (write and storage scaling). Production combines them: each shard is a primary plus replicas.
- **Routing:** something must map key → shard — the **app/driver** (a shard-map library in every service; no extra hop), a **proxy** (Vitess VTGate, MongoDB `mongos`, ProxySQL query rules, the Citus coordinator; the app sees one database), or the **database itself** (Cassandra, DynamoDB, CockroachDB: any node routes).

## Strategies

| Strategy | Shard chosen by | Pros | Cons |
|---|---|---|---|
| Range | key range (`id` 1–10M, names A–F, month) | range scans hit one shard; easy to split a hot range | monotonic keys (timestamps, auto-increment) send every insert to the last shard |
| Hash | `hash(key)` → bucket → shard | even spread of data and load | range queries scatter to every shard; naive `mod N` makes resharding painful |
| Directory / lookup | a lookup table `key → shard` | place or move any key or tenant individually | the lookup sits on every request path: cache it, make it HA |
| Geo / entity / tenant | region, `tenant_id`, root entity | data residency (GDPR), locality, tenant isolation | uneven shards (one giant tenant or region) |

## Advanced

- **Choosing the shard key:** high cardinality, even spread of data *and* traffic, present in the dominant queries so most hit one shard, and immutable (changing it means moving the row). `customer_id` or `tenant_id` usually beats `created_at`, `status` or `country`.
- **Hot keys / celebrities:** one key can outgrow a shard (a DynamoDB partition tops out at 3,000 RCU / 1,000 WCU). Salt it (`key#0…key#9`: writes spread, reads fan in), split the hot range, give the whale a dedicated shard, cache its reads.
- **Cross-shard queries:** scatter-gather to every shard, then merge/sort/limit at the router. Latency = slowest shard, every shard pays for every query, and a paged `ORDER BY … LIMIT` makes each shard return offset + limit rows.
- **Joins:** co-locate related tables on the same key (`orders` and `order_items` both by `customer_id`; Citus co-location, Vitess), copy small reference tables to every shard, or denormalise.
- **Transactions:** single-shard is ordinary local ACID. Cross-shard needs 2PC (blocking, slow) or sagas (eventual, compensations); Spanner and CockroachDB do distributed transactions natively at a latency cost. Model so each invariant lives inside one shard.
- **Secondary indexes:** *local* (each shard indexes its own rows; a lookup by a non-shard-key column scatters to all shards) vs *global* (index partitioned by the indexed value; one-shard lookup, but every write updates a remote index, usually asynchronously — DynamoDB GSIs are eventually consistent).
- **Resharding:** with `hash mod N`, going to N+1 moves almost every row. Use consistent hashing, or pre-split into many fixed **logical shards** (the 1,024 buckets above) mapped to fewer physical hosts — scaling out then moves whole buckets and updates the map (Notion: 480 logical shards on 32 Postgres hosts). Online move: snapshot/backfill → stream changes (CDC or dual writes) → verify (checksums, shadow reads) → brief write freeze and routing cutover → retire the old copy. Vitess automates it (`Reshard`, `MoveTables`).
- **Globally unique IDs:** per-shard auto-increment collides. Use Snowflake-style 64-bit IDs (41-bit ms timestamp + 10-bit machine + 12-bit sequence; roughly time-sorted), UUIDv7 (time-ordered, B-tree-friendly unlike v4; .NET 9 `Guid.CreateVersion7()`, Postgres 18 `uuidv7()`), per-shard ranges or offsets (MySQL `auto_increment_increment` / `auto_increment_offset`), or embed the shard id so routing needs no lookup (Instagram: 41-bit time + 13-bit logical shard + 10-bit sequence).
- **Operations × N:** backups (a consistent cross-shard snapshot needs coordination), schema migrations rolled shard by shard (code must handle both versions), rebalancing, connection pools (app instances × shards), per-shard dashboards — averages hide one hot shard.
- **Multi-tenant:** a shard (or database) per tenant gives isolation, per-tenant restore and residency, but thousands of databases to migrate; shared shards keyed by `tenant_id` are dense and cheap but need `tenant_id` in every query and per-tenant limits for noisy neighbours. Common hybrid: pool small tenants, dedicate shards to whales.
- **Shard last:** first try query and index tuning, caching, read replicas, a bigger box, archiving cold data, and single-node **table partitioning** (Postgres declarative partitioning, SQL Server partitioned tables, MySQL partitioning) — partition pruning and cheap drop/detach of old partitions with no distributed complexity.
- **Built-in sharding:** MongoDB (range or hashed shard keys, chunk balancer, online `reshardCollection` since 5.0), Cassandra, DynamoDB and Cosmos DB (hashed partition keys), Elasticsearch (primary shard count fixed at index creation), Vitess (MySQL), Citus (Postgres), Spanner, CockroachDB and TiDB (ranges auto-split by size and load and rebalance themselves).

## Trade-offs

- ✅ Near-linear scaling of writes, storage and working set; smaller per-shard indexes, backups and restores; an outage hits one shard's users, not everyone.
- ❌ Cross-shard queries, joins, transactions and unique constraints become hard or impossible; every operational task multiplies by N.
- ❌ The shard key is close to irreversible — a bad one means hotspots or scatter-gather forever, and resharding is a project.

## In the wild

Vitess (born at YouTube; Slack, PlanetScale) · Citus (Postgres extension) · MongoDB sharded clusters (`mongos` + config servers) · Cassandra / DynamoDB / Cosmos DB partition keys · Elasticsearch / OpenSearch shards · Spanner, CockroachDB, TiDB · Instagram (thousands of logical shards as Postgres schemas) · Notion (app-level) and Figma (DBProxy) sharded Postgres · MySQL: Vitess, ProxySQL · .NET / Azure SQL: Elastic Database client library (shard map manager, data-dependent routing).

## Gotchas

- **Replication ≠ sharding** — a classic mix-up: replicas scale reads and add failover; only sharding scales writes.
- **Monotonic keys on range sharding** (`created_at`, auto-increment) → one hot shard takes every insert; hash the key or prefix it with a bucket.
- **Unique constraints hold only per shard** — global uniqueness (email, username) needs a separate table sharded by that value, claimed first.
- **Fan-out tail latency:** if each shard is slow 1% of the time, a query touching 100 shards is slow 63% of the time (1 − 0.99¹⁰⁰).
- **UUIDv7 in SQL Server:** `uniqueidentifier` sorts by its last bytes first, so v7's timestamp prefix doesn't give sequential inserts there.
- **`mod N` over physical hosts** bakes today's host count into data placement; start with many logical shards, even on one host.

## Related

[Replication](04-replication.md) · [Consistent Hashing](07-consistent-hashing.md) · [Change Data Capture](08-cdc.md) · [Database Indexes](03-database-indexes.md) · [Two-Phase Commit](../07-distributed-coordination/03-two-phase-commit.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [Sharding pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Herding elephants: lessons learned from sharding Postgres at Notion](https://www.notion.com/blog/sharding-postgres-at-notion)
- [How Figma's databases team lived to tell the scale — Figma](https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/)

<!-- nav -->

---

[← CAP Theorem](05-cap-theorem.md) · [Index](../README.md) · [Consistent Hashing →](07-consistent-hashing.md)
