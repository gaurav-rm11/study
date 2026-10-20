# Distributed Locks

> **TL;DR** — A distributed lock lets one process, out of many on many machines, own a resource exclusively: one job runner, one processor per order, no double-booked seat. Holders crash and stall, so every such lock is really a **lease** with a timeout, and a lock that protects correctness also needs **fencing tokens**.

## How it works

```text
t0  Client A acquires the lock → fencing token 33, lease 10 s
t1  Client A stalls (GC pause, VM freeze, slow network) … its lease expires
t2  Client B acquires the lock → token 34; writes with 34 ✓  (storage now rejects < 34)
t3  Client A resumes, still believes it holds the lock, writes with 33 ✗ rejected
    Without the token check, both writes land → corrupted data
```

- **Acquire** — atomically create a lock key holding a unique owner token and a TTL; if it already exists, fail fast, poll or queue.
- **Release** — delete the key only if it still holds *your* token (compare-and-delete). If the holder dies, the TTL frees it.
- **Requirements** — mutual exclusion (safety); no deadlock (a lease/TTL or session expiry frees a dead holder's lock); fault tolerance (the lock service survives node failures).
- **Efficiency vs correctness locks (Kleppmann)** — an *efficiency* lock only avoids duplicate work (a report built twice, a duplicate email); rare overlap is tolerable, so a single Redis node is enough. A *correctness* lock guards data (double charge, double booking); overlap must never happen, so it needs a consensus-backed lock service **and** fencing.

## Implementations

| Option | Mechanism | Safety |
|---|---|---|
| Redis, single node | `SET lock:k <random-token> NX PX 30000`; release with a Lua script that `DEL`s only if `GET` returns your token | Fast. No fencing token. Replication is async, so a failover can promote a replica that never saw the key → two holders |
| Redlock | `SET NX PX` on N independent masters (e.g. 5); held if a majority granted it within a time well under the TTL | Contested (Kleppmann vs antirez): safety rests on bounded clock drift, pauses and delays; still no fencing token |
| ZooKeeper | Ephemeral *sequential* znode under `/locks/k`; lowest number holds the lock; each waiter watches only its predecessor (no herd on release) | Session expiry deletes the znode → auto-release. FIFO-fair. The zxid or znode version works as a fencing token |
| etcd | Key bound to a lease (TTL + keep-alive); waiters queue by create revision (`concurrency.Mutex`, `etcdctl lock`) | Raft-linearizable. The key's revision is a monotonic fencing token; txns can guard writes on "I still own the key" |
| Consul | Session (tied to node health checks and/or a TTL) + KV `PUT ?acquire=<session>` | Raft-backed; *lock-delay* blocks re-acquisition for a while after a session is invalidated |
| Database | `SELECT … FOR UPDATE`; insert into a unique-keyed lock table; Postgres `pg_advisory_lock` / `pg_advisory_xact_lock`; SQL Server `sp_getapplock`; MySQL `GET_LOCK()` | Infra you already run; session-scoped locks free on disconnect. A lock taken in the same transaction as the data it guards is safe even across pauses — if the session dies, its writes roll back too |

## Advanced

- **The core hazard:** a holder can't tell that its lease expired. A GC stop-the-world, VM stall or live migration, swapping, a delayed packet or a clock jump (if TTLs use wall-clock time) can outlast the TTL, and an "am I still the owner?" check right before writing only moves the race.
- **Fencing tokens:** the lock service issues a number that grows with every grant; the protected resource stores the highest one seen and rejects lower ones (`UPDATE … SET …, fence = @t WHERE id = @id AND fence <= @t`). Sources: ZooKeeper zxid/znode version, etcd revision, Chubby *sequencers*. Redis and Redlock have none. The resource must cooperate — that is the price of real safety.
- **TTL sizing and renewal:** TTL well above the p99 critical-section time, or a short TTL plus a renewal watchdog (Redisson keeps extending its 30 s default lease while the client is alive). Renewal can't help a paused process, and a hung worker with a healthy watchdog holds the lock forever — cap the total hold time.
- **Timing assumptions (Redlock):** validity = TTL − time spent acquiring − a clock-drift allowance, assuming pauses and drift stay small. A clock jump on one master, or a master restarting without the key (no fsync), lets a second client win a majority; Redis's docs advise keeping a crashed node down for longer than the max TTL.
- **Granularity and contention:** one lock per entity (`lock:order:42`), never one global lock; a hot key serializes all its traffic.
- **Fairness, reentrancy, timeouts:** `SET NX` retry loops are unfair and stampede on release; ZooKeeper/etcd give a FIFO queue. Most locks aren't reentrant — re-acquiring your own lock deadlocks (Redisson keeps a hold count). Always acquire with a timeout plus jittered backoff, and decide up front what a timeout means: skip, retry or fail.
- **Slow calls:** don't hold an unfenced lock across slow network calls (a third-party HTTP API) — the longer the hold, the likelier the lease expires mid-call.
- **Multiple locks:** acquire in one global order (sorted keys) or try-lock all and back off; otherwise A holds X waiting for Y while B holds Y waiting for X.
- **Avoid the lock entirely:** idempotency keys + unique constraints (the DB rejects the second booking); optimistic concurrency/CAS (`WHERE version = @v`, EF Core concurrency tokens, SQL Server `rowversion`, Redis `WATCH`, etcd txns, DynamoDB conditional writes); a single writer per key (a Kafka partition owned by one consumer, Orleans grains); leader election for singleton jobs (fence with the leader's term); RabbitMQ single-active-consumer queues; `SELECT … FOR UPDATE SKIP LOCKED` for job queues (SQL Server: `UPDLOCK, READPAST`).
- **Libraries:** Redisson (Java), RedLock.net and the DistributedLock library (.NET — SQL Server, Postgres, MySQL, Redis, ZooKeeper, Azure blobs), `redlock` (Node), Apache Curator `InterProcessMutex` (ZooKeeper), etcd's Go `concurrency` package.

## Trade-offs

- ✅ Simple model for "only one at a time" across nodes; cheap to add on Redis or the database you already run.
- ✅ Consensus-backed locks (ZooKeeper, etcd, Consul) release automatically on crash and, with fencing, are actually safe.
- ❌ Without fencing, mutual exclusion is best-effort; fencing needs every protected resource to check tokens.
- ❌ Serializes work (throughput ceiling, lock convoys), adds a round trip per acquire (a quorum write on ZooKeeper/etcd), and the lock service's availability caps yours.

## In the wild

Google Chubby (coarse-grained lock service; sequencers as fencing tokens) · Apache Curator recipes on ZooKeeper · Kubernetes leader election via `Lease` objects (kube-scheduler, controller-manager) · HDFS HA: JournalNodes accept edits only from the NameNode with the highest epoch — fencing in practice · Redis/Redisson locks for cron de-duplication · .NET: Hangfire `[DisableConcurrentExecution]`, Quartz.NET clustering (DB row locks), DistributedLock · Azure Blob leases · AWS DynamoDB Lock Client.

## Gotchas

- A lock doesn't make side effects exactly-once: if the holder charges a card and crashes before recording it, the next holder charges again — you still need idempotency.
- Releasing with a plain `DEL` can delete someone else's lock after yours expired — always compare the token.
- Redis Sentinel/Cluster failover can lose a lock key (async replication); `WAIT` narrows that window but doesn't close it.
- Measure leases with a monotonic clock (`Stopwatch`, `performance.now()`), never wall-clock time — NTP corrections make wall time jump.
- Postgres session-level advisory locks break behind PgBouncer transaction pooling (lock and unlock can run on different server connections); use `pg_advisory_xact_lock`.
- Interview trap: "a distributed lock prevents double booking". The guarantee comes from a unique constraint or conditional write; the lock only reduces contention.

## Related

[Consensus Algorithms](02-consensus-algorithms.md) · [Two-Phase Commit (2PC)](03-two-phase-commit.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [ACID Transactions](../04-databases/02-acid-transactions.md) · [Replication](../04-databases/04-replication.md) · [Message Queues](../06-async-and-events/01-message-queues.md)

## Read more

- [How to do distributed locking (Martin Kleppmann)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [Is Redlock safe? — antirez's reply](https://antirez.com/news/101)
- [Distributed locks with Redis — single-instance pattern and Redlock (Redis docs)](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)

<!-- nav -->

---

[← CQRS](../06-async-and-events/05-cqrs.md) · [Index](../README.md) · [Consensus Algorithms →](02-consensus-algorithms.md)
