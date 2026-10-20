# Two-Phase Commit (2PC)

> **TL;DR** — 2PC makes one transaction atomic across several participants (databases, shards, resource managers): a coordinator first collects durable YES votes, then broadcasts the decision, so all commit or all abort. It's correct but **blocking** — a coordinator crash at the wrong moment leaves participants stuck holding locks.

## How it works

```text
Coordinator                              Participants (A, B, …)
  │── PREPARE T1 ────────────────────────►│ write changes + prepare record, fsync,
  │                                       │ keep locks, vote
  │◄──────────────────────── YES / NO ────│ after YES: "in doubt", can't decide alone
  │ fsync "COMMIT T1"  ◄── commit point (all YES; any NO or timeout → ABORT)
  │── COMMIT / ABORT T1 ─────────────────►│ apply, release locks
  │◄───────────────────────────── ACK ────│
  │ all ACKed → forget T1
```

- **Phase 1 — prepare:** each participant does everything that could fail (constraint checks, writing redo/undo), durably logs a *prepared* record, keeps its locks and votes. YES is a promise to commit if told to — even after a crash and restart.
- **Phase 2 — commit/abort:** if every vote is YES, the coordinator durably logs COMMIT — **the commit point**; from then on the transaction *will* commit, even if every node crashes. Otherwise it logs ABORT. It resends the decision until every participant ACKs, then forgets the transaction.
- **Two points of no return:** a participant's YES (it gives up the right to abort) and the coordinator's decision record.
- **Recovery:** a restarted participant finds prepared-but-undecided transactions in its log, restores their locks and asks the coordinator for the outcome.

## Failure cases

| Failure | Outcome |
|---|---|
| A participant votes NO, crashes or times out before voting | Coordinator aborts; everyone rolls back |
| A participant crashes after voting YES | On restart it restores the prepared transaction and its locks, then asks the coordinator |
| Coordinator crashes before deciding | Participants that haven't voted may abort; those that voted YES are **in doubt** — blocked, locks held, until it returns |
| Coordinator crashes after logging the decision | On restart it reads its log and resends the decision |
| Decision message lost / network partition | Participants wait; the coordinator retries until it gets through — the decision is final |

## Advanced

- **Blocking is the fatal flaw:** after voting YES a participant can neither commit (someone may have voted NO) nor abort (the coordinator may already have committed). It waits with locks held, stalling every transaction that touches those rows. Asking peers (cooperative termination) helps only if one of them knows the decision or hasn't voted yet.
- **Heuristic decisions:** an operator (or an XA transaction manager's heuristic policy) can force an in-doubt participant to commit or roll back to free its locks — risking a *heuristic mixed* outcome, i.e. broken atomicity.
- **The coordinator log is a SPOF:** it must be durable and outlive the host. Classic TMs (MSDTC, app-server TMs) log to one machine's disk; lose it and in-doubt transactions need manual resolution. Fix: replicate the coordinator's state.
- **Cost:** two round trips on the critical path; 4 messages and 2 forced log writes per participant plus the coordinator's forced decision record; locks held for at least 2 RTTs; availability is the *product* of all participants' (three at 99.9% ≈ 99.7%).
- **Timeouts:** a participant may abort on timeout only before voting YES; the coordinator may abort only before logging COMMIT. After that, the only option is retrying.
- **3PC:** adds a pre-commit phase so participants can finish on a timeout — non-blocking only if delays are bounded and crashes detectable. Under a network partition the two sides can decide differently, so it's rarely used.
- **2PC over consensus groups:** make the coordinator and every participant a replicated Paxos/Raft group so no single crash blocks — Gray & Lamport's *Paxos Commit*; Spanner (2PC across Paxos-group leaders); CockroachDB *parallel commits* (the transaction record goes STAGING in parallel with the writes and counts as committed once they're all replicated — one round of consensus); TiDB's Percolator-style 2PC over Raft.
- **Optimisations:** *presumed abort* (no forced abort record, no abort acks; a coordinator with no record answers "abort" — the common choice); *presumed commit* (no commit acks, at the cost of a forced "collecting" record up front); *read-only* participants vote READ-ONLY and skip phase 2; a single participant gets one-phase commit; *last-resource commit* lets one non-XA resource cast the deciding vote.
- **XA / JTA:** the X/Open XA interface between a transaction manager and resource managers (`xa_start`, `xa_end`, `xa_prepare`, `xa_commit`, `xa_rollback`, `xa_recover`); JTA is the Java API over it.
- **Postgres:** `PREPARE TRANSACTION 'gid'` → `COMMIT PREPARED 'gid'` / `ROLLBACK PREPARED 'gid'`; disabled by default (`max_prepared_transactions = 0`). Prepared transactions survive restarts — an orphaned one holds its locks and blocks VACUUM (eventually risking transaction-ID wraparound); watch `pg_prepared_xacts`.
- **MySQL:** `XA START / END / PREPARE / COMMIT`; `XA RECOVER` lists in-doubt transactions. MySQL also runs an internal 2PC between the binlog and InnoDB.
- **.NET:** `TransactionScope` starts as a lightweight local transaction and **escalates** to MSDTC when a second durable resource enlists (a second database, or two connections open at once). MSDTC is Windows-only: .NET 7+ supports it on Windows after setting `TransactionManager.ImplicitDistributedTransactions = true`; elsewhere escalation throws.
- **Why microservices avoid it:** synchronous coupling (every service must be up to commit), locks held across services, and most brokers and NoSQL stores (RabbitMQ, Kafka, MongoDB, DynamoDB) don't speak XA. Instead: local transactions + **saga** compensations, the **transactional outbox** (the event is written in the same local transaction and relayed later), idempotent consumers.
- **2PC isn't consensus:** 2PC needs *every* participant's YES (each has a veto) and blocks on one failure; consensus needs only a *majority* and stays live through f failures. Real systems combine them: consensus within a shard, 2PC across shards.

## Trade-offs

- ✅ Real atomicity across independent resources, with each participant's isolation intact; app code stays a plain transaction (`TransactionScope`, JTA).
- ❌ Blocking: a coordinator failure can freeze locks indefinitely, and the coordinator is a SPOF unless replicated.
- ❌ Latency (2 RTTs + several fsyncs) and lock contention grow with distance and participant count; every added participant lowers availability.
- ❌ Thin support: managed services and brokers often lack XA, and MSDTC brings Windows-only ops pain.

## In the wild

Java XA/JTA managers (Narayana, Atomikos) · Postgres `PREPARE TRANSACTION` (Citus uses it for multi-node writes) · MySQL XA · SQL Server + MSDTC (`TransactionScope`, linked servers' `BEGIN DISTRIBUTED TRANSACTION`) · Spanner, CockroachDB, YugabyteDB, TiDB (2PC over Paxos/Raft groups) · Kafka transactions (a coordinator writes commit/abort markers to every partition involved — 2PC-like, Kafka-only).

## Gotchas

- 2PC (atomic commit) ≠ 2PL (two-phase locking, a concurrency-control scheme) — a common interview mix-up.
- Atomic ≠ simultaneously visible: phase-2 commits land at different moments, so a reader can see A's change before B's — no global snapshot unless participants share timestamps (Spanner, CockroachDB).
- Orphaned prepared transactions (Postgres, MySQL XA) survive restarts and hold locks until resolved — alert on old entries in `pg_prepared_xacts` / `XA RECOVER`.
- Opening a second connection inside a `TransactionScope` can silently escalate to MSDTC — slow on Windows, an exception on Linux containers.
- The commit point is the coordinator's log write, not the participants' commits: a crash right after it still means COMMIT, and recovery must finish the job.
- "Just use 2PC across microservices" is an interview red flag — propose saga + outbox + idempotency, and say why.

## Related

[Saga Pattern](04-saga-pattern.md) · [Consensus Algorithms](02-consensus-algorithms.md) · [ACID Transactions](../04-databases/02-acid-transactions.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Database Sharding](../04-databases/06-database-sharding.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Distributed Locks](01-distributed-locks.md)

## Read more

- [Two-Phase Commit — Patterns of Distributed Systems (Unmesh Joshi, martinfowler.com)](https://martinfowler.com/articles/patterns-of-distributed-systems/two-phase-commit.html)
- [Parallel Commits: an atomic commit protocol for globally distributed transactions (CockroachDB)](https://www.cockroachlabs.com/blog/parallel-commits/)
- [PREPARE TRANSACTION (PostgreSQL docs)](https://www.postgresql.org/docs/current/sql-prepare-transaction.html)

<!-- nav -->

---

[← Consensus Algorithms](02-consensus-algorithms.md) · [Index](../README.md) · [Saga Pattern →](04-saga-pattern.md)
