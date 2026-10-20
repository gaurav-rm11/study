# Circuit Breaker

> **TL;DR** — A circuit breaker wraps calls to a dependency and, once failures cross a threshold, *opens*: calls fail fast with an error or fallback instead of waiting on a sick dependency. That stops cascading failures and thread/connection exhaustion, and gives the dependency time to recover.

## How it works

```text
           failure threshold crossed
   CLOSED ──────────────────────────► OPEN     fail fast: error or fallback
     ▲                                │  ▲
     │         break duration elapses │  │     a probe fails
     │                                ▼  │
     └──── enough probes succeed ── HALF-OPEN  lets a few trial calls through
```

- **Closed:** calls pass through; each outcome (success, failure, slow) is recorded in a sliding window.
- **Open:** calls are rejected immediately — no network call, no timeout wait. A request that would have held a thread and a connection for 30 s gets an answer in microseconds.
- **Half-open:** after the break duration, a few probe calls go through; success closes the breaker, a failure re-opens it.
- **Why:** a slow dependency ties up caller threads/connections waiting on timeouts → the caller stalls → *its* callers stall (cascading failure). Full traffic plus retries also keeps the sick service from recovering.

## Trip logic and knobs

| Knob | Options | Defaults: Polly v8 · Resilience4j · Hystrix |
|---|---|---|
| Trip condition | N **consecutive** failures (simple, suits low traffic) or **failure rate** ≥ X% in a window | 10% · 50% · 50% |
| Window | **count-based** (last N calls) or **time-based** (last N s) | 30 s · last 100 calls · 10 s |
| Minimum throughput | evaluate only once the window holds ≥ N calls (1 failure in 2 calls ≠ outage) | 100 · 100 · 20 |
| Slow-call rate | calls slower than T count as failures even if they succeed | via an inner timeout · native · via timeout |
| What counts | timeouts, connection errors, 5xx → yes; 4xx (400/404/409) → no: a caller bug or a valid answer | .NET standard HTTP handler: network errors, timeouts, 5xx, 408, 429 |
| Break duration | how long to stay open; can grow on repeated trips | 5 s · 60 s · 5 s |
| Half-open probes | trial calls allowed before deciding | 1 · 10 · 1 |

## Advanced

- **Timeouts first:** a breaker only counts calls that *end*. Set connect and per-attempt timeouts, and propagate a **deadline** downstream (gRPC deadlines, .NET `CancellationToken`, Node `AbortSignal`) so callees drop work the caller already abandoned.
- **Ordering with retries:** usual stack is total timeout → retry → breaker → per-attempt timeout → call (the .NET standard handler; Resilience4j's default aspect order also nests the breaker inside retry). The breaker records every attempt, and once it opens the remaining retries fail instantly. Wrapping the retry *inside* the breaker counts a whole retried call as one outcome, so it trips later.
- **Retry amplification:** 3 attempts per layer × 3 layers = 3³ = 27 calls on the bottom service per user request — right when it's failing. Retry at **one** layer, with **exponential backoff + full jitter** (`sleep = random(0, min(cap, base × 2^attempt))`), capped by a **retry budget** (Google SRE: retries ≤ 10% of a client's requests; Envoy `retry_budget`; gRPC retry throttling). Return "overloaded, don't retry" errors upstream.
- **Fallbacks:** cached/stale data, a static default (empty recommendations), a degraded feature (hide the widget), or queue the write for later (outbox → RabbitMQ) and return 202. Never fall back to the same failing dependency or one sharing its database; fallbacks take full traffic while open, so keep them cheap.
- **Granularity:** per dependency is the default; per endpoint keeps a broken `/search` from blocking `/checkout`; per host is outlier detection's job. Too coarse blocks healthy paths; too fine never reaches minimum throughput (.NET: `SelectPipelineByAuthority()` gives one pipeline per host).
- **Per-instance vs shared state:** breakers normally live in process memory — no extra dependency, but each of 50 pods must see enough failures itself before tripping. Shared state (e.g. Redis) trips the whole fleet at once but adds a hop and a new failure mode; rarely worth it.
- **Half-open herd:** instances re-probing in lockstep can stampede a recovering service — jitter the break duration and grow it on repeated trips (Polly `BreakDurationGenerator`, Resilience4j exponential wait interval).
- **Bulkheads:** separate pools or concurrency limits per dependency, so a slow one exhausts only its own slots (Hystrix thread pools, Resilience4j `Bulkhead`, Polly concurrency limiter, `MaxConnectionsPerServer` on each `HttpClient` handler).
- **Load shedding & graceful degradation:** the server-side half — reject excess work early (429/503 + `Retry-After`), serve critical traffic first (checkout over recommendations), switch off non-essential features under stress.
- **Mesh vs app breaker:** Envoy/Istio "circuit breaking" = connection-pool limits (max connections, pending and active requests, concurrent retries; overflow fails fast) + **outlier detection** (eject a host after N consecutive 5xx, for a growing ejection time). That handles one bad pod among healthy ones; an app-level breaker handles the whole dependency being down and can run a business-aware fallback. Use both.
- **Adaptive concurrency limits:** instead of binary open/closed, continuously tune the allowed in-flight requests from observed latency, like TCP congestion control (Netflix `concurrency-limits`, Envoy adaptive concurrency filter). Google SRE's client-side throttling rejects locally with probability `max(0, (requests − K·accepts) / (requests + 1))`, K ≈ 2.
- **Operate & test it:** metric and log every state transition, alert on breakers stuck open, keep a manual force-open as a kill switch (Polly `ManualControl`, Resilience4j forced-open state). Prove trips and fallbacks with **fault injection** — Polly chaos strategies (ex-Simmy), Istio fault injection, Toxiproxy, AWS FIS.

## Trade-offs

- ✅ Failures cost microseconds instead of a full timeout; threads, connections and memory stay free for healthy work.
- ✅ Gives a struggling dependency room to recover; confines damage to the features that need it.
- ❌ Tuning is empirical: too sensitive flaps on blips, too lax never trips; low-traffic paths may never reach minimum throughput.
- ❌ While open, every call fails — including ones that would have succeeded (partial outages, coarse granularity).
- ❌ Fallback paths rarely run, so they rot unless exercised.

## In the wild

.NET: Polly v8 (`AddCircuitBreaker`), Microsoft.Extensions.Http.Resilience `AddStandardResilienceHandler()` (shipped with .NET 8: rate limiter → total timeout → retry → breaker → attempt timeout; supersedes Microsoft.Extensions.Http.Polly) · Java: Resilience4j (Netflix Hystrix is in maintenance mode) · Node: opossum · Go: sony/gobreaker · Envoy / Istio `DestinationRule` (connection pool + outlier detection) · Netflix concurrency-limits · Pattern origin: Michael Nygard's *Release It!*

## Gotchas

- **No timeout, no trip:** a hung call never fails, so the breaker never opens — and .NET `HttpClient.Timeout` defaults to 100 s.
- **Counting 4xx as failures** trips the breaker on caller bugs or legitimate "not found" answers.
- **Retry ≠ breaker:** retries bet the failure is transient, breakers bet it isn't — compose them, and don't retry open-circuit errors (`BrokenCircuitException`) in a loop.
- **Map "open" to a meaningful response:** a fallback or 503 + `Retry-After`, not a generic 500.
- **Queue consumers:** when the downstream breaker is open, pause consuming (cancel the RabbitMQ consumer) instead of nacking every message into the dead-letter queue.
- **Keep breaker state out of health checks:** if `/health` fails whenever a downstream breaker is open, the LB or Kubernetes pulls every instance at once — a partial outage becomes a total one.

## Related

[Service Mesh](02-service-mesh.md) · [Observability](03-observability.md) · [Availability](../01-foundations/02-availability.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Message Queues](../06-async-and-events/01-message-queues.md)

## Read more

- [CircuitBreaker — Martin Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Addressing Cascading Failures — Google SRE book](https://sre.google/sre-book/addressing-cascading-failures/)
- [Build resilient HTTP apps (standard resilience handler) — Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience)

<!-- nav -->

---

[← Saga Pattern](../07-distributed-coordination/04-saga-pattern.md) · [Index](../README.md) · [Service Mesh →](02-service-mesh.md)
