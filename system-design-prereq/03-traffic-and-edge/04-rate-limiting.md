# Rate Limiting

> **TL;DR** — Rate limiting caps how many requests a client (user, API key, IP, tenant) may make per time window and rejects or delays the excess. It stops abuse, buggy clients and noisy neighbours from eating shared capacity, and enforces quotas and pricing tiers.

## How it works

```text
request ──► key = API key | user | tenant | IP  (+ route)
        ──► limiter state for that key (in-process or Redis)
              ├─ within limit ✓ ──► handle it, spend budget
              └─ over limit   ✗ ──► 429 Too Many Requests + Retry-After
```

- **Why:** brute force and credential stuffing, scraping, fairness between tenants, protecting capacity and fragile dependencies, capping spend (paid upstream APIs, LLM tokens), pricing tiers (free 60/min, paid 1,000/min).
- **Response:** `429` + `Retry-After` (seconds or an HTTP date); advertise budgets with `X-RateLimit-Limit` / `-Remaining` / `-Reset` (de facto, GitHub-style) or the IETF draft `RateLimit` / `RateLimit-Policy` fields.
- **Where:** *client* (SDK self-throttling; a courtesy, never trusted) → *edge/WAF* (coarse per-IP, before traffic costs you) → *API gateway* (per API key/plan; the usual home) → *service* (business rules: 5 OTP attempts per account per 15 min) → *per dependency* (stay under a partner's limit, protect your DB).

## Algorithms

| Algorithm | Mechanism | Bursts | State per key |
|---|---|---|---|
| Token bucket | holds ≤ B tokens, refilled at r/s; each request spends one | up to B at once, then r/s | tokens + last refill time (refill lazily on access) |
| Leaky bucket | requests queue and drain at a constant rate; full queue → reject | none: smooth output, queueing adds latency | queue length |
| Fixed window counter | `INCR count:{key}:{floor(now/W)}` | up to 2× the limit around a window boundary | 1 counter |
| Sliding window log | timestamp per request; count those in the last W | exact | every timestamp (memory-heavy) |
| Sliding window counter | `prev × (W − elapsed)/W + curr` | close approximation, smooth | 2 counters |
| Concurrency limiter | cap in-flight requests; release on completion | limits parallelism, not rate | 1 counter (semaphore) |

Token bucket is the API default (AWS API Gateway, Stripe); leaky bucket shapes traffic (Nginx `limit_req`); the sliding window counter is cheap and accurate enough at scale (Cloudflare); concurrency limits guard slow, expensive endpoints.

## Rate limiting vs its cousins

| Control | Triggered by | Action |
|---|---|---|
| Rate limiting | one client exceeding its allowance | reject the excess (429) |
| Throttling | the same | delay, queue or degrade instead of rejecting (often used as a synonym) |
| Load shedding | the *server* overloaded (CPU, queue depth, latency), whoever is calling | drop low-priority work first (503) |
| Backpressure | a consumer slower than its producer | make upstream slow down: TCP windows, bounded queues (.NET `Channel<T>`, Node stream `drain`), RabbitMQ prefetch and publisher flow control |
| Adaptive concurrency | rising latency | auto-tune the in-flight limit (AIMD, TCP-Vegas-style gradients: Netflix `concurrency-limits`); in-flight = throughput × latency (Little's law) |

## Advanced

- **Distributed limits need atomic ops:** instances share counters in Redis. Fixed window: `INCR` + `EXPIRE … NX` in one `MULTI` or Lua script (a crash between separate calls leaves a key with no TTL: a permanent block). Token bucket: one Lua script reads, refills, spends and writes atomically. Never `GET` → compare → `SET` from the app; concurrent instances all read 99/100 and all allow.
- **GCRA:** token-bucket behaviour storing one value per key, a "theoretical arrival time" pushed forward by 1/rate per request; reject if it runs further ahead of now than the burst allowance (redis-cell).
- **Latency vs accuracy:** a Redis round trip per request adds latency and a hot shared dependency (hot keys for big tenants). Cheaper: per-instance buckets at limit/N (wrong under uneven load or autoscaling), local counting with periodic sync to the central store (approximate, brief overshoot), or reserving tokens in batches. Envoy layers both: a local token bucket as first line, a global rate-limit service for exact quotas.
- **Fail open vs fail closed:** when the limiter store is down or slow (give it a tight timeout), fail **open** for general traffic, since availability beats precision, ideally with a generous local fallback; fail **closed** for security limits (login, OTP, password reset, payments) so an outage doesn't open a brute-force window.
- **Multi-dimensional limits:** check several keys per request (API key, user, IP, tenant, endpoint, global) and reject on the first exceeded. **Rates** (per second; burst protection) ≠ **quotas** (per day/month; billing, durable, calendar resets). **Plan tiers** map keys to limits via cached config. **Cost-weighted** limits charge expensive calls more tokens (search = 10, read = 1); GitHub's and Shopify's GraphQL APIs budget by computed query cost.
- **Client behaviour:** honour `Retry-After`; otherwise exponential backoff with full jitter, `sleep = random(0, min(cap, base × 2^attempt))`, plus a retry cap or budget. Without jitter, rejected clients retry in lock-step and recreate the spike. .NET: Polly / `Microsoft.Extensions.Http.Resilience` (can honour `Retry-After`).
- **Identity is the hard part:** NAT/CGNAT and corporate networks put thousands of users behind one IP, IPv6 gives each user at least a /64 (limit per prefix, not per address), and botnets rotate IPs, so per-IP limits are both unfair and easy to evade. Prefer authenticated keys (API key, user, tenant); keep IP limits for unauthenticated endpoints (login, signup) and coarse guards.
- **DDoS is a different layer:** volumetric L3/L4 floods saturate links before your app's limiter runs; that's anycast/CDN scrubbing (Cloudflare, AWS Shield, Azure DDoS Protection). App limiters handle L7 abuse and fairness.

## Trade-offs

- ✅ Protects availability and cost from abuse, bugs and retry storms; isolates tenants; makes quotas and pricing tiers enforceable.
- ❌ Legitimate bursts get rejected; size burst capacity per tier and alert on 429 rates per client.
- ❌ Exact distributed limits cost a network hop and a hot shared store; local limits are cheap but approximate.
- ❌ Static limits don't track real capacity: too high and they don't save a degraded backend, too low and capacity sits idle. Pair them with load shedding or adaptive concurrency.

## In the wild

ASP.NET Core `AddRateLimiter` + `[EnableRateLimiting]` (.NET 7+: fixed window, sliding window, token bucket, concurrency; partitioned per key; in-memory per instance) · Node: `express-rate-limit` + `rate-limit-redis`, `rate-limiter-flexible`, `@fastify/rate-limit`, `@nestjs/throttler` · Nginx `limit_req` / `limit_conn` · Envoy local + global rate limiting (external gRPC service backed by Redis) · Kong rate-limiting plugin (local / cluster / Redis) · AWS API Gateway usage plans + API keys · Azure API Management `rate-limit-by-key` / `quota-by-key` · Cloudflare rate-limiting rules · GitHub REST API: 5,000 req/h authenticated vs 60/h unauthenticated · Stripe: token bucket + concurrency limiter + load shedders.

## Gotchas

- **Fixed-window boundary burst:** "100/min" admits 200 requests within two seconds around :59/:00, a classic interview probe; answer with a sliding window counter or token bucket.
- **Wrong status code:** ASP.NET Core's rate limiter and Nginx `limit_req` both reject with **503** by default. Set `RejectionStatusCode = 429` / `limit_req_status 429` and add `Retry-After`, or clients and dashboards read it as an outage.
- **Per-instance limits under autoscaling:** an in-memory 100/s limiter is really 100 × N/s, drifts as you scale, and is uneven under skewed load balancing.
- **Keying on the proxy's IP:** behind an LB/CDN everyone shares a few buckets; trusting a raw `X-Forwarded-For` instead lets attackers pick their key.
- **Login limits need per-account *and* per-IP keys:** per-IP alone misses distributed credential stuffing; hard per-account lockouts let attackers lock victims out, so prefer progressive delays or CAPTCHA.
- **Per-client limits ≠ capacity protection:** 1,000 clients each within 100/s is still 100k/s; add a global limit and load shedding.

## Related

[API Gateways](../02-apis-and-communication/03-api-gateways.md) · [Proxy vs Reverse Proxy](01-proxy-vs-reverse-proxy.md) · [CDN](03-cdn.md) · [Load Balancing](02-load-balancing.md) · [Circuit Breaker](../08-microservices-and-reliability/01-circuit-breaker.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [Availability](../01-foundations/02-availability.md)

## Read more

- [Scaling your API with rate limiters — Stripe](https://stripe.com/blog/rate-limiters)
- [How we built rate limiting capable of scaling to millions of domains — Cloudflare](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/)
- [Rate limiting middleware in ASP.NET Core — Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/performance/rate-limit)

<!-- nav -->

---

[← CDN](03-cdn.md) · [Index](../README.md) · [SQL vs NoSQL →](../04-databases/01-sql-vs-nosql.md)
