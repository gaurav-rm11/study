# Cache Eviction Policies

> **TL;DR** — When a cache is full, the eviction policy decides which entry to drop for the new one — a bet on which keys won't be needed again. Matching the policy to the access pattern buys a higher hit ratio from the same memory; a mismatch (plain LRU under scans) can collapse it.

## How it works

- **Three different exits — don't conflate them:** **eviction** is about capacity (the cache is full; the victim may still be valid), **expiration** about time (the TTL elapsed), **invalidation** about correctness (the source changed, so the app deletes it).
- Every hit and insert updates policy metadata (recency order, frequency counter, reference bit); a full insert asks the policy for a victim. Policies differ in hit ratio and in what that bookkeeping costs per hit — memory, CPU, locks.

```text
LRU = hash map (key → node) + doubly linked list ordered by recency

HEAD (MRU) ⇄ [C] ⇄ [A] ⇄ [D] ⇄ [B] ⇄ TAIL (LRU)
get(A)         unlink A, relink at HEAD                          O(1)
put(E), full   unlink TAIL (B), delete B from map, add E at HEAD O(1)
```

## Policies

| Policy | Evicts | Strengths | Weaknesses |
|---|---|---|---|
| LRU | least recently used | O(1); adapts fast when popularity shifts | one scan flushes the hot set; every hit is a list write under a lock |
| LFU | least frequently used | keeps long-term favourites; ignores one-off scans | needs aging/decay or stale favourites squat forever; new keys die before building counts; O(1) needs frequency buckets |
| FIFO | oldest inserted | trivial; hits touch no metadata | ignores reuse, so hot keys get evicted on schedule |
| Random | a random entry | no metadata, no locks; decent at scale | can evict a hot key at any time |
| MRU | most recently used | loops bigger than the cache, where LRU gets 0% hits | wrong for ordinary recency-skewed traffic |
| TTL-based (`volatile-ttl`) | soonest to expire | reuses the app's TTLs as a hint | a TTL says nothing about reuse |
| Size-aware (GreedyDual-Size, GDSF) | lowest cost ÷ size score, with aging | more hits per byte when object sizes vary (CDNs, web proxies) | priority queue, O(log n) per access |

## Choosing by workload

| Workload | Reach for |
|---|---|
| General app or KV cache, unknown pattern | W-TinyLFU (Caffeine-style), else LRU |
| Recency-driven (sessions, feeds, recent orders) | LRU |
| Stable popularity (catalogue, Zipf) | LFU with decay, W-TinyLFU |
| Point reads mixed with scans (DB buffer pools, page caches) | scan-resistant: midpoint LRU, 2Q, SLRU, ARC, CLOCK-Pro |
| Many cores, very high QPS | CLOCK, SIEVE, S3-FIFO, sampled LRU |

## Advanced

- **Scan pollution:** a one-off sweep (full table scan, backup, export, crawler) touches each key once and flushes the hot set out of LRU. Scan-resistant policies make a key prove reuse before it reaches the protected area:
  - **2Q** — new keys enter a small FIFO; its evictions go to a ghost list (keys only); a ghost hit admits the key to the main LRU.
  - **SLRU** — probationary + protected LRU segments; a hit promotes, protected overflow demotes back to probation.
  - **LRU-K** — evicts by the K-th most recent access (LRU-2 is common), so keys seen once rank last.
  - **ARC** — a seen-once list and a seen-twice list, plus ghost lists that shift the split between recency and frequency online; used by ZFS; IBM's patent pushed Postgres off it.
  - **LIRS** — ranks by reuse distance (inter-reference recency); strong on loops and scans.
  - **CLOCK / CLOCK-Pro** — entries sit in a ring with a reference bit; a hit just sets the bit, a sweeping hand clears set bits and evicts the first clear one ("second chance"). Used by OS page caches and Postgres's buffer manager (clock-sweep, usage count capped at 5); CLOCK-Pro approximates LIRS.
  - **In practice:** InnoDB inserts new pages at the midpoint (old sublist, 37% by default) and promotes them only if read again more than `innodb_old_blocks_time` (1 s) after the first read; Postgres pushes large sequential scans through a small ring buffer.
- **Admission vs eviction:** eviction picks who leaves; admission decides whether a newcomer deserves a slot at all. LRU admits everything, so one-hit wonders displace hot keys.
- **W-TinyLFU (Caffeine):** new keys land in a small window LRU (~1% of capacity); the window's victim competes with the main region's victim, and a **Count-Min sketch** of 4-bit frequency counters admits whichever is more popular. The main region is SLRU (80% protected). Counters are halved periodically (aging), and Caffeine hill-climbs the window size.
- **FIFO is back:** **S3-FIFO** (SOSP 2023) — a small FIFO (~10%), a main FIFO and a ghost FIFO of keys; most objects are one-hit wonders, so they leave the small queue fast and only re-accessed ones reach main. **SIEVE** (NSDI 2024) — one FIFO with a visited bit and a hand that evicts the first unvisited entry, leaving survivors in place. Neither moves entries on a hit, so both scale with cores; SIEVE targets web/KV caches and isn't scan-resistant.
- **Bélády's MIN (1966):** evict the key whose next use is furthest in the future — optimal but clairvoyant. Replay a production trace through it and your candidates at the target size: it's the hit-ratio ceiling (byte hit ratio when sizes vary).
- **Redis `maxmemory-policy`:** `noeviction` (default), `allkeys-lru`, `allkeys-lfu`, `allkeys-random`, `volatile-lru`, `volatile-lfu`, `volatile-random`, `volatile-ttl`; `volatile-*` only consider keys with a TTL. Pure cache → `allkeys-lru`/`allkeys-lfu`; cache keys mixed with must-keep data → `volatile-*`.
- **Redis approximates:** no linked list — each key stores a 24-bit clock; eviction samples `maxmemory-samples` keys (default 5) into a pool of candidates and evicts the best. 10 samples ≈ true LRU at more CPU.
- **Redis LFU:** the same 24 bits hold a 16-bit decay timestamp and an 8-bit **logarithmic counter** — increments get less likely as it grows (`lfu-log-factor`, default 10, saturates around 1M hits); it decays with idle time (`lfu-decay-time`, default 1 min); new keys start at 5 so they aren't evicted instantly. `OBJECT FREQ key` shows it.
- **Redis expiry:** expired keys are removed lazily on access, plus an active cycle (~10×/s by default) that samples keys with TTLs and repeats while a large share of the sample was expired — so expired keys can occupy memory for a while.
- **Memcached:** a slab allocator groups items into size classes (chunk sizes grow ×1.25 by default) and keeps an LRU **per slab class**, so one class can evict while another sits on free memory (slab calcification; fixed by slab automove). Each class is a segmented LRU (HOT/WARM/COLD) maintained by a background thread, so most hits just set a flag.
- **Concurrency:** strict LRU turns every read into a write under a lock — contention on many cores. Escapes: sampling (Redis), CLOCK (set a bit), buffered reordering (Caffeine records hits in ring buffers and replays them under a try-lock), lock striping across segments, FIFO-family policies.
- **Implement LRU (LeetCode 146):** O(1) `get`/`put`. C#: `Dictionary<K, LinkedListNode<(K Key, V Value)>>` + `LinkedList<(K, V)>` — on hit `Remove(node)` then `AddFirst(node)`; when full `RemoveLast()` and delete its key from the dictionary (why nodes store the key). JS: `Map` iterates in insertion order — `delete` + `set` moves a key to the newest end; evict `map.keys().next().value`. Follow-ups: thread safety, per-entry TTL, O(1) LFU (LeetCode 460: frequency buckets + a min-frequency pointer).

## Trade-offs

- ✅ Recency-based (LRU, CLOCK): cheap, adapts instantly to shifting popularity.
- ❌ Recency-based: scan pollution; strict LRU's per-hit lock caps multi-core throughput.
- ✅ Frequency-based (LFU, TinyLFU): protects long-term hot keys, filters one-hit wonders.
- ❌ Frequency-based: needs decay, reacts slowly to new hot keys, spends memory on counters.
- ✅ Adaptive (ARC, W-TinyLFU): robust across workloads without tuning.
- ❌ Adaptive: more code and metadata (ghost lists, sketches); approximations (sampling, CLOCK) give up a little hit ratio for a lot of throughput.

## In the wild

Redis / Valkey (sampled LRU/LFU) · Memcached (segmented LRU per slab class) · Caffeine (W-TinyLFU), Ristretto (Go; TinyLFU admission + sampled LFU) · .NET: `MemoryCache` with `SizeLimit` (compacts expired entries, then by priority, then LRU), BitFaster.Caching (`ConcurrentLru`, `ConcurrentLfu`) · Node: `lru-cache` · Postgres clock-sweep + ring buffers · MySQL InnoDB midpoint-insertion LRU · Linux page cache (active/inactive lists; multi-gen LRU since 6.1) · ZFS ARC · Squid (heap GDSF).

## Gotchas

- Self-hosted Redis defaults to no memory limit (`maxmemory 0` on 64-bit) and `noeviction`: it grows until the OS kills it, and once `maxmemory` is set a full instance rejects writes with OOM errors instead of evicting. Set both explicitly for a cache (ElastiCache defaults to `volatile-lru`).
- `volatile-*` with no TTL'd keys has nothing to evict → the same OOM errors.
- Rising `evicted_keys` means the working set doesn't fit; rising `expired_keys` is normal — separate alarms.
- .NET `IMemoryCache` is unbounded by default; with `SizeLimit` set, every entry must declare `Size` (or it throws), and inserts over the limit are silently dropped while compaction runs in the background.

## Related

[Caching](01-caching.md) · [Caching Strategies](02-caching-strategies.md) · [Bloom Filters](04-bloom-filters.md) · [CDN](../03-traffic-and-edge/03-cdn.md) · [Concurrency vs Parallelism](../01-foundations/05-concurrency-vs-parallelism.md)

## Read more

- [Key eviction — Redis docs](https://redis.io/docs/latest/develop/reference/eviction/)
- [Caffeine wiki: Efficiency (W-TinyLFU)](https://github.com/ben-manes/caffeine/wiki/Efficiency)
- [SIEVE is Simpler than LRU (Zhang et al., NSDI 2024)](https://www.usenix.org/conference/nsdi24/presentation/zhang-yazhuo)

<!-- nav -->

---

[← Caching Strategies](02-caching-strategies.md) · [Index](../README.md) · [Bloom Filters →](04-bloom-filters.md)
