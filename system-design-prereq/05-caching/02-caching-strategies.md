# Caching Strategies

> **TL;DR** — A caching strategy fixes who loads the cache, when writes reach it, and in what order the cache and the DB are touched. That choice sets the staleness window, the write latency and what breaks when either side fails; cache-aside + TTL + delete-on-write is the default.

## How it works

```text
Read    app ──GET k──► cache ──hit──► return
                         │ miss
                         ▼
        app ──SELECT──► DB ──► app ──SET k, TTL──► cache ──► return
Write   app ──UPDATE──► DB (commit) ──► app ──DEL k──► cache
        the next read misses and reloads the committed row
```

- The diagram is **cache-aside (lazy loading)**: the app owns both paths, the cache is a dumb key-value store holding only what was read, and a cache outage means slower, not down.
- Strategies differ on three axes: who loads on a miss (app or cache provider), when writes reach the cache (synchronously, asynchronously, never) and the order of DB and cache operations.

## Patterns

| Pattern | Mechanics | Consistency | Latency | Failure risk | Use when |
|---|---|---|---|---|---|
| **Cache-aside** (lazy loading) | app checks cache, on miss reads DB and fills; on write updates DB, deletes key | stale up to TTL or a race window | a miss costs 3 round trips | cache down → DB takes every read | default for read-heavy data |
| **Read-through** | the cache library calls a loader on miss | as cache-aside | as cache-aside | loader ties the cache to the DB | centralise loading logic |
| **Write-through** | each write updates cache and DB synchronously before the ack | reads after writes are fresh | slower writes (two stores) | write fails if either store fails; caches never-read data | read-after-write freshness matters |
| **Write-behind** (write-back) | write to cache, ack, flush to DB asynchronously in batches | DB lags the cache | fastest writes | cache crash before flush loses acked writes | write-heavy, loss-tolerant (counters) |
| **Write-around** | writes go only to the DB; reads are cache-aside | an old copy lives until TTL unless deleted | first read after a write misses | low | written once, rarely read (logs, imports) |
| **Refresh-ahead** | reload hot keys shortly before they expire | hot keys stay fresh | hot keys never miss | wasted loads if predictions are wrong | predictable hot keys, slow loads |

## Advanced

- **Delete, don't update, on write:** two writers can commit A then B but `SET` the cache B then A, leaving A cached. A delete is idempotent and order-insensitive, and skips computing values nobody reads.
- **DB first, then delete:** deleting first lets a concurrent reader refill the old value before your commit lands.
- **The cache-aside race survives even that:** reader misses and reads v1 → writer commits v2 and deletes → reader `SET`s v1, stale until TTL. Rare against the primary; common when refills read a lagging replica. Fixes:
  - **short TTL** — the safety net that bounds the damage;
  - **delayed double delete** — delete again after replica lag + one read's duration (via a delayed job);
  - **versioned values / compare-and-set** — store the row version (`rowversion`, `updated_at`) and write only if newer (Redis Lua script, Memcached `cas`);
  - **leases** (Facebook memcache) — a miss hands out a lease token and a delete voids it, so the late stale `SET` is rejected; handing out one token per key every few seconds also stops stampedes;
  - **CDC-driven invalidation** — tail the binlog/WAL (Debezium; Facebook's mcsqueal) and delete keys after commit, in commit order.
- **Dual-write failure:** the DB commits but the cache `DEL` fails (network blip, crash) → stale until TTL. Retry the delete through a queue, write an "invalidate k" row in the same transaction for a relay to execute (outbox), or use CDC so there's no second write. Never fill the cache inside a DB transaction — a rollback leaves cached data that never existed.
- **Write-behind risks:** a cache crash before flush loses acknowledged writes (Redis AOF `appendfsync everysec` still risks ~1 s); safer is appending to Kafka/RabbitMQ and letting consumers apply to the DB. Flushes must keep per-key order (partition by key). **Coalescing** is the payoff — 1,000 `INCR`s become one `UPDATE`. DB constraint errors surface after the client got "OK". Keep a replay log for recovery.
- **Read-/write-through providers:** DynamoDB Accelerator (DAX), NCache (.NET; read-through, write-through and write-behind providers), Hibernate second-level cache, Caffeine `LoadingCache` (`refreshAfterWrite` = refresh-ahead), Hazelcast `MapStore` (write delay > 0 = write-behind).
- **Mix per entity:** counters, likes, views → write-behind; logs, audit trails, bulk imports → write-around; product catalogue → cache-aside + TTL + delete on write; config and feature flags → refresh-ahead into L1.
- **TTL:** derive it from how stale each data type may be; add **jitter** (±10–20%) so bulk-loaded keys don't expire together; keep it even with explicit invalidation; cap sliding expiration with an absolute one or a hot key never refreshes.
- **Negative caching:** cache a "not found" sentinel with a short TTL to stop penetration; delete it when the item gets created.
- **Key design:** `{namespace}:{schema-version}:{tenant}:{entity}:{id}`, e.g. `catalog:v3:t42:product:1001`. Bump the version when the cached shape changes so old and new code never read each other's values; hash normalised query parameters instead of raw strings; Redis Cluster hash tags (`{user:42}:cart`) keep related keys in one slot for multi-key ops.
- **L1/L2 coherence:** each instance keeps a short-TTL in-memory L1 over Redis L2; a write deletes L2 and publishes the key on a pub/sub channel so every instance evicts its L1. Pub/sub drops messages during disconnects — the L1 TTL bounds that.
- **Don't cache decisions:** balances, inventory decrements, seat holds, uniqueness checks — anything that needs strong or read-your-writes consistency. Cache for display; enforce in the DB with transactions, constraints or conditional updates (`UPDATE … WHERE stock >= @qty`).
- **.NET / Node:** `IMemoryCache` (in-process; `GetOrCreateAsync` isn't stampede-safe — concurrent misses all run the factory), `IDistributedCache` (byte[] API over Redis, SQL Server, NCache), `HybridCache` (L1 + L2, stampede protection, tag invalidation), FusionCache (adds a Redis backplane for L1 invalidation) · Node: ioredis or node-redis, `lru-cache` for L1, single-flight by sharing one pending Promise per key.

## Trade-offs

- ✅ Lazy patterns (cache-aside, read-through) cache only what's read and degrade gracefully when the cache fails.
- ❌ Lazy patterns pay a miss on first read and after every invalidation, and need stampede protection on hot keys.
- ✅ Eager patterns (write-through, refresh-ahead) keep hot data fresh and miss-free.
- ❌ Eager patterns spend writes and memory on data nobody reads, and tie the write path to cache availability.
- ❌ Write-behind makes the cache a temporary system of record, so it now needs durability, ordering and replay.

## In the wild

Facebook memcache (cache-aside with leases + mcsqueal invalidation from the MySQL binlog) · AWS DAX (read-/write-through for DynamoDB) · Hazelcast MapStore, NCache, Ehcache, Hibernate L2 · Caffeine, Spring Cache abstraction · Debezium → Kafka → cache invalidators · .NET: `IMemoryCache`, `IDistributedCache`, `HybridCache`, FusionCache · Node: ioredis, node-redis, `lru-cache`, cache-manager.

## Gotchas

- "Update the cache on write" sounds fresher but is the race-prone option — delete instead.
- Refilling from a read replica right after a write caches the pre-write row; refill from the primary, or double-delete after the lag.
- Changing the key format or cached shape in a deploy drops the hit ratio to ~0 → stampede. Version keys and pre-warm.
- Write-through without TTLs fills memory with never-read data and evicts the hot set.
- "Write to the cache and the DB in one transaction" is a classic wrong interview answer — there's no atomic commit across them; design for the delete failing.

## Related

[Caching](01-caching.md) · [Cache Eviction Policies](03-cache-eviction-policies.md) · [Bloom Filters](04-bloom-filters.md) · [CDC](../04-databases/08-cdc.md) · [Replication](../04-databases/04-replication.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [CQRS](../06-async-and-events/05-cqrs.md)

## Read more

- [Cache-Aside pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [Scaling Memcache at Facebook (Nishtala et al., NSDI 2013)](https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala)
- [HybridCache library in ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid)

<!-- nav -->

---

[← Caching](01-caching.md) · [Index](../README.md) · [Cache Eviction Policies →](03-cache-eviction-policies.md)
