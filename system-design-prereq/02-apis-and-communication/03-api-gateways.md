# API Gateways

> **TL;DR** — An API gateway is the single entry point between clients and backend services: it routes each request to the right service and applies cross-cutting edge policy (auth, rate limits, TLS, transformation, telemetry) once, instead of in every service.

## How it works

```text
                 Web · Mobile · Partners
                            │ HTTPS
                            ▼
┌──── API gateway (stateless replicas behind an LB) ────┐
│ TLS → request ID → authN/Z → rate limit → route       │
│ → transform / aggregate / cache → telemetry           │
└───────┬──────────────────┬──────────────────┬─────────┘
        ▼                  ▼                  ▼
  Orders (REST)      Users (gRPC)       Search (REST)
```

- **Routing:** by host, path, method or header to a service; rewrites prefixes; finds backends via service discovery.
- **AuthN + coarse authZ:** validates JWTs (signature via JWKS, `exp`, `iss`, `aud`) or API keys and checks per-route scopes (`orders:write`), rejecting junk before it reaches services.
- **Rate limits and quotas:** per key, user, IP or plan; 429 + `Retry-After`.
- **TLS termination:** certificates in one place; optionally re-encrypt or mTLS to backends.
- **Transformation:** header/path/body rewrites; **protocol translation** (REST/JSON ↔ gRPC); **aggregation** (one client call fans out to several services, results merged).
- **Edge extras:** response caching, CORS preflight, compression, request-size limits, WAF rules.
- **Observability:** access logs and per-route rate/errors/latency metrics; creates the request ID and starts the trace (W3C `traceparent`) that every downstream hop propagates.

## Gateway vs proxy vs LB vs mesh

| | Reverse proxy | Load balancer | API gateway | Service mesh |
|---|---|---|---|---|
| Core job | forward to backends: TLS, caching, compression | spread traffic over healthy replicas | API policy: auth, keys, quotas, transformation | service-to-service mTLS, retries, LB, telemetry |
| Traffic | north-south | north-south, also between tiers | north-south (clients → system) | east-west (service ↔ service) |
| Understands | hosts, paths, headers | backend health and load | consumers, API keys, plans, versions | workload identities |
| Deployed as | central proxy tier | central tier or client-side | central tier, or one per client (BFF) | sidecar or per-node proxy |
| Examples | Nginx, HAProxy | AWS ALB / NLB | Kong, AWS API Gateway, Azure APIM | Istio, Linkerd |

Boundaries blur: Envoy or Nginx can fill every column; a gateway is a reverse proxy plus API-management features.

## Advanced

- **Backend for Frontend (BFF):** one gateway per client type (web, mobile, partners), owned by that client's team and shaping responses for its screens, so no single gateway accretes every client's special cases. Cost: duplicated logic; push shared rules down into services.
- **Aggregation vs GraphQL:** gateway aggregation is hand-written fan-out per endpoint (`GET /home` → profile + orders + recommendations in parallel, each with its own timeout and a degraded fallback). A GraphQL gateway or federation router lets clients compose queries and plans the fan-out itself.
- **Anti-patterns:** business logic in the gateway (it becomes an ESB / distributed monolith); a central gateway team approving every route change (use self-service declarative config instead: GitOps, Kubernetes Gateway API `HTTPRoute`s owned per team); stacking CDN → WAF → gateway → ingress → sidecar until hops dominate latency.
- **SPOF and latency:** run several stateless replicas across availability zones behind an L4 LB; keep state (counters, keys) in external stores; roll out config like code (staged, canaried), since one bad global config is a global outage.
- **Auth patterns:** validate at the edge, then propagate identity as a signed token (forward the JWT or mint a short-lived internal one), never a bare `X-User-Id` header anything inside could forge; services still verify it (zero trust). **Phantom token:** clients hold opaque tokens (revocable, leak no claims); the gateway introspects them (RFC 7662) and forwards a JWT internally; RFC 8693 token exchange formalises swapping tokens per audience. Strip client-supplied internal headers at the edge.
- **Distributed rate limiting:** N replicas counting locally allow N × the limit. Options: shared counters in Redis (atomic `INCR` or Lua; one extra round trip per request), local limits of limit/N (approximate, skewed by uneven balancing), or local buckets synced periodically. Kong's rate-limiting plugin offers `local`, `cluster` and `redis` policies; Envoy has local and global (rate-limit service) modes.
- **Release control:** weighted splits for canaries (1% → 100%), blue-green cut-over, header-based routing for testers, and **request mirroring** (shadow live traffic to a new version and discard its responses; make sure mirrored writes can't double-charge anyone).
- **API management:** developer portal, self-service keys, usage plans/quotas per pricing tier, analytics, monetisation (AWS usage plans, Azure APIM products and subscriptions, Apigee).
- **Resilience at the edge:** per-route timeouts, retries only for idempotent methods and within a retry budget, circuit breaking / outlier ejection per upstream, load shedding with 503 + `Retry-After`. Retries multiply across layers: client, gateway and service each trying 3 times = 27 calls on the bottom service.
- **Protocols:** WebSockets need upgrade support, long idle timeouts and connection draining on deploys; gRPC needs HTTP/2 end-to-end (trailers), or gRPC-Web / JSON transcoding at the edge for browsers. AWS API Gateway has separate REST, HTTP and WebSocket API types.

## Trade-offs

- ✅ Cross-cutting concerns implemented once and consistently; services keep to business logic.
- ✅ Hides service topology: split, merge or move services behind stable public routes; one place for keys, quotas and analytics.
- ✅ Fewer client round trips via aggregation; internal services can speak gRPC while clients use REST.
- ❌ Another hop and another critical tier to scale, secure and keep highly available.
- ❌ Bottleneck risk, both organisational (central team) and architectural (logic creep).
- ❌ Managed gateways bill per request and impose hard limits (timeouts, payload size).

## In the wild

Kong (OpenResty/Nginx-based, plugin ecosystem) · AWS API Gateway (Lambda integrations, usage plans) · Azure API Management (XML policies, products/subscriptions, self-hosted gateway) · Google Apigee · Tyk · Envoy Gateway (Kubernetes Gateway API on Envoy) · NGINX / NGINX Plus, Traefik · KrakenD (aggregation-focused) · Netflix Zuul, Spring Cloud Gateway · .NET: YARP (Microsoft's reverse-proxy library to build a gateway or BFF on) and Ocelot (config-driven gateway: routing, auth, rate limiting, aggregation) · Node: BFFs usually hand-built on Fastify/NestJS.

## Gotchas

- **The gateway does coarse authZ only:** "valid token with scope `orders:read`" belongs at the edge; "may user 7 read order 42?" belongs in the service. Broken Object Level Authorization is #1 on the OWASP API Security Top 10.
- **Bypass risk:** if services trust gateway-added headers, anything that reaches them directly (misrouted traffic, SSRF) can impersonate any user; lock down network paths and verify signed identity in each service.
- **Timeouts must nest:** client > gateway > service > DB. A gateway that times out first returns 504 while the backend may still commit the write; the client retries and duplicates it unless the call carries an idempotency key.
- **Managed limits bite late:** AWS API Gateway REST APIs default to a 29 s integration timeout and 10 MB payloads; design long jobs as 202 + polling and send big files to object storage via pre-signed URLs.
- **Gateway metrics stop at the edge:** they show a slow route, not which service hop is slow; you still need distributed tracing through every service.
- **Not every system needs one:** one client and three services can live with a reverse proxy plus auth middleware; a gateway pays off with many consumers, API keys/quotas or many services.

## Related

[Proxy vs Reverse Proxy](../03-traffic-and-edge/01-proxy-vs-reverse-proxy.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md) · [JWTs](04-jwts.md) · [REST vs GraphQL](02-rest-vs-graphql.md) · [Service Mesh](../08-microservices-and-reliability/02-service-mesh.md) · [Circuit Breaker](../08-microservices-and-reliability/01-circuit-breaker.md)

## Read more

- [Pattern: API Gateway / Backends for Frontends — microservices.io](https://microservices.io/patterns/apigateway.html)
- [Backends For Frontends — Sam Newman](https://samnewman.io/patterns/architectural/bff/)
- [The Phantom Token Pattern — Curity](https://curity.io/resources/learn/phantom-token-pattern/)

<!-- nav -->

---

[← REST vs GraphQL](02-rest-vs-graphql.md) · [Index](../README.md) · [JWTs →](04-jwts.md)
