# Service Mesh

> **TL;DR** — A service mesh is an infrastructure layer for service-to-service (east-west) traffic: a proxy beside every service instance intercepts all calls and applies mTLS, retries, timeouts, routing and telemetry, programmed centrally by a control plane. It moves networking concerns out of every codebase into one uniform, language-agnostic layer.

## How it works

```text
      control plane (e.g. istiod): config · policy · discovery · certs
                 │ xDS: listeners, routes, endpoints, certs   │
                 ▼                                            ▼
 app A ──► Envoy sidecar ════════════ mTLS ═══════════► Envoy sidecar ──► app B
 └─────── pod A ───────┘                                └─────── pod B ───────┘
                  data plane: every call crosses both proxies
```

- **Data plane:** one proxy per instance, usually **Envoy** as a sidecar container in the pod (Linkerd uses its own Rust micro-proxy). The app still calls `http://payments`; traffic is transparently redirected through the local proxy to the callee's proxy, then to the callee.
- **Control plane** (Istio's `istiod`): watches Kubernetes services/endpoints and mesh config (routes, policies), compiles them into proxy config and pushes it; acts as the CA for workload certificates. It is not in the request path.

## Features

| Area | What you get without code changes |
|---|---|
| Security | **mTLS** on every hop; **workload identity** — a SPIFFE ID such as `spiffe://cluster.local/ns/shop/sa/orders` in the cert; **authorization policy** (only `checkout` may `POST /payments`); end-user JWT validation |
| Resilience | timeouts, retries, circuit breaking (connection-pool limits), **outlier detection** (eject failing hosts) |
| Traffic | L7 per-request load balancing (fixes HTTP/2 and gRPC connection pinning), locality-aware routing; **traffic shifting**: canary (95/5 split), blue-green, header-based routing, **mirroring** (shadow copy of live traffic, responses discarded) |
| Testing | **fault injection**: delays and aborts on chosen routes |
| Telemetry | per-route request rate, errors and latency, access logs, proxy trace spans |

## Mesh vs API gateway vs library

| | API gateway | Service mesh | Resilience library (Polly, Resilience4j) |
|---|---|---|---|
| Traffic | north-south (clients → services) | east-west (service ↔ service) | one app's outbound calls |
| Jobs | end-user auth, API keys, rate limits, request transforms, API versioning | mTLS, identity, retries/timeouts, traffic shifting, uniform telemetry | retries, breakers, business-aware fallbacks (serve cached data) |
| Runs | at the edge | beside every instance, or per node | in-process, one language |

## Advanced

- **Traffic interception:** an init container installs **iptables** rules redirecting the pod's TCP traffic to the proxy (Istio: outbound → 15001, inbound → 15006). That needs `NET_ADMIN`, so hardened clusters use a **CNI plugin** that programs the rules during pod network setup instead.
- **xDS:** Envoy's dynamic config APIs over gRPC streams — **LDS** (listeners), **RDS** (routes), **CDS** (clusters), **EDS** (endpoints), **SDS** (secrets/certs); **ADS** carries them all on one stream for ordered updates. Changes apply live, no restarts. At scale, pushing every service's config to every proxy bloats memory and push time — scope it (Istio `Sidecar` resource, discovery selectors).
- **Certificates:** the proxy's agent generates a key, gets a short-lived cert from the mesh CA (Istio default: 24 h) and rotates it automatically via SDS; private keys never leave the pod. Roll out in **PERMISSIVE** mode (accepts plaintext and mTLS), then enforce **STRICT**.
- **Tracing still needs the app:** proxies emit spans, but only the app knows which inbound request caused which outbound call — it must copy trace headers (`traceparent` or B3) across, or you get one-hop traces.
- **Costs:** two extra proxy hops per call (caller and callee side) add latency, mostly at the tail; sidecar CPU/memory × every pod (1,000 pods = 1,000 Envoys); sidecar upgrades mean pod restarts; debugging spans app, proxy and control plane; start/stop ordering bugs (app starts before its proxy, Jobs never finish) — Kubernetes native sidecar containers fix the ordering.
- **Sidecarless:** **Istio ambient** — a per-node **ztunnel** (Rust) handles L4: mTLS, identity, L4 policy, TCP telemetry; optional **waypoint** proxies (Envoy, e.g. one per namespace) add L7 only where needed. **Cilium** — eBPF in the kernel for L3/L4, a per-node Envoy for L7. **Proxyless gRPC** — gRPC libraries consume xDS directly: no extra hop, but language-bound and fewer features.
- **Implementations:** **Istio** — Envoy (sidecar or ambient), richest features (multi-cluster, VMs, fine-grained policy), most complexity. **Linkerd** — Rust micro-proxy, mTLS on by default, little config, lower overhead, fewer knobs. **Consul** — Envoy data plane spanning Kubernetes, VMs and Nomad. **AWS App Mesh** is discontinued — don't propose it (AWS points to ECS Service Connect and VPC Lattice).
- **Multi-cluster:** one mesh across clusters/regions with a shared trust root, cross-cluster discovery and east-west gateways; locality-aware failover shifts traffic to another zone or region when local endpoints fail.
- **Zero trust:** identity, not network location, is the perimeter — every call authenticated (mTLS), authorized per request (L7 policy) and encrypted. Complements Kubernetes `NetworkPolicy`, which is IP/port-based (L3/L4).
- **Gateway API convergence:** the Kubernetes Gateway API (`Gateway`, `HTTPRoute`) now configures both ingress and mesh routing (the GAMMA initiative, supported by Istio and Linkerd) — fewer mesh-specific CRDs to learn.

## Trade-offs

- ✅ Uniform mTLS, identity, authz, traffic control and telemetry across every service and language, changed by config instead of redeploys.
- ✅ Safer releases: canaries, traffic shifting, mirroring, fault injection.
- ✅ Worth it with many polyglot services, mTLS/zero-trust mandates or a need for uniform traffic control — and a platform team to own it.
- ❌ Latency and CPU/memory overhead on every call and pod; another complex distributed system to upgrade, secure and debug.
- ❌ Not worth it for a handful of services, a single-language stack (a library does the job) or a small team.

## In the wild

Istio and Linkerd (both CNCF graduated) · Consul service mesh · Cilium Service Mesh · Kuma / Kong Mesh · Envoy as the common data plane · Managed: Google Cloud Service Mesh, AKS Istio-based add-on · Dapr is a sidecar *app runtime* (pub/sub, state, service invocation with mTLS), not a full mesh · .NET and Node apps keep calling plain `http://payments` via `HttpClient`/`fetch`; the sidecar upgrades the hop to mTLS.

## Gotchas

- **Retries stack:** mesh + app (Polly, axios-retry) + gateway retries multiply (3 × 3 × 3 = 27 attempts) — retry at one layer.
- **Protocol detection:** Istio sniffs undeclared ports, and what it can't identify becomes opaque TCP — no L7 routing, retries or per-request metrics. Declare `appProtocol` or port names (`http-`, `grpc-`); server-first protocols (e.g. MySQL) defeat sniffing, so declare them as TCP.
- **STRICT mTLS breaks non-mesh callers** (legacy VMs, jobs outside the mesh) — check for remaining plaintext traffic in PERMISSIVE mode first.
- **Control plane down ≠ instant outage:** proxies keep their last config, but new pods can't get config and certs eventually expire.
- **A mesh doesn't fix chatty architecture:** long synchronous call chains still multiply latency and failure odds — the mesh just makes them visible.

## Related

[Circuit Breaker](01-circuit-breaker.md) · [Observability](03-observability.md) · [API Gateways](../02-apis-and-communication/03-api-gateways.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Proxy vs Reverse Proxy](../03-traffic-and-edge/01-proxy-vs-reverse-proxy.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md) · [JWTs](../02-apis-and-communication/04-jwts.md)

## Read more

- [What is a service mesh? — Linkerd](https://linkerd.io/what-is-a-service-mesh/)
- [Istio architecture — Istio docs](https://istio.io/latest/docs/ops/deployment/architecture/)
- [Ambient mode overview — Istio docs](https://istio.io/latest/docs/ambient/overview/)

<!-- nav -->

---

[← Circuit Breaker](01-circuit-breaker.md) · [Index](../README.md) · [Observability →](03-observability.md)
