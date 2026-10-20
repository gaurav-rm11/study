# Bloom Filters

> **TL;DR** — A Bloom filter is a bit array that answers "is x in the set?" with **definitely not** or **possibly yes**: never a false negative, false positives at a rate you choose. It lets you skip an expensive lookup (disk, network, DB) for keys that aren't there, at ~10 bits per key whatever the key's size.

## How it works

```text
m = 16 bits, k = 3     add "cat" → bits 2, 7, C     add "dog" → bits 5, 7, E
index (hex)  0 1 2 3 4 5 6 7 8 9 A B C D E F
bits         0 0 1 0 0 1 0 1 0 0 0 0 1 0 1 0
has "eel"?   bits 2, 9, E → bit 9 is 0 → definitely not ✗
has "fox"?   bits 5, C, E → all 1      → maybe — a false positive, never added
```

- **Insert:** hash the key with k functions (mod m) and set those k bits to 1.
- **Lookup:** any of the k bits is 0 → definitely absent; all 1 → possibly present (other keys may have set them). Both are O(k), independent of n.
- **No false negatives** because bits only ever flip 0 → 1.
- **No delete** (clearing a bit can erase other keys → false negatives), **no enumeration** and no values: the filter keeps bits, not keys.

## The math

n = keys, m = bits, k = hash functions, p = false-positive rate:

```text
p ≈ (1 − e^(−kn/m))^k
optimal k = (m/n)·ln 2 = log2(1/p)        → at the optimum half the bits are 1
m = −n·ln p / (ln 2)²  ≈ 1.44·n·log2(1/p)
```

| Target p | Bits per key (m/n) | Optimal k |
|---|---|---|
| 1% | 9.6 | 7 |
| 0.1% | 14.4 | 10 |
| 0.01% | 19.2 | 13 |

- **Rule of thumb:** each extra ~4.8 bits per key cuts p by 10×.
- **Worked example:** 100 M keys at 1% → m ≈ 100 M × 9.6 = 960 Mbit ≈ 120 MB, k = 7; at 0.1% ≈ 180 MB, k = 10. A hash set of 100 M 40-byte IDs needs > 4 GB.

## Use cases

| Use case | Filter holds | Payoff | A false positive costs |
|---|---|---|---|
| LSM engines (Cassandra, RocksDB, LevelDB, HBase) | keys in each SSTable | skip reading SSTables that can't hold the key | one wasted SSTable read |
| Cache-penetration guard | every valid ID | lookups for nonexistent IDs (bugs, scrapers, attacks) never reach the DB | one DB query |
| Crawler "seen URLs" | URLs already fetched | no re-fetches, ~10 bits per URL | that page is never crawled |
| CDN one-hit wonders (Akamai) | URLs requested once | cache only on the 2nd request, so one-off objects don't evict hot ones | an object cached on its 1st request |
| Username "taken?" pre-check | registered names | instant "available" without a DB round-trip | one DB check (the unique constraint still decides) |
| Bloom join (Spark runtime filters, SQL Server bitmap filters) | join keys of the smaller side | drop big-side rows that can't match before shuffling/joining | a row survives to the real join |

## Advanced

- **Double hashing (Kirsch–Mitzenmacher):** derive all k indexes from two hashes, gᵢ(x) = h1(x) + i·h2(x) mod m, with no asymptotic FPR penalty — in practice one 128-bit Murmur3/xxHash split into two 64-bit halves (Guava does exactly this).
- **Overfilling:** p climbs steeply past design capacity — a filter sized for 1% at n is at ~16% by 2n — and it can't be resized because the keys aren't stored, so size for peak n. Watch the fill ratio (~50% of bits set = at capacity, for optimal k) or estimate n ≈ −(m/k)·ln(1 − X/m) from X set bits. **Scalable Bloom filters** add a larger layer with a tighter p each time one fills; lookups check every layer (RedisBloom does this unless `NONSCALING`).
- **Deletes:** **counting Bloom filters** swap bits for ~4-bit counters (4× memory; decrement to delete). **Cuckoo filters** keep short fingerprints in a cuckoo hash table (2 candidate buckets × 4 slots): deletable, ≤ 2 cache misses per lookup, smaller than Bloom below ~3% FPR; inserts start failing near 95% load. **Quotient filters** split a fingerprint into quotient (slot) and remainder: deletable, mergeable, resizable, sequential access (SSD-friendly). Deleting a never-inserted key corrupts any of them.
- **Static sets:** **XOR** and **binary fuse filters** are built once from a fixed key set (no later inserts) and need ~1.23× and ~1.13× the theoretical minimum space vs Bloom's 1.44×, with 3–4 memory reads per lookup.
- **Cache-friendly layouts:** a **blocked** filter hashes each key to one 64-byte cache line and sets all k bits inside it → 1 cache miss instead of k, for a slightly higher FPR at equal memory (RocksDB's newer Bloom is cache-local; its **Ribbon** filter saves ~30% memory for more CPU). **Partitioned** filters give each hash its own m/k slice.
- **Set algebra:** with identical m, k and hash seeds, bitwise **OR** gives exactly the union's filter; **AND** approximates the intersection (FPR ≥ a filter built from it directly). Filters are small enough to ship between nodes — the basis of Bloom joins.
- **Sibling sketches:** **Count-Min sketch** answers "how often?" (d rows × w counters, take the min; only over-estimates; heavy hitters, TinyLFU cache admission in Caffeine). **HyperLogLog** answers "how many distinct?" — Redis `PFADD` / `PFCOUNT` / `PFMERGE`, ≤ 12 KB per key, 0.81% standard error.
- **Adversaries and privacy:** with known, unseeded hashes an attacker can craft keys that saturate the filter (FPR → 100%, every request hits the DB) or collide with a target; use keyed hashes (SipHash or a random per-filter seed). A published filter leaks membership — anyone can test candidates offline (Bitcoin's BIP 37 SPV filters exposed which addresses a wallet owned).

## Trade-offs

- ✅ ~10 bits per key at 1% regardless of key size; O(k) insert and lookup; a "no" is always safe to act on.
- ✅ Mergeable with OR and trivially serialisable — ships alongside SSTables, between nodes, or to clients.
- ❌ A "maybe" still pays for the real lookup, so it only helps when most lookups are misses and the miss path is expensive.
- ❌ No delete, enumerate or count; capacity fixed at creation; k random memory reads per lookup unless blocked.

## In the wild

LSM knobs: Cassandra per-table `bloom_filter_fp_chance`, RocksDB `bits_per_key` (10 ≈ 1%) · RedisBloom (Redis Stack; built into Redis 8+): `BF.RESERVE key 0.01 1000000`, `BF.ADD`, `BF.EXISTS`, `BF.MADD`; cuckoo `CF.ADD` / `CF.EXISTS` / `CF.DEL`; `CMS.*`, `TOPK.*` · no module: k × `SETBIT` / `GETBIT` on a plain Redis bitmap (≤ 2³² bits = 512 MB per key), pipelined · .NET: NRedisStack `db.BF()` / `db.CF()` · Node: node-redis `client.bf.*`, `bloom-filters` npm · Java: Guava `BloomFilter` · Postgres: `bloom` index extension (multi-column equality), BRIN bloom operator classes (PG 14+) · Bitcoin BIP 158 compact block filters (Golomb-coded sets, replacing BIP 37).

## Gotchas

- **Keep it in sync with the source of truth:** a key in the DB but not yet in the filter is a false negative in practice — a cache-penetration guard 404s a brand-new user. Add to the filter on (or before) the write, rebuild periodically, and fail open if the filter is unavailable.
- **Source deletes push p up:** deleted keys answer "maybe" forever; rebuild on a schedule or use a cuckoo filter.
- **Don't run k cryptographic hashes** — one fast 128-bit hash plus double hashing is standard; seed it if inputs are attacker-controlled.
- **Merging needs identical m, k, hash functions and seeds** — fix them cluster-wide or filters built on different nodes can't be OR'd.
- **Choose the error direction:** when the filter answers "already seen?", false positives *skip* work that should have happened (unfetched pages, hidden feed items). If that's unacceptable, let the filter gate only a definitive check.

## Related

[Caching](01-caching.md) · [Caching Strategies](02-caching-strategies.md) · [Cache Eviction Policies](03-cache-eviction-policies.md) · [CDN](../03-traffic-and-edge/03-cdn.md) · [Database Indexes](../04-databases/03-database-indexes.md) · [SQL vs NoSQL](../04-databases/01-sql-vs-nosql.md) · [Idempotency](../06-async-and-events/02-idempotency.md)

## Read more

- [Network Applications of Bloom Filters: A Survey — Broder & Mitzenmacher](https://www.eecs.harvard.edu/~michaelm/postscripts/im2005b.pdf)
- [Cuckoo Filter: Practically Better Than Bloom — Fan et al.](https://www.cs.cmu.edu/~dga/papers/cuckoo-conext2014.pdf)
- [Bloom filter calculator — Thomas Hurst](https://hur.st/bloomfilter/)

<!-- nav -->

---

[← Cache Eviction Policies](03-cache-eviction-policies.md) · [Index](../README.md) · [Message Queues →](../06-async-and-events/01-message-queues.md)
