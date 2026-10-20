# Load Balancing

> **TL;DR** — A load balancer (LB) sits in front of a pool of servers, spreads traffic across the healthy ones and pulls dead ones out of rotation. It's what makes horizontal scaling and zero-downtime deploys possible.

## How it works

```text
                          ┌──► Server A  ✓
Client ──► DNS ──► LB ────┼──► Server B  ✓
                  (VIP)   └──► Server C  ✗  failed health check → gets no traffic
```

- Clients hit one address (a virtual IP or DNS name); the LB picks a backend per **connection** (L4) or per **request** (L7).
- **Health checks** — *active*: the LB probes `GET /health` every N s; a backend is marked down after X failures, up after Y passes. *Passive* (outlier detection): eject a backend that keeps returning 5xx/timeouts on real traffic.
- **Connection draining** — on deploy or scale-in, stop sending new requests to a backend but let in-flight ones finish.

## L4 vs L7

| | L4 (transport) | L7 (application) |
|---|---|---|
| Sees | IPs, ports, TCP/UDP | HTTP method, host, path, headers, cookies |
| Balances | connections | individual requests |
| TLS | usually passthrough | terminates TLS so it can read HTTP |
| Extras | very fast, any protocol | path/host routing, retries, header rewrites, auth, caching, compression |
| Examples | AWS NLB, kube-proxy, LVS/IPVS, Maglev | AWS ALB, Nginx, HAProxy, Envoy, YARP |

## Algorithms

| Algorithm | Picks | Use when |
|---|---|---|
| Round robin | next server in turn | identical servers, uniform requests |
| Weighted round robin | in proportion to weights | mixed instance sizes; canary (send 5%) |
| Least connections / requests | fewest in-flight | long or uneven requests (uploads, WebSockets) |
| Least response time (EWMA) | lowest recent latency | backends with varying latency |
| Power of two choices (P2C) | 2 random servers, take the less loaded | big fleets — near-optimal without global state (Envoy's least-request default) |
| IP hash / consistent hash | `hash(client IP or key) → server` | stickiness, cache affinity |
| Random | uniform random | stateless and good enough at scale |

## Advanced

- **Global load balancing (GSLB):** GeoDNS / latency-based DNS (e.g. Route 53) or **anycast** (one IP announced from many sites; BGP routes to the network-nearest) picks the region; a regional LB does the rest. DNS failover is slow — resolvers and clients cache past the TTL.
- **Who balances the balancer?** An active-passive pair sharing a floating VIP (VRRP/keepalived), or routers spreading flows via **ECMP** across a fleet of stateless L4 LBs that all use the same consistent hash, so a flow keeps its backend even if one LB dies (Google Maglev, Facebook Katran).
- **Direct Server Return (DSR):** the L4 LB handles only inbound packets; servers reply straight to the client. Big win when responses dwarf requests (video, downloads).
- **Long-lived connections defeat L4:** HTTP/2, gRPC and WebSockets multiplex many requests over one TCP connection, so an L4 LB pins all of them to one backend. Use L7 (per-request) or client-side balancing, and recycle connections periodically (e.g. gRPC max connection age).
- **Client-side load balancing:** the client gets the endpoint list from service discovery (Consul, Kubernetes endpoints, xDS) and picks a backend itself — no extra hop. Used by gRPC and service-mesh sidecars.
- **Slow start:** ramp traffic gradually to a new instance (cold caches, JIT warm-up); otherwise least-connections floods it the moment it joins.
- **Bounded-load consistent hashing:** cap each server at (1+ε) × average load and spill the overflow to the next server on the ring — cache affinity without hot spots (HAProxy `hash-balance-factor`).
- **Retries:** L7 LBs can retry idempotent requests on another backend; cap them with a retry budget (e.g. ≤ 20% extra load) or retries amplify an outage.
- **Fail open / panic mode:** when most backends look unhealthy, many LBs send traffic to *all* of them rather than black-holing it (AWS ALB when every target is unhealthy; Envoy below its default 50% healthy "panic threshold").

## Trade-offs

- ✅ Horizontal scaling, failover, zero-downtime deploys (drain → update → re-add), one place for TLS certs and edge policy.
- ❌ An extra network hop, another thing to run and pay for, and a SPOF/bottleneck unless it's redundant.
- ❌ Sticky sessions skew load and lose sessions when a server dies; prefer stateless servers plus a shared session store.

## In the wild

Nginx, HAProxy, Envoy, Traefik · AWS ALB / NLB, GCP Cloud Load Balancing, Azure Load Balancer / Application Gateway / Front Door · Kubernetes Service (L4 via kube-proxy) + Ingress / Gateway API (L7) · .NET: YARP · Google Maglev, Facebook Katran (L4 at hyperscale).

## Gotchas

- In a design, place LBs explicitly: DNS/GSLB → CDN/edge → regional L7 LB → services, plus LBs (or client-side LB) between internal tiers.
- **Deep health checks** (that also ping the DB) can mark *every* server unhealthy at once when a shared dependency blips → self-inflicted outage. Keep LB checks shallow; monitor dependencies separately.
- Detection time ≈ check interval × unhealthy threshold — that's how long a dead server keeps getting traffic.
- IP-hash stickiness breaks when mobile clients change IPs and is skewed by NAT (a whole office behind one IP); cookie-based affinity avoids both.
- Timeouts must nest: backend keep-alive timeout > LB idle timeout (Node's 5 s default vs ALB's 60 s is a classic cause of random 502s), and WebSocket heartbeats must be more frequent than the LB idle timeout.

## Related

[Proxy vs Reverse Proxy](01-proxy-vs-reverse-proxy.md) · [Consistent Hashing](../04-databases/07-consistent-hashing.md) · [Scalability](../01-foundations/01-scalability.md) · [SPOF](../01-foundations/03-spof.md) · [Stateful vs Stateless](../01-foundations/04-stateful-vs-stateless.md) · [API Gateways](../02-apis-and-communication/03-api-gateways.md) · [Service Mesh](../08-microservices-and-reliability/02-service-mesh.md)

## Read more

- [Load Balancing — interactive visual guide (Sam Rose)](https://samwho.dev/load-balancing/)
- [Implementing health checks — AWS Builders' Library](https://aws.amazon.com/builders-library/implementing-health-checks/)
- [Maglev: a fast and reliable software network load balancer (Google)](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/)

<!-- nav -->

---

[← Proxy vs Reverse Proxy](01-proxy-vs-reverse-proxy.md) · [Index](../README.md) · [CDN →](03-cdn.md)
