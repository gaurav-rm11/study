# Consistent Hashing

> **TL;DR** — Consistent hashing puts nodes and keys on the same hash ring, so adding or removing one of N nodes remaps only ~1/N of the keys instead of nearly all of them. It's how distributed caches, Dynamo-style databases and hash-based load balancers scale in and out without a mass reshuffle.

## How it works

```text
                 A                  ring = hash space 0 … 2^32−1, wrapping around
           k4         k1            key → first node clockwise:
        D                 B         k1→B   k2→C   k3→D   k4→A
           k3         k2
                 C                  add E between k2 and C → only k2 moves (C → E)
```

- **The problem:** `node = hash(key) mod N`. When N changes, a key stays put only if `h mod N = h mod (N+1)` — going from 4 to 5 nodes remaps 80% of keys: a cache-miss storm on the database, or most of the data copied between nodes.
- **The ring:** hash node IDs and keys into the same space and wrap it into a circle. A key belongs to the first node clockwise from its hash; each node owns the arc between its predecessor and itself.
- **Membership changes:** a new node takes over part of one neighbour's arc; a removed node's arc passes to its successor. Only ~K/N of K keys move; every other key keeps its node.

## Variants and alternatives

| Scheme | How a key is placed | Lookup / memory | Notes |
|---|---|---|---|
| Ring + virtual nodes | first token clockwise | O(log V) / V tokens | general purpose; any node can join or leave; Dynamo, Cassandra, ketama |
| Rendezvous (HRW) | node with the highest `hash(key, node)` | O(N) / node list only | no ring; even spread; easy weights; removing a node moves only its keys |
| Jump consistent hash | short loop seeded by `hash(key)` | O(ln n) / none | buckets numbered 0…n−1, added or removed only at the end → sharded storage with its own replication, not caches whose nodes die at random |
| Maglev | prebuilt lookup table of prime size (Envoy default 65,537) | O(1) / the table | near-perfect balance, slightly more churn than a ring; L4 load balancers |
| Multi-probe | one point per node; hash the key k times, take the closest node | O(k log N) / N points | ring-level balance without vnodes |
| Fixed hash slots | `hash(key) mod S` → slot → node map | O(1) / S entries | not consistent hashing, same goal: Redis Cluster's 16,384 slots, Couchbase vBuckets |

## Advanced

- **Virtual nodes (vnodes):** with one point per node, arcs are badly uneven — the largest is about ln N times the average. Give each node v tokens and the load spread shrinks roughly as 1/√v (v = 100 → std dev ≈ 10% of the mean). Bonuses: a dead node's load spreads over many survivors instead of doubling one neighbour; rebalancing streams from many nodes in parallel; weight capacity by giving bigger machines more tokens.
- **Too many vnodes:** more metadata and more small ranges to repair, and every node ends up sharing ranges with nearly every other, so almost any two simultaneous node failures (RF = 3) cost some range its quorum. Part of why Cassandra 4.0 cut the default `num_tokens` from 256 to 16 (with a smarter token allocator).
- **Replication:** keep walking clockwise and take the next R **distinct physical** nodes, skipping further vnodes of hosts already chosen (Dynamo's *preference list*). Rack/AZ-aware strategies (Cassandra `NetworkTopologyStrategy`) also skip racks already used.
- **Lookup:** tokens in a sorted array; binary-search the first token ≥ `hash(key)`, wrapping to index 0 past the end — O(log V). Rebuild the array on membership change.
- **Hash function:** fast, uniform, non-cryptographic and stable across processes and languages — Murmur3 (Cassandra), xxHash; ketama used MD5. Never .NET's `string.GetHashCode()` (randomised per process); use `System.IO.Hashing.XxHash64`. If clients choose keys, a keyed hash (SipHash) stops attackers aiming everything at one node.
- **Bounded loads:** cap each node at (1+ε) × average load; a key whose node is full continues clockwise to the next node with room. Keeps affinity without hotspots (Mirrokni, Thorup, Zadimoghaddam, Google 2017; HAProxy `hash-balance-factor`, contributed by Vimeo).
- **Membership agreement:** every client must see the same node list, or one key routes to different nodes (stale reads, duplicate cache entries). Cassandra and Riak gossip the ring; others use ZooKeeper/etcd or a config service.
- **Data still moves:** consistent hashing minimises movement; it doesn't perform it. A joining Cassandra node streams its ranges from their old owners before serving them; a cache just eats misses on the moved slice. Dynamo adds hinted handoff for nodes that are briefly down.
- **Hot keys:** it balances key *counts*, not request *rates* — one viral key still lands on one node (plus its replicas). Fix with local L1 caches, extra replicas for hot keys, key splitting, or bounded loads.

## Trade-offs

- ✅ Minimal data and cache movement on scale-out, scale-in and failure; no central directory — any client computes placement from the membership list; replication falls out of "next R nodes clockwise".
- ❌ Uneven without vnodes or bounded loads; vnodes add metadata and operational overhead.
- ❌ Hashing destroys key order, so range scans hit every node (use range partitioning for those); hot keys and membership agreement remain your problem.

## In the wild

Akamai (from Karger et al., 1997) · Amazon Dynamo · Cassandra / ScyllaDB (Murmur3 tokens, vnodes) · Riak · memcached clients via ketama (libmemcached, spymemcached) · Nginx `hash $key consistent` · HAProxy `hash-type consistent` · Envoy `RING_HASH` / `MAGLEV` · Google Maglev · Discord (Elixir `ex_hash_ring`) · RabbitMQ consistent-hash exchange plugin (spreads messages across queues by routing key) · Redis Cluster uses fixed hash slots instead.

## Gotchas

- **"Consistent" has nothing to do with data consistency** — it means a stable key → node mapping, not CAP or linearizability.
- **"Only 1/N moves" assumes an even ring** — without vnodes a new node may land on a tiny arc and take almost no load.
- **Kafka's default partitioner is `murmur2(key) mod partitions`** — adding partitions remaps most keys and breaks per-key ordering; size partition counts up front.
- **Replicas on one host:** naive "next R tokens" can pick two vnodes of the same machine (or rack) — always dedupe by physical node.
- **Rolling config changes:** while clients hold different server lists they disagree on owners — expect a burst of misses and stale entries.

## Related

[Database Sharding](06-database-sharding.md) · [Replication](04-replication.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Caching](../05-caching/01-caching.md) · [CAP Theorem](05-cap-theorem.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [Dynamo: Amazon's highly available key-value store (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Consistent hashing with bounded loads — Google Research](https://research.google/blog/consistent-hashing-with-bounded-loads/)
- [A fast, minimal memory, consistent hash algorithm (jump hash) — Lamping & Veach](https://arxiv.org/abs/1406.2294)

<!-- nav -->

---

[← Database Sharding](06-database-sharding.md) · [Index](../README.md) · [CDC →](08-cdc.md)
