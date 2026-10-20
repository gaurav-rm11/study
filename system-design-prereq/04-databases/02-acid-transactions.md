# ACID Transactions

> **TL;DR** — A transaction groups reads and writes into one unit that either commits entirely or leaves no trace. ACID (Atomicity, Consistency, Isolation, Durability) is the set of guarantees that lets you write correct code against a database that is used concurrently and can crash at any moment.

## How it works

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';  -- crash here → as if nothing ran
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;  -- returns only once the commit record is durable in the log
```

- **Atomicity:** all or nothing. On `ROLLBACK`, error or crash, partial work is undone (InnoDB/Oracle use undo logs; Postgres marks the transaction aborted so its row versions stay invisible). It's about failure, not concurrency.
- **Consistency:** each transaction moves the DB from one valid state to another. The DB enforces *declared* invariants (PK, FK, UNIQUE, CHECK, NOT NULL); the rest ("money is never created or destroyed") is the app's job. Not CAP's C, which means linearizability.
- **Isolation:** concurrent transactions behave as if they ran one after another, to a configurable degree (the isolation level). Weaker levels allow anomalies in exchange for less blocking.
- **Durability:** once `COMMIT` returns, the data survives a crash. The commit record is appended to the write-ahead log (WAL / redo log / transaction log) and fsync'd *before* the ack; data pages are written later. Crash recovery replays the log (redo) and discards unfinished transactions.

## Isolation levels

| Level | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew |
|---|---|---|---|---|---|
| Read Uncommitted | possible | possible | possible | possible | possible |
| Read Committed | — | possible | possible | possible | possible |
| Repeatable Read (standard, lock-based) | — | — | possible | — | via phantoms |
| Snapshot Isolation | — | — | — | — | possible |
| Serializable | — | — | — | — | — |

- **Dirty read:** you see another transaction's uncommitted write, which may still roll back.
- **Non-repeatable read:** reading the same row twice returns different committed values.
- **Phantom:** re-running a range query returns rows another transaction inserted or deleted.
- **Lost update:** two read-modify-write cycles interleave; the second write silently overwrites the first.
- **Write skew:** two transactions read the same rows, then each updates a *different* row based on what it read; together they break an invariant. Two on-call doctors each see "2 on call" and each goes off call, leaving 0.

Engines don't follow the standard's names:

| Engine | Default | What the levels really are |
|---|---|---|
| PostgreSQL | Read Committed | RR = snapshot isolation (no phantoms; a concurrent update raises a serialization error); Serializable = SSI; RU behaves as RC |
| MySQL InnoDB | Repeatable Read | plain SELECTs read a snapshot; locking reads, UPDATE and DELETE take **next-key locks** (row + gap) that block phantom inserts; read-then-write via a plain SELECT can still lose updates |
| SQL Server | Read Committed (locking) | `READ_COMMITTED_SNAPSHOT ON` → per-statement row versioning (default in Azure SQL Database); `SNAPSHOT` = SI; RR is lock-based; Serializable uses key-range locks |
| Oracle | Read Committed | "Serializable" is really snapshot isolation (write skew possible); no RU or RR |

## Advanced

- **Two-phase locking (2PL):** acquire locks as you go, release them all at commit (strict 2PL). Reads take shared (S) locks, writes exclusive (X); only S+S are compatible, so readers block writers and vice versa. Key-range locks stop phantoms. SQL Server escalates row/page locks to a table lock once a statement holds ~5,000 locks on one object.
- **MVCC:** each write creates a new row version; readers see a snapshot (per statement in RC, per transaction in RR/SI), so **readers never block writers**, and only writers block writers on the same row. Old versions need garbage collection: Postgres keeps them in the table as dead tuples (`VACUUM`/autovacuum, plus freezing to prevent transaction-ID wraparound); InnoDB and Oracle keep them in undo logs; SQL Server in the tempdb version store.
- **SSI (Serializable Snapshot Isolation):** Postgres `SERIALIZABLE` runs on snapshots, tracks read/write dependencies with non-blocking SIREAD locks, and aborts a transaction that could complete a dangerous dependency cycle (SQLSTATE `40001`). Optimistic: no blocking, but some false-positive aborts. CockroachDB defaults to Serializable.
- **Fixing lost updates:** (1) one atomic statement (`UPDATE … SET stock = stock - 1 WHERE id = @id AND stock > 0`, then check rows affected); (2) pessimistic `SELECT … FOR UPDATE` (SQL Server: `WITH (UPDLOCK)`); (3) **optimistic concurrency**: `… WHERE id = @id AND version = @v`, and 0 rows means reload and retry (SQL Server `rowversion`; EF Core concurrency tokens → `DbUpdateConcurrencyException`). Write skew needs Serializable, `FOR UPDATE` on every row the decision reads, or a constraint (Postgres exclusion constraints block overlapping bookings).
- **Deadlocks:** T1 locks A then B while T2 locks B then A. The engine detects the cycle and kills a victim (SQL Server error 1205; Postgres checks after `deadlock_timeout`). Prevent them with a consistent lock order (e.g. sorted IDs), short transactions and indexed predicates (scans lock more rows); retrying the victim is normal.
- **Durability costs:** fsync per commit is the bottleneck, so engines **group commit** (one flush for many transactions). Knobs that trade durability for speed: Postgres `synchronous_commit = off` (can lose commits from the last 3 × `wal_writer_delay`, never corrupts), MySQL `innodb_flush_log_at_trx_commit = 2`, SQL Server delayed durability. A dead disk still loses everything, hence synchronous replication (Postgres `synchronous_standby_names`, MySQL semi-sync, SQL Server AG synchronous commit).
- **Savepoints:** `SAVEPOINT s1` / `ROLLBACK TO SAVEPOINT s1` (SQL Server: `SAVE TRANSACTION`) undo part of a transaction. In Postgres any error aborts the whole transaction unless you roll back to a savepoint.
- **Long transactions hurt:** they hold locks (blocking chains) and pin old snapshots: Postgres bloat (vacuum can't clean), growing InnoDB undo history, a growing SQL Server version store. Never keep a transaction open across an HTTP call, user think time or a queue publish (Postgres: `idle_in_transaction_session_timeout`).
- **BASE vs ACID:** many distributed NoSQL stores choose Basically Available, Soft state, Eventually consistent, giving up per-operation guarantees for availability and latency; the app resolves conflicts (last-write-wins, CRDTs).
- **Across databases or services:** ACID stops at one database. Options: 2PC/XA (blocking coordinator; .NET `TransactionScope` escalates to MSDTC), sagas (local transactions + compensations), and the **transactional outbox** (write the event row in the same local transaction; a relay publishes it to RabbitMQ/Kafka) to avoid dual writes.

## Trade-offs

- ✅ No partial writes, declared invariants enforced, crash recovery built in; you reason about one transaction at a time.
- ✅ Isolation is tunable per transaction: pay for Serializable only where an invariant needs it.
- ❌ Stronger isolation means more blocking (2PL) or more aborts (SI/SSI): lower throughput and mandatory retry loops.
- ❌ Durability costs latency: an fsync per commit group, plus a network round trip for synchronous replication.
- ❌ Stops at one database; ACID across shards, regions or services needs 2PC/consensus (latency) or sagas (no isolation).

## In the wild

PostgreSQL, MySQL InnoDB, SQL Server, Oracle (single-node ACID) · Spanner, CockroachDB, YugabyteDB, TiDB (distributed ACID over consensus) · MongoDB multi-document transactions, DynamoDB `TransactWriteItems`, Cosmos DB transactional batch (one partition) · .NET: `SaveChanges()` wraps its changes in one transaction; `Database.BeginTransaction()`, `TransactionScope` · Node: `pg` (one checked-out client per transaction), Prisma `$transaction`, Knex/TypeORM/Sequelize `transaction()` · RabbitMQ channel transactions exist but are slow; use publisher confirms plus an outbox.

## Gotchas

- **Read Committed (the Postgres, SQL Server and Oracle default) allows lost updates and write skew:** "SELECT, decide in C#, UPDATE" is a race unless you lock, version-check or use one atomic UPDATE.
- **Same name, different behaviour:** Oracle's Serializable and Postgres's Repeatable Read are snapshot isolation; MySQL's Repeatable Read is neither standard RR nor SI. Test the anomaly you care about (Hermitage-style), not the level's name.
- **Retries are part of the design:** serialization failures (`40001`) and deadlock victims (1205) are expected; retry the whole transaction, idempotently. With EF Core's retrying execution strategy, a user transaction must run inside `CreateExecutionStrategy().Execute(...)`.
- **Node `pg` pool trap:** `pool.query('BEGIN')` and the next `pool.query()` can land on different connections, so there's no transaction at all. Take one client with `pool.connect()`, use it throughout, release it in `finally`.
- **Committed ≠ replicated:** with async replication a failover can lose acknowledged commits, and a replica read right after commit can miss your write.
- **ACID's C, CAP's C and "eventual consistency"** are three different meanings of one word.

## Related

[Two-Phase Commit (2PC)](../07-distributed-coordination/03-two-phase-commit.md) · [Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [Replication](04-replication.md) · [CAP Theorem](05-cap-theorem.md) · [SQL vs NoSQL](01-sql-vs-nosql.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Distributed Locks](../07-distributed-coordination/01-distributed-locks.md)

## Read more

- [Transaction Isolation (PostgreSQL docs)](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Hermitage: testing transaction isolation levels across databases (Martin Kleppmann)](https://github.com/ept/hermitage)
- [A Critique of ANSI SQL Isolation Levels (Berenson et al., 1995)](https://arxiv.org/abs/cs/0701157)

<!-- nav -->

---

[← SQL vs NoSQL](01-sql-vs-nosql.md) · [Index](../README.md) · [Database Indexes →](03-database-indexes.md)
