# Consensus Algorithms

> **TL;DR** — Consensus gets a group of nodes to agree on a value — in practice an ordered log of commands — even while some crash and messages are delayed. Leader election, replicated state machines and coordination stores (etcd, ZooKeeper, Consul) are built on it.

## How it works

- **Replicated state machine:** every node applies the same deterministic commands in the same order, so all end in the same state; consensus fixes the order, one log slot at a time.
- **Properties:** *agreement* (no two correct nodes decide differently), *validity* (the decided value was actually proposed), *termination* (every correct node eventually decides). The first two are safety; the third is liveness.
- **Uses:** leader election, replicated logs, config/metadata stores, locks with fencing, cluster membership.
- **Majority quorums:** 2f+1 nodes tolerate f crashed nodes (3 → 1, 5 → 2, 7 → 3). Any two majorities share a node, so a new leader always finds every committed entry. Even sizes buy nothing: 4 nodes still tolerate only 1 failure but need 3 acks.
- **Cost:** every write = a round trip from the leader to a majority + an fsync on each acknowledging node, so latency ≥ RTT to the nearest majority + a disk flush.

## Raft

```text
          election timeout, term+1           majority of votes
Follower ──────────────────────────► Candidate ─────────────────► Leader
    ▲                                    │                           │
    └──────────────── sees a higher term ┴───────────────────────────┘

Client ─► Leader appends x=5 at (term 4, index 7) ─► AppendEntries ─► F1 ✓ F2 ✓ F3 ✗ F4 ✗
          stored on 3 of 5 (leader, F1, F2) → committed → applied → reply to client
```

- **Roles and terms:** one leader, the rest followers; candidates exist only during elections. A term is a numbered epoch with at most one leader; any node that sees a higher term adopts it and steps down.
- **Election:** a follower that hears no heartbeat within its **randomised** election timeout (150–300 ms in the paper) increments the term, votes for itself and sends `RequestVote`. Randomisation makes split votes rare; a split vote just times out into the next term.
- **Voting rules:** one vote per term, first come first served, and only for a candidate whose log is at least as up to date (higher last term, or same last term and ≥ last index) — so the winner holds every committed entry.
- **Log replication:** the leader appends each command and sends `AppendEntries` (empty ones double as heartbeats) carrying the previous entry's index and term; a follower rejects a mismatch, and the leader walks back and overwrites the follower's conflicting suffix.
- **Commit:** an entry is committed once stored on a majority; the leader then applies it and replies. It only counts replicas for entries from its *own* term; older entries commit indirectly (Figure 8 of the paper).
- **Safety properties:** election safety (≤ 1 leader per term), leader append-only, log matching (same index and term ⇒ identical prefix), leader completeness (committed entries appear in every later leader's log), state-machine safety.
- **Membership changes:** add/remove one server at a time (old and new majorities always overlap), or *joint consensus* (C_old,new needs majorities of both) for arbitrary changes.
- **Snapshots:** compact applied state into a snapshot and truncate the log; `InstallSnapshot` catches up followers that fell too far behind.

## Algorithms

| Algorithm | Key idea | Used by |
|---|---|---|
| Paxos | Proposers, acceptors, learners. Phase 1: `prepare(n)` → acceptors *promise* to ignore lower numbers and report any value already accepted. Phase 2: `accept(n, v)`, where v must be the highest-numbered reported value (else the proposer's own) → chosen once a majority has *accepted* | The theoretical foundation; hard to turn into a full system |
| Multi-Paxos | A stable leader runs phase 1 once, then one round trip per log slot — effectively Raft | Chubby, Spanner |
| Raft | Strong leader; the log flows only leader → followers; designed for understandability | etcd, Consul, CockroachDB, TiKV, Kafka KRaft, RabbitMQ quorum queues |
| Zab | Primary-backup atomic broadcast with epochs | ZooKeeper |
| Viewstamped Replication | Primary + view changes; published before Paxos and close to Raft | TigerBeetle |
| EPaxos, leaderless variants | Any replica can lead a command; non-conflicting commands commit in one round trip | Mostly research, WAN setups |
| PBFT, Tendermint, HotStuff | Byzantine-tolerant: 3f+1 nodes, 2f+1 quorums | Blockchains, permissioned ledgers |

## Advanced

- **FLP impossibility:** in a fully asynchronous network no deterministic protocol can guarantee termination if even one node may crash. Real algorithms are always safe and live under *partial synchrony* via timeouts — during instability they stall, never disagree.
- **Split-brain prevention:** terms/epochs/ballot numbers make a deposed leader harmless — its stale term can't gather a majority, and anyone seeing a higher term steps down. Hand the term to external systems as a fencing token. *Pre-Vote* stops a rejoining partitioned node from forcing elections; *CheckQuorum* makes an isolated leader step down (both in etcd).
- **Linearizable reads:** a leader can be deposed without knowing it, so plain local reads may be stale. Options: put the read through the log (slow); **ReadIndex** — record the commit index, confirm leadership with one heartbeat round, serve once applied (etcd's default); **leader leases** — skip the round by trusting bounded clock drift (CockroachDB, TiKV). ZooKeeper serves reads locally (sequentially consistent); call `sync` first for a fresh read.
- **Byzantine faults:** if nodes may lie or be compromised you need 3f+1 nodes (PBFT, Tendermint, HotStuff). Raft and Paxos are crash-fault tolerant — they assume honest nodes, which fits one organisation's own servers.
- **Expensive, so keep it small or shard it:** one leader caps write throughput, so use consensus for small critical metadata (config, leases, membership, schema) — or run thousands of independent groups: Multi-Raft (CockroachDB ranges, TiKV regions, YugabyteDB tablets), Spanner's per-split Paxos groups. Transactions across groups then run 2PC on top.
- **Multi-region:** commit latency ≥ RTT to a majority. 5 replicas over 3 regions (2-2-1) survive a region loss; keep the leader near the writers and serve local reads from followers via ReadIndex or leases.
- **Learners and witnesses:** learners/non-voters (etcd learners, ZooKeeper observers) replicate the log without voting — they scale reads and catch up before promotion without hurting the quorum. Witnesses vote but hold no full copy of the data (Spanner witness replicas) — cheap tie-breakers.
- **Gossip isn't consensus:** SWIM-style gossip (Consul's Serf, Cassandra) spreads membership and failure suspicion with eventual convergence — no agreed order, no safety guarantee. Consul pairs gossip for membership with Raft for its catalog/KV; Cassandra runs Paxos only for lightweight transactions.
- **Logical clocks order events without agreement:** Lamport clocks (a total order consistent with causality; can't detect concurrency), vector clocks (detect concurrent updates; Dynamo, Riak), hybrid logical clocks (physical time + counter; CockroachDB, YugabyteDB, MongoDB), TrueTime (GPS + atomic clocks expose an uncertainty bound that Spanner waits out — *commit wait* — for external consistency).

## Trade-offs

- ✅ Linearizable state with automatic failover and no split brain; survives a minority of failures without human help.
- ❌ Every write pays a majority round trip + fsync; one leader caps throughput; more voters mean slower writes.
- ❌ The minority side of a partition can't make progress (CP), and a leader crash costs roughly an election timeout or more of write unavailability.
- ❌ Notoriously hard to implement correctly — reuse etcd's `raft`, HashiCorp `raft`, Apache Ratis or an existing store instead of rolling your own.

## In the wild

etcd (all Kubernetes cluster state) · Consul (Raft + Serf gossip) · ZooKeeper (Zab; HBase, Hadoop, Kafka before KRaft) · Kafka KRaft (Raft-based metadata quorum; Kafka 4.0 dropped ZooKeeper) · Google Chubby and Spanner (Multi-Paxos) · CockroachDB, TiKV/TiDB, YugabyteDB (Multi-Raft) · RabbitMQ quorum queues (Raft via the Ra library) · MongoDB replica-set elections (Raft-inspired) · Postgres HA via Patroni on etcd/Consul/ZooKeeper · SQL Server Always On via Windows Server Failover Clustering quorum · .NET: DotNext Raft (`DotNext.Net.Cluster`).

## Gotchas

- Async primary–replica replication with automatic failover is *not* consensus: it can lose acknowledged writes or split-brain (a Jepsen favourite).
- Two datacentres can't give site-level fault tolerance: whichever site holds the majority takes the cluster down when it fails. Add a third site, even just a tie-breaker.
- Adding voters adds latency, not throughput; scale reads with learners/followers and writes by sharding into more groups.
- etcd and ZooKeeper are coordination stores, not databases: keep data small (etcd's default backend quota is 2 GB) and disks fast — slow fsync means missed heartbeats and leader churn.
- Election timeout ≫ heartbeat interval ≫ RTT (etcd defaults: 100 ms heartbeat, 1,000 ms election); keep LAN defaults on a cross-region cluster and leaders flap.
- Interview trap: "nodes vote on every write". The leader orders writes and followers just acknowledge; votes happen only in elections.

## Related

[Distributed Locks](01-distributed-locks.md) · [Two-Phase Commit (2PC)](03-two-phase-commit.md) · [Replication](../04-databases/04-replication.md) · [CAP Theorem](../04-databases/05-cap-theorem.md) · [Database Sharding](../04-databases/06-database-sharding.md) · [Availability](../01-foundations/02-availability.md) · [SPOF](../01-foundations/03-spof.md)

## Read more

- [In Search of an Understandable Consensus Algorithm — the Raft paper (Ongaro & Ousterhout)](https://raft.github.io/raft.pdf)
- [Paxos Made Simple (Leslie Lamport)](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)
- [The Secret Lives of Data — Raft, visualised](https://thesecretlivesofdata.com/raft/)

<!-- nav -->

---

[← Distributed Locks](01-distributed-locks.md) · [Index](../README.md) · [Two-Phase Commit →](03-two-phase-commit.md)
