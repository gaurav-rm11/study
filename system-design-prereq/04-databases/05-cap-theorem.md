# CAP Theorem

> **TL;DR** — When a network partition cuts replicas off from each other, a distributed data store must either refuse some requests to stay consistent (CP) or keep answering with possibly stale or conflicting data (AP). Partitions can't be ruled out, so CAP is a choice about behavior *during* a partition; with no partition you can have both C and A.

## How it works

```text
client A ── write x=2 ──► N1 ──✗ partition ✗── N2 ◄── read x ── client B
                         (x=2)                 (x=1, can't reach N1)

CP: N2 returns an error / times out  → stays consistent, loses availability
AP: N2 returns x=1 (stale)           → stays available, loses consistency
```

- **C (Consistency)** = **linearizability**: the system behaves as if there were one copy. Once a write completes, every later read on any node returns it (or something newer), or an error.
- **A (Availability):** every request that reaches a *non-failing* node gets a non-error response, eventually (no time bound), from any node, not just the majority side.
- **P (Partition tolerance):** the system keeps working while the network drops or delays any number of messages between nodes.
- **No "CA" for distributed systems:** networks do partition (switch or NIC failure, bad firewall rule, cut cross-region link, long GC pause). "CA" describes only a single node, or a cluster that breaks unpredictably when a partition comes. The real choice is **CP or AP while partitioned**.

## CP vs AP

| | CP | AP |
|---|---|---|
| During a partition | nodes that can't reach a quorum/leader reject or time out | every reachable node keeps serving reads and writes; replicas diverge |
| After it heals | nothing to merge | reconcile divergent writes |
| Fits | leader election, locks, config, uniqueness, money, stock counts | carts, feeds, likes, presence, metrics, DNS-style lookups |
| Examples | etcd, ZooKeeper, Consul, HBase, Spanner | Cassandra, Riak, CouchDB, DynamoDB (default eventually consistent reads), DNS |

Most systems are **tunable per request or configuration**, so the label describes a default: DynamoDB `ConsistentRead=true`, Cassandra consistency levels, MongoDB read/write concerns, Cosmos DB's five levels, Consul `?stale` and etcd serializable reads (answered by any member, possibly stale).

## Advanced

- **PACELC (Abadi, 2012):** if **P**artitioned, choose **A** or **C**; **E**lse choose **L**atency or **C**onsistency. It names the everyday cost CAP ignores: strong consistency needs leader/quorum round trips on every request, failure or not. Dynamo, Cassandra and Riak (defaults) are **PA/EL**; VoltDB/H-Store and Megastore are **PC/EC**.
- **Consistency spectrum** (strong → weak): **linearizable** (one copy, real-time order) > **sequential** (one total order every node agrees on, may lag real time) > **causal** (causes seen before effects everywhere; concurrent ops in any order) > **read-your-writes / monotonic reads** > **eventual** (replicas converge once writes stop). Causal is the strongest model that can stay available during a partition (as long as each client sticks to the replicas it uses); sequential and stronger can't.
- **Brewer's correction ("CAP Twelve Years Later", 2012):** "2 of 3" is misleading. Partitions are rare, so give up nothing until one happens; choose C vs A per operation and per data item; design an explicit partition mode (limit risky operations, log the rest) and a recovery step that merges state and compensates (his ATM example: cap withdrawals while partitioned).
- **A narrow proof:** Gilbert & Lynch (2002) proved Brewer's 2000 conjecture for a single read/write register in an asynchronous network: if N1 and N2 can't communicate, a read on N2 after a write on N1 must return stale data or never answer. Transactions, latency and crashed nodes are outside the model, so CAP says nothing about most real trade-offs.
- **Partitions are detected by timeouts:** a node can't tell a dead peer from a cut link, a slow network or a paused process. When a timeout fires, CP fails the request and AP answers from local state, so **high latency ≈ partition** and timeout values are a C-vs-A knob.
- **What happens during a partition:** CP majority-quorum systems keep the majority side running (electing a new leader if needed); an isolated leader can't commit and steps down once it loses contact with the majority; minority-side clients get errors. AP systems accept writes on both sides. On heal, hinted handoff, read repair and Merkle-tree anti-entropy exchange missed writes, and conflicts resolve by **LWW** (silently drops writes; skewed clocks pick the wrong winner), **vector clocks / version vectors** (detect concurrent writes, keep siblings for the app to merge, as in Dynamo's shopping cart) or **CRDTs** (merge automatically).
- **Quorum math:** N replicas, writes wait for W acks, reads for R. **W + R > N** makes every read quorum overlap the latest successful write quorum; 2f+1 replicas with majority quorums tolerate f failures. Cassandra sets this per request: `ONE`, `QUORUM` (majority of all replicas across DCs), `LOCAL_QUORUM` (majority in the coordinator's DC: no cross-DC wait, keeps working when DCs split), `EACH_QUORUM` (majority in every DC), `ALL` (one replica down → request fails). RF=3 with `QUORUM` reads and writes: 2 + 2 > 3 and survives one replica down, yet still isn't linearizable (partial writes aren't rolled back; LWW).
- **CAP availability ≠ SLA availability:** CAP-A demands an answer from every non-failed node; SLA nines measure the share of successful requests. Spanner is technically CP, yet Brewer (2017) calls it "effectively CA" because partitions on Google's network are rare enough for better than five nines. An AP store can still be down (overload, every replica lost).
- **ACID C ≠ CAP C:** ACID consistency = transactions preserve your invariants (constraints, foreign keys); CAP consistency = recency across replicas. Likewise serializable (transactions appear in *some* serial order) ≠ linearizable (single-object operations respect real-time order); both together = **strict serializability** (Spanner, FoundationDB).
- **Kleppmann's critique ("Please stop calling databases CP or AP", 2015):** most systems meet neither definition. Single-leader async replication with replica reads isn't C (stale reads) or A (the minority side can't write); ZooKeeper serves reads locally, so they can be stale unless the client calls `sync`. Describe each operation's guarantee (linearizable, read-your-writes, eventual) and its sensitivity to network delay instead.

## Trade-offs

- ✅ **CP:** no stale reads or conflicting writes; app logic stays simple.
- ❌ **CP:** minority-side requests fail during partitions; writes (and linearizable reads) pay a leader/quorum round trip, cross-region if replicas span regions.
- ✅ **AP:** always writable; low-latency local reads and writes in every region; rides out partitions and region loss.
- ❌ **AP:** stale reads and conflicts. The app must define merges (or accept LWW's lost writes), and invariants like "never oversell" can't be enforced locally.

## In the wild

etcd (Kubernetes' source of truth), ZooKeeper (HBase, pre-KRaft Kafka), Consul, Spanner, CockroachDB: CP · Cassandra / ScyllaDB (per-request consistency levels), Riak (siblings, CRDTs), CouchDB (revision trees), DynamoDB eventually consistent reads and Global Tables (LWW by default), DNS: AP · MongoDB read/write concerns and Azure Cosmos DB (Strong, Bounded Staleness, Session, Consistent Prefix, Eventual; Session is the default): tunable · SQL Server AGs, Postgres or MySQL with async replicas: stale secondary reads and possible lost writes on failover, so neither C nor A in CAP terms · Redis Sentinel / Cluster: async replication can lose acknowledged writes.

## Gotchas

- **"Pick any 2 of 3" is the trap answer:** partitions aren't optional. Say "CP or AP under partition, latency vs consistency otherwise (PACELC)", per operation.
- **Choose per data item, not per system:** the cart can be AP while payment capture and stock decrement are CP.
- **CAP is about replicated state:** a stateless API tier isn't CP or AP; the data store behind it is.
- **"Eventually consistent" has no time bound:** measure replica lag (p99) and design the UX around it (read-your-writes, optimistic UI).
- **Quorums (W + R > N) aren't linearizability:** use consensus (Raft/Paxos, Cassandra lightweight transactions) for compare-and-set, locks and uniqueness.

## Related

[Replication](04-replication.md) · [ACID Transactions](02-acid-transactions.md) · [SQL vs NoSQL](01-sql-vs-nosql.md) · [Consensus Algorithms](../07-distributed-coordination/02-consensus-algorithms.md) · [Distributed Locks](../07-distributed-coordination/01-distributed-locks.md) · [Availability](../01-foundations/02-availability.md)

## Read more

- [CAP Twelve Years Later: How the "Rules" Have Changed — Eric Brewer](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)
- [Please stop calling databases CP or AP — Martin Kleppmann](https://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)
- [Consistency Models — Jepsen](https://jepsen.io/consistency)

<!-- nav -->

---

[← Replication](04-replication.md) · [Index](../README.md) · [Database Sharding →](06-database-sharding.md)
