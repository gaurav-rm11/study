# Database Indexes

> **TL;DR** — An index is a separate sorted (or hashed) structure that maps column values to row locations, so the database jumps to matching rows instead of scanning the whole table: O(log n) instead of O(n) reads, paid for on every write.

## How it works

```text
              B+tree on users(email)
                   [ g | p ]                ← root: always cached
          ┌────────────┼────────────┐
       [a … f]      [g … o]      [p … z]    ← internal pages: hundreds of keys each
          │            │            │
   leaf ⇄ leaf ⇄ leaf ⇄ leaf ⇄ leaf ⇄ leaf   ← sorted keys + row pointers, linked for range scans
```

- **B+tree** (the default almost everywhere): a balanced tree of disk pages; internal nodes only route, leaves hold sorted keys + row pointers and are linked. Serves `=`, ranges (`<`, `BETWEEN`), `ORDER BY`, `MIN/MAX` and prefix `LIKE 'abc%'`.
- **Hash index:** O(1) equality only — no ranges, no ordering.
- **The cost:** every INSERT/UPDATE/DELETE also maintains each affected index → slower writes, more storage, more pages competing for the buffer pool.
- **The planner decides:** from statistics (row counts, value distribution) the optimizer picks an index **seek** or a full **scan** — check with `EXPLAIN ANALYZE` (Postgres/MySQL) or the actual execution plan (SQL Server).

## Index types

| Type | What it does | Example |
|---|---|---|
| Clustered | table rows stored in key order (one per table) | InnoDB primary key, SQL Server clustered index |
| Secondary / non-clustered | separate structure pointing at rows | most `CREATE INDEX` |
| Composite | several columns, ordered left to right | `(tenant_id, created_at)` |
| Covering / INCLUDE | holds every column the query needs | `INCLUDE (status, total)` |
| Partial / filtered | indexes only rows matching a predicate | `WHERE status = 'active'` |
| Expression / functional | indexes a computed value | `lower(email)` |
| Unique | enforces uniqueness | email, natural keys |
| Full-text (inverted) | word → documents | Postgres GIN + tsvector, SQL Server full-text, Elasticsearch |
| Spatial | 2-D ranges / nearest neighbour | GiST/R-tree (PostGIS), SQL Server spatial |
| BRIN | min/max per block range — tiny, for huge naturally ordered tables | Postgres time-series |
| Bitmap | low-cardinality columns in analytics | Oracle (Postgres only builds bitmaps at query time) |
| Columnstore | column-wise compressed storage for scans/aggregates | SQL Server columnstore |

## Advanced

- **B+tree depth:** fan-out of hundreds of keys per page (8 KB in Postgres/SQL Server, 16 KB in InnoDB) means 3–4 levels cover billions of rows; upper levels stay cached, so a lookup costs ~1–2 disk reads.
- **Clustered vs heap:** InnoDB and SQL Server (with a clustered index) store rows inside the clustered B+tree; secondary indexes store the clustering key, so a secondary lookup is a second traversal ("key lookup") → keep clustering keys narrow. Postgres tables are heaps: every index is secondary and points at a tuple id; HOT updates skip index writes when no indexed column changes.
- **Key choice:** random UUIDv4 keys insert at random leaves → page splits, fragmentation, poor cache locality; sequential keys (identity, UUIDv7, `NEWSEQUENTIALID()`) append on the right. Extreme insert rates on one ever-increasing key can cause last-page contention (SQL Server `OPTIMIZE_FOR_SEQUENTIAL_KEY`).
- **Leftmost-prefix rule:** `(a, b, c)` serves `a`, `a,b`, `a,b,c` and `ORDER BY a, b` — not `b` or `c` alone (except skip scans: Oracle, MySQL 8.0.13+, Postgres 18). Put equality columns first, then the range/sort column: `WHERE tenant_id = ? AND created_at > ? ORDER BY created_at` → `(tenant_id, created_at)`.
- **Covering & index-only scans:** if the index holds every needed column, the table isn't touched; `INCLUDE` adds non-key columns to leaves without changing order. Postgres index-only scans also need an up-to-date visibility map (VACUUM).
- **Selectivity:** an index wins when the predicate matches a small fraction of rows; beyond that, one sequential pass beats thousands of random row lookups and the planner (correctly) scans.
- **Sargability killers:** a function on the column (`YEAR(created_at) = 2024` → use a range or an expression index), leading wildcards (`LIKE '%abc'` → trigram/full-text), implicit conversions (NVARCHAR parameter vs VARCHAR column in SQL Server), `OR` across different columns (→ `UNION` or bitmap OR), `!=`.
- **Statistics & plan caching:** stale stats → bad cardinality estimates → wrong plans. SQL Server **parameter sniffing** reuses a plan built for the first parameter value; with skewed data, fix via `OPTION (RECOMPILE)`, `OPTIMIZE FOR` or Query Store plan forcing.
- **LSM-tree engines** (Cassandra, RocksDB, ScyllaDB, MyRocks): writes hit a memtable + WAL, flush to immutable sorted SSTables, merged by compaction; reads check per-SSTable **Bloom filters** to skip files. Write-optimised at the cost of read and space amplification.
- **Over-indexing:** each index adds writes, WAL/replication volume and memory pressure. Drop unused and duplicate-prefix indexes (`pg_stat_user_indexes`, `sys.dm_db_index_usage_stats`).
- **Maintenance:** bloat/fragmentation → `REINDEX`/`VACUUM` (Postgres), `ALTER INDEX … REBUILD/REORGANIZE` (SQL Server), `OPTIMIZE TABLE` (MySQL).
- **Online builds:** a plain `CREATE INDEX` can block writes on a big table → `CREATE INDEX CONCURRENTLY` (Postgres), `WITH (ONLINE = ON)` (SQL Server Enterprise / Azure SQL), InnoDB online DDL (`ALGORITHM=INPLACE, LOCK=NONE`).
- **Keyset pagination:** `WHERE (created_at, id) < (?, ?) ORDER BY created_at DESC, id DESC LIMIT 20` walks the index directly; `OFFSET 100000` reads and discards 100k rows.
- **Sharded / distributed DBs:** local secondary indexes live per shard (non-shard-key queries scatter to all shards); global secondary indexes are partitioned by the indexed value and usually updated asynchronously (DynamoDB GSIs are eventually consistent).

## Trade-offs

- ✅ Orders-of-magnitude faster lookups, ranges, sorts and joins; unique indexes enforce integrity.
- ❌ Slower writes, more storage and memory, more WAL/replication traffic, ongoing maintenance.
- ❌ Too many or wrong indexes waste buffer pool and confuse the planner; revisit them as query patterns change.

## In the wild

Postgres (B-tree, Hash, GIN, GiST, SP-GiST, BRIN; `EXPLAIN (ANALYZE, BUFFERS)`) · MySQL InnoDB (clustered PK, invisible indexes, FULLTEXT) · SQL Server (clustered/non-clustered, INCLUDE, filtered, columnstore, missing-index DMVs, Query Store) · MongoDB (compound, multikey, TTL, 2dsphere, text) · Elasticsearch/Lucene (inverted index) · Cassandra/RocksDB (LSM + Bloom filters).

## Gotchas

- An index on `(a, b)` doesn't help `WHERE b = ?`; two single-column indexes are usually worse than one well-ordered composite.
- Adding an index isn't free on write-heavy tables — weigh read gains against write cost.
- Low-cardinality columns (`is_active`) rarely help alone — fold them into a composite or use a partial/filtered index.
- `SELECT *` defeats covering indexes; fetch only the columns you need.
- Postgres and SQL Server don't auto-index foreign-key columns (MySQL InnoDB does) → slow joins and cascading deletes.
- Always confirm with the actual plan — the optimizer may ignore your index (stats, selectivity, type mismatch).

## Related

[SQL vs NoSQL](01-sql-vs-nosql.md) · [ACID Transactions](02-acid-transactions.md) · [Database Sharding](06-database-sharding.md) · [Geohashing](09-geohashing.md) · [Bloom Filters](../05-caching/04-bloom-filters.md) · [Caching](../05-caching/01-caching.md)

## Read more

- [Use The Index, Luke — SQL indexing and tuning for developers](https://use-the-index-luke.com/)
- [PostgreSQL docs — Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [SQL Server index architecture and design guide](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide)

<!-- nav -->

---

[← ACID Transactions](02-acid-transactions.md) · [Index](../README.md) · [Replication →](04-replication.md)
