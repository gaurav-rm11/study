# SQL vs NoSQL

> **TL;DR** — Relational (SQL) databases store normalised tables with an enforced schema, joins and ACID transactions. "NoSQL" is an umbrella for non-relational stores (key-value, document, wide-column, graph…) that give up some of that generality for a data model or scaling property that fits specific access patterns.

## How it works

```text
Relational: normalise + join at read time       Document: denormalise, one read
customers(id, name)                             { _id: 1001,
orders(id, customer_id → customers.id)            customer: { id: 7, name: "Asha" },
order_items(order_id → orders.id, sku, qty)       items: [ { sku: "A1", qty: 2 } ] }
```

- **Relational:** typed columns in tables. The DB enforces the schema on write (types, NOT NULL, FK, UNIQUE, CHECK). Each fact is stored once (normalised) and recombined with joins in declarative SQL. Transactions are multi-row, multi-table ACID. PostgreSQL, MySQL, SQL Server, Oracle.
- **NoSQL:** data is shaped around how it's read: an aggregate stored together (document), a value fetched by key, rows grouped into partitions by key. Joins are rare or absent, so you duplicate data instead. Partitioning and replication are usually built in.

## Families

| Family | Model | Strong at | Weak at | Examples |
|---|---|---|---|---|
| Key-value | opaque value by key | O(1) get/put: sessions, carts, caches | querying by anything but the key | Redis, DynamoDB, Memcached |
| Document | JSON/BSON documents, nested, secondary indexes | aggregates read and written as a unit, varying shapes | many-to-many, cross-document joins | MongoDB, Couchbase, Firestore, Cosmos DB |
| Wide-column | partition key → rows sorted by clustering key, sparse columns | huge write throughput, time-ordered data per key | ad-hoc queries, joins, aggregates | Cassandra, ScyllaDB, HBase, Bigtable |
| Graph | nodes + edges with properties | multi-hop traversals (friends-of-friends, fraud rings) | bulk analytics; graphs don't shard cleanly | Neo4j, Amazon Neptune |
| Time-series | (series, timestamp) → value | append-heavy metrics, downsampling, retention | random updates | InfluxDB, TimescaleDB, Prometheus |
| Search | inverted index | full-text, relevance ranking, facets | being the source of truth | Elasticsearch, OpenSearch, Solr |
| Vector | embeddings + approximate-nearest-neighbour index (HNSW, IVF) | semantic similarity, RAG | everything else (pair with a primary DB) | pgvector, Pinecone, Milvus, Qdrant |

| | Relational | NoSQL (typical) |
|---|---|---|
| Schema | on write, enforced by the DB; migrations | on read; enforced by your code |
| Modelling | normalise, join at query time | denormalise per access pattern; duplicate data |
| Queries | ad-hoc SQL, joins, aggregates | by key/partition; queries planned up front |
| Transactions | multi-row, multi-table ACID | often single-item/partition; multi-item limited or costlier |
| Consistency | strong on the primary | eventual or tunable |
| Scaling | up + read replicas; sharding is manual | partitioning + replication built in |

## Advanced

- **Query-first modelling:** with NoSQL you list the access patterns *before* designing keys, so each pattern is a single request. DynamoDB **single-table design** stores several entity types under generic keys (`PK=CUSTOMER#7`, `SK=ORDER#2025-01-09`), so one Query returns a customer plus their orders; GSIs serve the other patterns. Cassandra uses **one table per query** and writes the same data to each. Relational models the data once; queries come later.
- **Scaling:** SQL scales up, then adds read replicas, but every write still goes to one primary. Sharding is done by the app or by a layer such as Vitess (MySQL) or Citus (Postgres), and cross-shard joins and transactions get hard. Cassandra, DynamoDB and sharded MongoDB partition by key and replicate automatically.
- **Consistency:** a relational primary gives read-your-writes; its async replicas are eventually consistent. Cassandra is **tunable** per query (`ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`); R + W > N makes read and write quorums overlap. DynamoDB reads are eventually consistent by default; `ConsistentRead=true` makes them strong at twice the read cost (not available on GSIs). **BASE** (Basically Available, Soft state, Eventually consistent) names the trade many NoSQL stores make for availability and latency.
- **The line has blurred:** MongoDB has multi-document ACID transactions (4.0 for replica sets, 4.2 for sharded clusters), and DynamoDB has `TransactWriteItems` / `TransactGetItems` (up to 100 items). Conversely, SQL stores hold flexible documents: Postgres JSONB (binary, GIN-indexable), MySQL `JSON`, SQL Server's JSON functions (`JSON_VALUE`, `OPENJSON`).
- **Distributed SQL / NewSQL:** SQL and ACID on top of automatic sharding and consensus (Raft/Paxos) replication: Google Spanner (TrueTime), CockroachDB, YugabyteDB, TiDB. The price is a consensus round trip on every write (painful across regions), plus cost and operational complexity.
- **Schema-on-write vs schema-on-read:** "schemaless" means the schema lives in your code, and every reader must handle every shape ever written. Version your documents (`schemaVersion`), migrate lazily on read or by backfill, and validate on write (MongoDB `$jsonSchema`).
- **Storage engines:** a **B-tree** (Postgres, InnoDB, SQL Server, MongoDB's WiredTiger) updates pages in place: read-optimised and predictable. An **LSM-tree** (Cassandra, ScyllaDB, HBase, RocksDB) appends to an in-memory memtable, flushes it as immutable sorted SSTables and merges them by **compaction**: write-optimised (sequential I/O). LSM pays **read amplification** (a read may check several SSTables; Bloom filters help) and **space amplification** (stale versions and tombstones linger until compaction). Both engines pay **write amplification**: B-trees rewrite whole pages, LSM-trees rewrite data in every compaction. The compaction strategy (size-tiered vs leveled) trades write cost against read and space cost.
- **OLTP vs OLAP:** OLTP is many small reads and writes by key, on row storage. OLAP is scans and aggregates over a few columns of billions of rows, so it uses **columnar** storage with compression and vectorised execution: ClickHouse, BigQuery, Snowflake, Redshift, DuckDB, SQL Server columnstore. This is a separate axis from SQL vs NoSQL; warehouses speak SQL. Feed them by ETL or CDC.
- **Polyglot persistence:** use one store per job (Postgres as source of truth, Redis for cache and sessions, Elasticsearch for search, ClickHouse for analytics) and keep them in sync with CDC or an outbox. Cost: more systems to run, sync lag, cross-store consistency.

## When to use which

- **Default to relational** (Postgres, SQL Server, MySQL) for relational data, ad-hoc queries and reports, and invariants that span entities (payments, inventory, bookings). One well-indexed primary plus replicas and a cache goes further than most systems need.
- **Key-value / wide-column** (DynamoDB, Cassandra) when the access patterns are simple, key-based and known up front, and the scale or write rate is beyond one primary, or you need multi-region active-active writes.
- **Document** (MongoDB) for self-contained aggregates with varying shapes that are read and written as a unit: catalogues, CMS content, user profiles.
- **Specialised models** (graph, search, time-series, vector) usually run *alongside* a relational source of truth.
- **In an interview,** start with "Postgres first", then justify any switch with an access pattern and a scale reason, e.g. append-only events at a write rate one primary can't absorb, always read by device ID → Cassandra.

## In the wild

Amazon's Dynamo paper inspired DynamoDB, Cassandra and Riak · Discord moved messages from MongoDB → Cassandra → ScyllaDB · Facebook's social graph lives in sharded MySQL behind the TAO cache · Vitess (built at YouTube) shards MySQL at YouTube and Slack · Netflix runs Cassandra at scale · Azure Cosmos DB (multi-model, five consistency levels) · .NET: EF Core providers for SQL Server, Postgres (Npgsql), MySQL and Cosmos DB, plus `MongoDB.Driver` · Node: `pg`, `mysql2`, Prisma, Mongoose.

## Gotchas

- **"Schemaless" is a myth:** the schema moves into application code, where the DB can't check it.
- **"SQL doesn't scale":** Facebook, Shopify and GitHub run huge MySQL fleets. The hard part is sharding writes, not SQL.
- **"NoSQL is faster":** an indexed primary-key lookup in Postgres is also sub-millisecond when cached. Speed comes from the access pattern (one partition, one request, no join), not the label.
- **"NoSQL means no transactions and always-eventual reads":** false for MongoDB, DynamoDB and Cosmos DB today. Check each operation's guarantees.
- **Choosing DynamoDB/Cassandra before you know the queries:** every new query then needs a new table or index plus a backfill, or a full scan.
- **Hot partitions:** a low-cardinality or time-based partition key (e.g. today's date) funnels all writes into one partition. DynamoDB caps each partition at 1,000 WCU / 3,000 RCU.

## Related

[ACID Transactions](02-acid-transactions.md) · [CAP Theorem](05-cap-theorem.md) · [Replication](04-replication.md) · [Database Sharding](06-database-sharding.md) · [Database Indexes](03-database-indexes.md) · [Change Data Capture (CDC)](08-cdc.md) · [CQRS](../06-async-and-events/05-cqrs.md)

## Read more

- [The What, Why, and When of Single-Table Design with DynamoDB (Alex DeBrie)](https://www.alexdebrie.com/posts/dynamodb-single-table/)
- [How Discord Stores Trillions of Messages (Discord engineering)](https://discord.com/blog/how-discord-stores-trillions-of-messages)
- [Polyglot Persistence (Martin Fowler)](https://martinfowler.com/bliki/PolyglotPersistence.html)

<!-- nav -->

---

[← Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md) · [Index](../README.md) · [ACID Transactions →](02-acid-transactions.md)
