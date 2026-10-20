# Replication

> **TL;DR** — Replication keeps copies of the same data on several nodes so you survive node and region loss without losing data, scale reads, serve distant users from a nearby copy, and run backups and analytics off the primary. Every design trades how quickly and safely the copies agree against write latency and availability.

## How it works

```text
           writes           WAL / binlog stream
Client ──────────► Leader ──────────┬──────► Follower 1 ──► reads
                                    ├──────► Follower 2 ──► reads
                                    └──────► Follower 3 ──► backups, analytics
```

- **Leader-follower** (primary-replica): every write goes to the leader, which appends it to its log (Postgres WAL, MySQL binlog, SQL Server transaction log) and streams the log to followers; they apply it in the same order. Reads can go to any node.
- **Log position:** each follower tracks how far it has applied (Postgres **LSN**, MySQL **GTID** set). A new follower restores a snapshot, then replays the log from the snapshot's position.
- **Replication lag** = how far a follower trails the leader: usually milliseconds, minutes under write bursts, long transactions or a slow link. Follower reads are *eventually consistent*.

## Sync vs async vs semi-sync

| Mode | Leader confirms the commit after | Data lost if the leader dies (RPO) | Write latency | Write availability |
|---|---|---|---|---|
| Synchronous | the required replicas confirm (Postgres `synchronous_standby_names`: `FIRST n`, or quorum `ANY n`) | none | + round trip to the slowest required replica | a slow or dead sync replica stalls writes |
| Semi-sync | ≥ 1 replica has *received* the change (MySQL semi-sync), or "one sync follower, rest async" | ~none | + one round trip | MySQL drops to async after an ack timeout (RPO > 0 until it recovers) |
| Asynchronous | the local write only | the unshipped tail | lowest | best: replicas never block writes |

Common setup: sync to any 1 of 2 in-region replicas (one slow replica can't block commits) plus async replicas in other regions.

## Topologies

| Topology | Who accepts writes | Conflicts | Examples |
|---|---|---|---|
| Single-leader | one leader per dataset or shard | none: one log order | Postgres, MySQL, SQL Server AGs, MongoDB, Redis |
| Multi-leader | one leader per region or device | yes, must be resolved | Cosmos DB multi-region writes, DynamoDB Global Tables, CouchDB/PouchDB, Postgres BDR |
| Leaderless | any replica, via quorums | yes: versions, LWW | Dynamo, Cassandra, ScyllaDB, Riak |
| Chain | the head; the tail acks and serves reads | none | CRAQ, HDFS write pipeline |
| Consensus (Raft / Paxos) | each group's leader; committed once a majority has it | none | etcd, CockroachDB, TiKV, YugabyteDB, Spanner |

## Advanced

- **Read-your-writes:** a user's save "vanishes" when their next read hits a lagging follower. Read what they may have just edited from the leader (or everything for a few seconds after a write), or keep the LSN/GTID of their last write and only read from a replica past it (Postgres `pg_last_wal_replay_lsn()`, MySQL `WAIT_FOR_EXECUTED_GTID_SET()`, MongoDB causally consistent sessions). Store it server-side for cross-device consistency.
- **Monotonic reads:** successive reads hit replicas with different lag, so data appears then disappears. Pin each user to one replica (hash of user ID); re-pin only on failure.
- **Consistent prefix reads:** causally related writes on different shards replicate independently, so a reader sees an answer before its question. Keep related writes on one shard, or track causal dependencies.
- **Failover:** missed heartbeats → promote the most up-to-date replica (highest LSN/GTID) → repoint clients (VIP, DNS, proxy, service discovery) → rejoin the old leader as a follower.
- **Failover pitfalls:** async replication loses acknowledged-but-unshipped writes (GitHub, 2012: a promoted, lagging MySQL replica reissued auto-increment IDs already used as Redis keys → private data shown to the wrong users). **Split-brain:** an old leader that was only partitioned or GC-paused keeps taking writes. Fence it (STONITH: power off or cut off the node; epoch/term numbers that storage rejects when stale) and let a consensus-backed manager choose the leader (Patroni's TTL'd leader key in etcd/Consul/ZooKeeper; MongoDB elections and SQL Server's WSFC need a majority). Aggressive timeouts trigger false failovers under load spikes → flapping.
- **Log formats:** *statement-based* ships SQL: compact, but nondeterminism (`UUID()`, `SYSDATE()`, `UPDATE … LIMIT` without `ORDER BY`) makes replicas diverge. *Physical* ships WAL byte changes: exact and cheap, but same engine and major version only (Postgres streaming). *Logical / row-based* ships row before/after images: cross-version, per-table (zero-downtime major upgrades), and what CDC tools like Debezium consume (Postgres logical replication; MySQL `binlog_format=ROW`, the default).
- **Replay is the bottleneck:** the leader writes from many sessions at once, but Postgres replays WAL in one process and MySQL's parallel appliers only parallelize independent transactions → write-heavy leaders outrun replicas.
- **Multi-leader:** each region (or offline device) writes locally and replicates asynchronously → local write latency, survives region loss, but concurrent writes to one key conflict. **LWW** by timestamp silently drops a write, and a fast clock wins even when its write came first; **app merge** keeps both versions (siblings) and merges on read or write; **CRDTs** (counters, sets, maps) merge deterministically (Riak data types, Redis Enterprise Active-Active, Automerge/Yjs). Best: avoid conflicts by homing each key/user to one region. Auto-increment IDs collide → UUIDs or interleaved ranges (MySQL `auto_increment_increment` / `auto_increment_offset`).
- **Leaderless (Dynamo-style):** writes go to all N replicas and succeed after W acks; reads ask R replicas and keep the newest version. **W + R > N** makes the sets overlap (N=3, W=2, R=2); W=N, R=1 favors reads, W=1 favors writes. Stale copies converge via **read repair** (write the newest version back during a read) and **anti-entropy** (compare **Merkle trees** in the background, ship only differing ranges). **Sloppy quorum** (Dynamo, Riak): if home replicas are unreachable, stand-ins take the write with a **hint** and hand it back later (**hinted handoff**). Writes stay available, but W + R > N no longer guarantees overlap.
- **Quorums aren't linearizable:** a read racing a write can return the new value and the next read the old one; a write with fewer than W acks isn't rolled back where it landed; LWW drops concurrent writes. Compare-and-set needs consensus (Cassandra's Paxos-based lightweight transactions).
- **Chain replication:** each write passes head → … → tail, giving strong consistency with work spread along the chain; one slow node stalls every write, and an external coordinator must repair the chain. CRAQ lets every node serve reads of committed objects.
- **Consensus replication:** each shard is a Raft/Paxos group of 2f+1 replicas tolerating f failures. A write commits once a majority logs it, so failover never loses acknowledged writes or splits the brain. Cost: a majority round trip per write (cross-region if the group spans regions). MongoDB's election protocol is Raft-based. **Kafka's ISR** is a cheaper cousin: `acks=all` waits for every in-sync replica, laggards are dropped from the ISR instead of blocking, and `min.insync.replicas` sets how small it may shrink.

## Trade-offs

- ✅ Survives node, AZ and region loss; scales reads; low-latency local reads; backups and analytics run off the primary.
- ❌ Doesn't scale writes: every replica applies every write. Shard for that.
- ❌ Async → stale reads and lost writes on failover; sync → slower writes that stall when replicas do.
- ❌ Multi-leader and leaderless push conflict resolution into your app; every copy costs storage, bandwidth and ops work.

## In the wild

Postgres streaming (physical) + logical replication (publications/subscriptions), Patroni for HA · MySQL binlog + GTID replication, semi-sync, Group Replication / InnoDB Cluster (Paxos-based, single- or multi-primary) · SQL Server Always On Availability Groups (sync/async commit, readable secondaries), Azure SQL active geo-replication · MongoDB replica sets (`w: "majority"`) · Redis async replication + Sentinel (`WAIT` blocks until N replicas ack) · Kafka ISR · RabbitMQ quorum queues (Raft) · Amazon Aurora (6 storage copies across 3 AZs, 4-of-6 write quorum).

## Gotchas

- **Replication is not a backup:** `DROP TABLE`, a bad migration or ransomware reaches every replica in milliseconds. Keep point-in-time recovery (base backup + archived WAL/binlog; SQL Server full + log backups); a delayed replica (Postgres `recovery_min_apply_delay`, MySQL `SOURCE_DELAY`) adds a quick undo window.
- **Sending every SELECT to replicas breaks read-after-write** (the classic read/write-splitting bug): uniqueness checks, balances and the page shown right after a save must read from the leader.
- **Postgres replication slots keep WAL until it's consumed:** a dead replica or stalled CDC connector fills the primary's disk. Cap it with `max_slot_wal_keep_size` and alert on slot lag.
- **MySQL `Seconds_Behind_Source` can show 0 while far behind:** it only compares the applier with relay logs already received. Measure lag with a heartbeat table (pt-heartbeat).
- **Long queries on a Postgres hot standby get cancelled** ("conflict with recovery") when replay must remove rows they still read; `max_standby_streaming_delay` trades that for lag, `hot_standby_feedback` for bloat on the primary.

## Related

[SQL vs NoSQL](01-sql-vs-nosql.md) · [CAP Theorem](05-cap-theorem.md) · [Database Sharding](06-database-sharding.md) · [Consistent Hashing](07-consistent-hashing.md) · [Change Data Capture (CDC)](08-cdc.md) · [Consensus Algorithms](../07-distributed-coordination/02-consensus-algorithms.md) · [Availability](../01-foundations/02-availability.md)

## Read more

- [Eventually Consistent — Werner Vogels](https://www.allthingsdistributed.com/2008/12/eventually_consistent.html)
- [Dynamo: Amazon's Highly Available Key-value Store (SOSP 2007)](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [High Availability, Load Balancing, and Replication — PostgreSQL docs](https://www.postgresql.org/docs/current/high-availability.html)

<!-- nav -->

---

[← Database Indexes](03-database-indexes.md) · [Index](../README.md) · [CAP Theorem →](05-cap-theorem.md)
