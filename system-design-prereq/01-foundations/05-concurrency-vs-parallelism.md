# Concurrency vs Parallelism

> **TL;DR** — Concurrency is structuring a program as many tasks whose lifetimes overlap, interleaved so one task's wait doesn't stall the rest; parallelism is executing computations at the same instant on multiple cores or machines. Concurrency lets one Node process juggle 10,000 open sockets; parallelism makes a CPU-heavy job finish sooner on 8 cores.

## How it works

```text
Concurrency on 1 core — tasks interleave; one runs at any instant
  core 1 │ A A B C A B B C A
Parallelism on 3 cores — tasks run at the same instant
  core 1 │ A A A
  core 2 │ B B B
  core 3 │ C C C
```

- **Rob Pike's framing:** "Concurrency is about *dealing with* lots of things at once. Parallelism is about *doing* lots of things at once." A concurrent program *can* run in parallel when cores are available; it doesn't have to.
- **Concurrency without parallelism:** one core time-slicing threads, or Node's single JS thread running callbacks for thousands of in-flight requests.
- **Parallelism without concurrency:** SIMD — one instruction stream applies the same operation to many data lanes at once (8 floats per AVX2 instruction).
- **Decision rule:** I/O-bound (waiting on DB, HTTP, disk) → async/non-blocking I/O, so a wait costs no thread. CPU-bound (hashing, compression, image rendering, huge JSON) → threads or processes, about one per core; more only adds context switches.

## Comparison

| | Concurrency | Parallelism |
|---|---|---|
| About | structure — many tasks in progress | execution — many computations at the same instant |
| Needs multiple cores | no | yes (or multiple machines) |
| Wins back | time spent waiting (I/O-bound) | time spent computing (CPU-bound) |
| Success metric | tasks in flight, throughput, responsiveness | wall-clock speed-up |
| Tools | async/await, event loops, coroutines, channels, actors | thread pools, processes, `Parallel.For`, PLINQ, SIMD, GPUs, Spark |
| Limited by | memory per task; downstream capacity (DB pool) | serial fraction (Amdahl), contention, coordination overhead |

## Models

| Model | How tasks coordinate | Examples |
|---|---|---|
| Threads + shared memory + locks | threads mutate shared objects under locks | Java/C# threads, pthreads |
| Event loop / async-await | few threads; the OS signals when I/O is ready (epoll, kqueue) or complete (IOCP); continuations resume on the loop (Node) or a pool thread (ASP.NET Core) | Node (libuv), .NET `Task`, Python asyncio, Nginx |
| Actors | each actor owns private state and handles one message at a time from its mailbox | Erlang/Elixir (BEAM), Akka / Akka.NET, Orleans |
| CSP / channels | independent processes pass values over channels — "share memory by communicating" | Go goroutines + channels, .NET `System.Threading.Channels` |
| Data parallelism | same operation over chunks of the data | SIMD (`Vector<T>`), GPUs (CUDA), `Parallel.For`, PLINQ, MapReduce/Spark |
| Task parallelism | different tasks on different cores at once | `Task.WhenAll` over `Task.Run` jobs, pipeline stages |

## Advanced

- **Race condition vs data race:** a *data race* is unsynchronized concurrent access to one memory location with at least one write (undefined behavior in C/C++). A *race condition* is any timing-dependent bug — check-then-act, read-modify-write (`count++` is load-add-store) — and survives per-statement locking: two requests both read `stock = 1` and both sell.
- **Deadlock needs all four Coffman conditions:** mutual exclusion, hold-and-wait, no preemption, circular wait. Break one: global lock ordering, try-lock with timeout and back off, or acquire everything up front. Databases detect wait-for cycles and abort a victim (SQL Server error 1205, Postgres "deadlock detected") — retry those transactions.
- **Livelock, starvation, priority inversion:** livelock — threads keep yielding and retrying without progress (fix: randomized backoff). Starvation — a thread never gets the resource, e.g., writers behind a stream of readers (fix: fair locks). Priority inversion — a low-priority thread holds a lock a high-priority one needs while medium-priority work preempts it (Mars Pathfinder, 1997; fix: priority inheritance).
- **Primitives:** mutex (one owner; C# `lock`); semaphore (N permits — `SemaphoreSlim(10)` caps concurrent calls to a dependency); reader-writer lock (many readers or one writer); atomics/CAS (`Interlocked.CompareExchange`: swap only if the value is still what you read, else retry); lock-free structures — CAS loops that guarantee system-wide progress but must handle ABA.
- **Memory visibility:** compilers and CPUs reorder operations and cores buffer writes, so an unsynchronized reader can see stale or out-of-order values. Only the memory model's *happens-before* edges guarantee visibility — unlock → later lock of the same mutex, volatile/atomic write → later read, thread start/join. `volatile` (Java, C#) gives visibility and ordering, not atomicity; x86's strong ordering hides bugs that surface on ARM (Graviton, Apple Silicon).
- **Amdahl's law (fixed problem size):** `S(N) = 1 / ((1 − p) + p/N)`, p = parallel fraction, N = cores; at p = 0.95 the speed-up never exceeds 20× — locks, coordination and serial I/O all count as serial. **Gustafson's law (problem grows with N):** `S(N) = s + (1 − s)·N`, s = serial share of the run on N cores — why scale-out works when data grows with the cluster.
- **Context switches:** an OS thread switch costs microseconds (kernel entry, register save, scheduler) plus cold caches and TLB afterwards, and each thread reserves an MB-scale stack — why thread-per-connection collapses at 10k connections (C10k) and event loops don't.
- **.NET thread-pool starvation:** sync-over-async (`.Result`, `.Wait()`, `GetAwaiter().GetResult()`) parks a pool thread waiting on a continuation that needs another pool thread. Under load every thread blocks, the queue grows and the pool adds threads only gradually → timeouts at low CPU (`dotnet-counters`: thread-pool queue length). Fix: async all the way. Under a UI or classic ASP.NET `SynchronizationContext` the same code deadlocks outright — hence `ConfigureAwait(false)` in libraries.
- **Blocking Node's event loop:** one JS thread per process, so a CPU-heavy callback (huge `JSON.parse`, `crypto.pbkdf2Sync`, a catastrophically backtracking regex) stalls every request in it. libuv's pool (4 threads by default, `UV_THREADPOOL_SIZE`) runs fs, `dns.lookup`, async crypto and zlib; sockets use epoll/kqueue/IOCP directly. Move CPU work to `worker_threads` or another service, run one process per core (`cluster`, more pods), alert on event-loop lag.
- **Green / virtual threads (M:N):** many cheap user-space threads multiplexed onto few OS threads, so blocking-style code scales like async. Go runs goroutines (KB-sized growable stacks) on at most `GOMAXPROCS` OS threads at once, with work stealing and preemption. Java 21 virtual threads unmount from their carrier thread when they block (pinning inside `synchronized` fixed in JDK 24). .NET and Node use stackless async/await state machines instead.
- **Optimistic vs pessimistic concurrency control — the bridge to databases:** pessimistic locks first (`SELECT … FOR UPDATE`, SQL Server `UPDLOCK`) and makes others wait — best under high contention. Optimistic runs `UPDATE … SET …, version = version + 1 WHERE id = @id AND version = @v`; 0 rows = conflict → reload and retry — best under low contention (EF Core `[Timestamp]` → `DbUpdateConcurrencyException`; HTTP `ETag` + `If-Match`). Mutex vs CAS, at row level.

## Trade-offs

- ✅ Async/event-loop concurrency: tens of thousands of in-flight I/O operations on a handful of threads.
- ❌ Async: one blocking call stalls throughput; async spreads up every caller ("function coloring"); harder stack traces and debugging.
- ✅ Parallelism: cuts wall-clock time of CPU-bound work, up to the Amdahl ceiling.
- ❌ Parallelism: coordination, contention and false sharing (cores writing different variables on one 64-byte cache line) can make small workloads slower than serial; bugs are non-deterministic.
- ✅ Shared-nothing models (actors, channels, processes): far fewer races; scale out across machines.
- ❌ Message passing: copy/serialization cost, mailbox backpressure, no cheap atomic update across several actors.

## In the wild

.NET ThreadPool + `async`/`await`, `Parallel.ForEachAsync`, `Interlocked` · Node `worker_threads`, `cluster`, Piscina · Java 21 virtual threads · Erlang/Elixir on BEAM (WhatsApp, Discord, RabbitMQ) · Nginx: event-loop worker processes, usually one per core · Redis: single-threaded command execution (I/O threads since 6.0) · Postgres: process per connection; MySQL: thread per connection · RabbitMQ prefetch caps in-flight messages per consumer · Spark, CUDA for data parallelism.

## Gotchas

- `async` ≠ parallel: awaiting calls one by one is sequential. Start them together and `Task.WhenAll` / `Promise.all` — then cap fan-out (`SemaphoreSlim`, `p-limit`) so you don't exhaust the DB connection pool.
- `Parallel.ForEach` with an `async` lambda compiles to `async void`: it returns before the work finishes and an exception can crash the process. Use `Parallel.ForEachAsync` (.NET 6+).
- `Task.Run` around blocking I/O in ASP.NET Core just moves the block to another pool thread; it adds no capacity.
- No `await` inside a C# `lock` — use `SemaphoreSlim.WaitAsync` with `try/finally`.
- `lock` and `SemaphoreSlim` protect one process. With N instances, coordinate through the database (row locks, optimistic versions) or a distributed lock.

## Related

[Stateful vs Stateless](04-stateful-vs-stateless.md) · [Scalability](01-scalability.md) · [ACID Transactions](../04-databases/02-acid-transactions.md) · [Distributed Locks](../07-distributed-coordination/01-distributed-locks.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [Batch vs Stream Processing](../06-async-and-events/03-batch-vs-stream-processing.md)

## Read more

- [Concurrency is not Parallelism — Rob Pike (Go blog)](https://go.dev/blog/waza-talk)
- [Don't Block the Event Loop (or the Worker Pool) — Node.js](https://nodejs.org/en/learn/asynchronous-work/dont-block-the-event-loop)
- [ASP.NET Core async guidance — David Fowler](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md)

<!-- nav -->

---

[← Stateful vs Stateless](04-stateful-vs-stateless.md) · [Index](../README.md) · [APIs →](../02-apis-and-communication/01-apis.md)
