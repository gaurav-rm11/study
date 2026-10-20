# Caching

> **TL;DR** — A cache keeps copies of expensive-to-fetch or expensive-to-compute data in faster storage closer to the consumer, so repeat reads skip the slow path. You trade memory and bounded staleness for lower latency and far less load on the source of truth.

## How it works

```text
read k ──► cache ──hit──► return
             │ miss
             ▼
           origin (DB / API / computation) ──► store k with TTL ──► return
```

- **Locality makes it pay:** access is skewed (often Zipf-like) and repeats over time, so a small hot subset absorbs most reads.
- **Latency ladder** (orders of magnitude, not benchmarks):

| Data comes from | Approx. latency |
|---|---|
| In-process memory (hash map lookup) | ~100 ns |
| Local NVMe SSD random read | ~10–100 µs |
| Redis / Memcached in the same data centre | ~0.2–1 ms, mostly network round trip |
| Indexed DB query over the network | ~1–10 ms; heavy joins/aggregates 100 ms+ |
| Cross-region round trip | ~50–200 ms |

- **Hit ratio rules:** origin load is proportional to the *miss* ratio, so going from 90% to 99% hits cuts DB reads 10×.

```text
h = hits / (hits + misses)
effective latency ≈ t_cache + (1 − h) × t_origin
t_cache = 0.5 ms, t_origin = 20 ms:   h = 0.90 → 2.5 ms    h = 0.99 → 0.7 ms
```

## Layers and what to cache

| Layer | Examples | Notes |
|---|---|---|
| Browser / HTTP | `Cache-Control`, `ETag` → `304 Not Modified` | per user; can't be purged, only expire — fingerprint asset URLs |
| CDN | Cloudflare, CloudFront, Akamai, Fastly | shared edge copies of static assets and public responses |
| Reverse proxy | Nginx `proxy_cache`, Varnish | whole HTTP responses in front of the app |
| In-process (L1) | .NET `IMemoryCache`, Node `lru-cache`, Caffeine | fastest, no serialization; per instance, lost on restart |
| Distributed (L2) | Redis / Valkey, Memcached | shared by all instances; one network hop |
| Database | InnoDB buffer pool, Postgres `shared_buffers` + OS page cache, SQL Server buffer pool | automatic; sizing it is often the cheapest win |

- **Cache data that is** read-heavy, expensive to produce (joins, aggregates, third-party calls, rendering), tolerant of some staleness, and concentrated in a hot subset.
- **Skip data that is** written more than read, rarely re-read (mostly unique keys), or must be exactly current.

## Redis vs Memcached

| | Redis / Valkey | Memcached |
|---|---|---|
| Data model | strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog | opaque byte values |
| Threads | one thread executes commands (optional I/O threads) | multi-threaded |
| Durability / HA | RDB/AOF persistence, replicas, Sentinel, Cluster | none — a restart means empty; clients shard keys |
| Extras | Lua, pub/sub, transactions, server-side ops on structures | `cas`, `incr`, slab allocator |
| Pick when | you need structures (leaderboards, rate limits, locks), HA or pub/sub | a big, plain, multi-core key-value cache |

## Advanced

- **Invalidation:** TTL (bounded staleness and the safety net), delete on write, event/CDC-driven (tail the DB log, evict after commit), versioned keys (`product:42:v7` — bump the version, old keys age out). Races and ordering → [Caching Strategies](02-caching-strategies.md). Phil Karlton: "There are only two hard things in Computer Science: cache invalidation and naming things."
- **Stampede (thundering herd, dogpile):** a hot key expires and thousands of concurrent misses all recompute it, flattening the DB. Fixes:
  - **single-flight / request coalescing** — one load per key per process; other callers await the same `Task`/Promise;
  - **lock or lease** across the fleet (`SET lock:k <token> NX PX 5000`) — the winner recomputes, others serve stale or briefly retry;
  - **probabilistic early expiration** (XFetch) — reads refresh early with rising probability near expiry: recompute if `now − Δ·β·ln(rand()) ≥ expiry` (Δ = recompute time, β ≈ 1);
  - **stale-while-revalidate** — serve the old value while one background refresh runs (also an HTTP `Cache-Control` directive).
- **Penetration:** requests for keys that don't exist (bugs, scrapers enumerating IDs) always miss and hit the DB. Fix with **negative caching** ("not found" with a short TTL) and a **Bloom filter** of existing keys ("definitely absent" → skip the DB).
- **Avalanche:** many keys expire at once (bulk-loaded with one TTL) or the cache tier dies, and the DB takes 100% of reads. Fix with **TTL jitter** (±10–20%), **warm-up** before taking traffic, **multi-level caches** (L1 keeps serving through an L2 outage) and **DB protection**: concurrency limits, circuit breakers, load shedding.
- **Hot keys:** one key (celebrity profile, flash-sale item) saturates the single Redis shard that owns it. Add a **local L1** with a seconds-long TTL, or **split/replicate** it (`k#1…k#N`; read a random copy, write all) and read from replicas. `redis-cli --hotkeys` finds them (needs an LFU eviction policy).
- **Big values:** multi-MB values block Redis's command thread and saturate the NIC, spiking latency for every key on that shard. Cache projections instead of whole aggregates, compress or chunk; remove big keys with `UNLINK` (async), not `DEL`; find them with `redis-cli --bigkeys`.
- **Cold starts:** a new cluster, a flush or a key-format change starts at 0% hits; restarting all instances at once empties every L1. Pre-warm the top-N keys from access logs, shift traffic gradually, roll deploys.
- **Local vs distributed:** in-process L1 is thousands of times faster than a network hop and skips serialization, but duplicates memory per instance and drifts between instances; distributed L2 is shared and survives deploys. **Two-level L1 + L2:** short-TTL L1 over Redis; on write, delete from L2 and publish the key on Redis pub/sub so every instance evicts its L1. Pub/sub is fire-and-forget, so keep L1 TTLs short; Redis 6+ client-side caching (`CLIENT TRACKING`) has the server push invalidations instead.
- **Sizing:** fit the hot working set, not the dataset — the hit-ratio curve flattens as memory grows. Rising evictions with a falling hit ratio = undersized. Leave Redis headroom for fragmentation, replication buffers and fork copy-on-write during snapshots and AOF rewrites.
- **Monitoring:** hit ratio per key prefix or endpoint (Redis `keyspace_hits` / `keyspace_misses`), `evicted_keys`, memory and fragmentation, p99 latency, origin QPS. A hit-ratio drop right after a deploy usually means changed keys.
- **Serialization:** (de)serializing big JSON blobs can cost more than the network hop — use MessagePack or protobuf, compress large values, cache only the fields you read.
- **Security:** never cache per-user data under a shared key — a CDN caching `GET /me` by URL serves A's profile to B. Put user/tenant in the key, mark per-user responses `Cache-Control: private`, and don't let unkeyed headers change a cached response (cache poisoning).
- **Staleness is a per-type decision:** catalogue text tolerates minutes, prices seconds (or invalidate on write), feature flags seconds; balances and stock at checkout read the source of truth.

## Trade-offs

- ✅ Orders-of-magnitude faster reads; offloads the DB far more cheaply than scaling it; absorbs read spikes; can serve stale data while the origin is down.
- ❌ Staleness, invalidation bugs, and consistency that now spans two systems.
- ❌ Another tier to size, secure, monitor and pay for, with its own failure modes (stampede, avalanche, hot keys).
- ❌ It becomes load-bearing: once the DB is sized for a 95% hit ratio, a cold or failed cache is an outage — capacity-plan the origin for degraded hit ratios.

## In the wild

Redis / Valkey (Linux Foundation fork), Memcached · AWS ElastiCache, Azure Cache for Redis, GCP Memorystore · Facebook's memcache fleet, Twitter's Twemcache / Pelikan · .NET: `IMemoryCache`, `IDistributedCache`, `HybridCache` (L1 + L2 with stampede protection), FusionCache, StackExchange.Redis · Node: `lru-cache`, ioredis, node-redis.

## Gotchas

- In-process caches hand out the same object reference — mutate it and every caller sees the change. Treat cached values as immutable.
- Caching an error, timeout or empty result with a normal TTL prolongs an outage; give negative and error entries short TTLs.
- Set a TTL even when you invalidate explicitly — one lost delete otherwise means stale forever.
- "Just turn on the DB query cache" is a trap: MySQL removed its query cache in 8.0 (global lock; any write to a table flushed its entries). Cache at the application layer.
- Caches fix read load, not write load — write bottlenecks need batching, queues or sharding.

## Related

[Caching Strategies](02-caching-strategies.md) · [Cache Eviction Policies](03-cache-eviction-policies.md) · [Bloom Filters](04-bloom-filters.md) · [CDN](../03-traffic-and-edge/03-cdn.md) · [Consistent Hashing](../04-databases/07-consistent-hashing.md) · [CDC](../04-databases/08-cdc.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [Caching challenges and strategies — AWS Builders' Library](https://aws.amazon.com/builders-library/caching-challenges-and-strategies/)
- [Optimal Probabilistic Cache Stampede Prevention (Vattani et al., VLDB 2015)](https://www.vldb.org/pvldb/vol8/p886-vattani.pdf)
- [Client-side caching — Redis docs](https://redis.io/docs/latest/develop/reference/client-side-caching/)

<!-- nav -->

---

[← Geohashing](../04-databases/09-geohashing.md) · [Index](../README.md) · [Caching Strategies →](02-caching-strategies.md)
