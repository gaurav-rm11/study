# Stateful vs Stateless

> **TL;DR** — A stateless service keeps no per-client state in instance memory between requests, so any instance can serve any request; a stateful one binds state to a specific instance, so each request must reach that instance. Stateless app tiers make scaling, failover and deploys cheap — the state doesn't disappear, it moves to the client or a shared store.

## How it works

```text
Stateless — any instance can serve; state lives in the shared store
  client ──► LB ──┬──► A ──┐
                  ├──► B ──┼──► Redis / DB
                  └──► C ──┘
Stateful — the request must reach the instance that owns the key
  client ──► router: owner(room 142) ──► B    (room 142's state is in B's memory)
```

- **Stateless:** the request carries what's needed (token, IDs, cursor) or the instance loads it from a shared store; instance memory holds only request-scoped data and disposable caches.
- **Stateful:** state outlives the request on one instance — in-memory sessions, open TCP/WebSocket connections, game and chat rooms, a database's data files, a stream operator's windows and aggregates.
- **Litmus test:** kill an instance mid-traffic. If clients just retry elsewhere with no logouts or lost data, the tier was stateless.

## Comparison

| | Stateless | Stateful |
|---|---|---|
| State lives in | client (token/cookie) or shared store (Redis, DB) | instance memory / local disk |
| Scaling | add instances; autoscale on CPU/RPS | partition the state; rebalance on every membership change |
| Load balancing | any algorithm, any instance | sticky sessions or route-by-key to the owner |
| Failover | retry on another instance; nothing lost | state lost unless replicated/checkpointed; recovery takes time |
| Deploys | rolling / blue-green; kill instances freely | drain, hand off or persist state first; ordered rollouts |
| Latency | +1 hop to the store per request | memory-local reads, fewest hops |

## Session state options

| Option | How | Upside | Downside |
|---|---|---|---|
| Sticky sessions | LB pins a client to one instance (cookie or IP hash) | no app changes | skewed load; sessions die with the instance; scale-in logs users out |
| Session replication | instances copy sessions to peers (app-server clustering) | survives a node loss | all-to-all chatter; only practical for small clusters |
| Central store | Redis / SQL holds sessions; instances stay stateless | any instance serves; survives deploys | extra hop; the store must be highly available |
| Client-side token | signed (optionally encrypted) cookie or JWT carries the state | no lookup; scales trivially | ~4 KB cookie limit; revocation needs server state; claims stale until expiry |

Default to a central store or tokens. Classic ASP.NET's default `InProc` session mode is the sticky-session trap.

## Advanced

- **"Stateless" moves state, it doesn't remove it:** replication, consistency and failover become the data tier's job. The goal is a few purpose-built stateful systems (DB, Redis, Kafka) behind many disposable app nodes — 12-factor's "processes are stateless and share-nothing", which calls sticky sessions a violation.
- **Scaling stateful services = partition ownership:** split state by key (user, room, tenant); each partition has one owner, or a leader plus followers (Kafka partition leaders, a Raft group per CockroachDB range). Route each request to its owner via consistent hashing (Dynamo, Cassandra), a partition map in a coordinator (etcd, ZooKeeper) or an actor runtime.
- **Actors:** Orleans *virtual actors* (grains) are activated on demand on some silo, located via a grain directory, and every call to a grain ID routes to its single activation; idle grains are deactivated and reload their persisted state on the next call. Akka / Akka.NET Cluster Sharding does the same for entities, by shard ID.
- **Rebalancing and handoff:** on scale-out or failure, partitions move — the new owner reloads from durable storage, receives a state transfer from the old owner, or a replica is promoted. Fence with epochs/tokens so a paused old owner can't keep writing (split brain); consistent hashing limits movement to ~1/N of keys.
- **Kubernetes:** Deployment pods are interchangeable (random names, any order). StatefulSet pods get stable identities (`db-0`, `db-1`, each with DNS via a headless Service), their own PersistentVolumeClaim that follows the pod across reschedules, and ordered create/terminate/rolling updates — use for databases, brokers, search clusters.
- **WebSockets make a gateway stateful:** each connection lives on one node. To reach a user connected elsewhere, publish through a backplane every node subscribes to (Redis pub/sub, NATS, RabbitMQ) or keep a `userId → node` registry and route directly.
- **Stream processors are stateful by design:** Flink keeps keyed state (windows, joins, aggregates) local, in memory or RocksDB, and periodically checkpoints it plus source offsets to durable storage (S3/HDFS); after a failure it restores the last checkpoint and replays from those offsets. Kafka Streams backs local state stores with compacted changelog topics.
- **Hidden state in "stateless" services:** in-process caches (`IMemoryCache`, a module-level `Map`) diverge between instances, so answers flip depending on which instance serves the request. Same trap: in-memory rate-limit counters, in-process locks, timers/cron (run once per instance), files on local disk. Fix with a shared store, pub/sub invalidation or short TTLs.
- **Protocols:** HTTP is stateless — each request is self-contained; cookies/tokens layer state on top. TCP is stateful — both ends track sequence numbers and windows, so a live connection can't move between servers, and NAT, stateful firewalls and L4 LBs keep connection-tracking tables.

## Trade-offs

- ✅ Stateless: retries, idempotency and autoscaling get simple — any instance takes the retry, idempotency keys live in the shared store, nodes come and go without migration; spot instances and serverless become viable.
- ❌ Stateless: every request pays a store round-trip or ships a bigger token; the shared store becomes the bottleneck and SPOF you now have to scale.
- ✅ Stateful: compute next to its data — no per-request fetch, in-memory aggregation and write batching; the natural fit for connections, live rooms, games, collaborative editing.
- ❌ Stateful: you own partitioning, routing, rebalancing, replication and recovery; hot partitions; slower, riskier deploys.

## In the wild

ASP.NET Core session over `IDistributedCache` (Redis, SQL Server) · Node: `express-session` + `connect-redis` · SignalR Redis backplane / Azure SignalR Service · Socket.IO Redis adapter · Orleans (Halo's cloud services), Akka.NET Cluster Sharding · Kubernetes StatefulSets · Flink checkpoints, Kafka Streams state stores · AWS Lambda, Azure Functions, Cloud Run as stateless compute.

## Gotchas

- "JWTs make us stateless" — until you need revocation: a denylist, or short expiry plus refresh tokens, is server state again.
- In containers, ASP.NET Core Data Protection keys (they encrypt auth cookies and antiforgery tokens) default to the container's local storage: other instances can't decrypt them and a redeploy logs everyone out. Persist keys to Redis or blob storage.
- ASP.NET Core session is non-locking — when two concurrent requests both write the session, the last write wins.
- Container local disk is ephemeral; uploads or SQLite files written there vanish on reschedule.
- WebSocket deploys drop every connection on the drained node — clients need jittered exponential backoff or they reconnect as a thundering herd.
- Interview trap: "make it stateless" is no answer for the DB, cache or chat gateway — say where the state lives and how it's partitioned, replicated and recovered.

## Related

[Scalability](01-scalability.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [JWTs](../02-apis-and-communication/04-jwts.md) · [Long Polling vs WebSockets](../02-apis-and-communication/06-long-polling-vs-websockets.md) · [Consistent Hashing](../04-databases/07-consistent-hashing.md) · [Caching](../05-caching/01-caching.md) · [Batch vs Stream Processing](../06-async-and-events/03-batch-vs-stream-processing.md)

## Read more

- [The Twelve-Factor App — VI. Processes](https://12factor.net/processes)
- [StatefulSets — Kubernetes docs](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [Orleans overview — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/orleans/overview)

<!-- nav -->

---

[← SPOF](03-spof.md) · [Index](../README.md) · [Concurrency vs Parallelism →](05-concurrency-vs-parallelism.md)
