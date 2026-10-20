# Proxy vs Reverse Proxy

> **TL;DR** — Both are middlemen that terminate a connection and re-issue the request. A **forward proxy** acts for *clients*: servers see the proxy, not the client. A **reverse proxy** acts for *servers*: clients think it is the origin.

## How it works

```text
FORWARD — run by the client side; servers see the proxy's IP
Client A ─┐
Client B ─┼──► forward proxy ──► internet ──► any server
Client C ─┘

REVERSE — run by the site owner; clients see only the proxy
                                            ┌──► app 1
any client ──► internet ──► reverse proxy ──┼──► app 2
                                            └──► app 3
```

- **Forward proxy:** clients opt in (browser/OS settings, PAC file, `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` env vars) or the network forces traffic through it. It makes the outbound request itself, so destinations see its IP. Uses: egress filtering and allow-lists, audit/DLP, anonymity, shared caching, getting around (or enforcing) geo/IP restrictions, a fixed egress IP that partners can allow-list.
- **Reverse proxy:** DNS for `api.example.com` points at it; it terminates the client connection, picks a backend by host/path rules and forwards over its own pooled connection. Uses: load balancing, TLS termination, caching, compression, WAF/DDoS shielding, auth, and one entry point that hides the internal topology.
- **Two connections, not one:** an HTTP proxy parses each request and opens a separate TCP (usually TLS) session per hop; it doesn't relay packets. Exceptions: `CONNECT` tunnels and L4 proxies.

## Forward vs reverse

| | Forward proxy | Reverse proxy |
|---|---|---|
| Acts for | clients | servers |
| Run by | user, company network, ISP | service owner |
| Client aware of it? | yes — configured or forced | no — it *is* "the server" |
| Hides | client identity from servers | backend topology from clients |
| Typical jobs | egress control, audit, caching, anonymity | LB, TLS, caching, compression, WAF, auth, routing |
| Examples | Squid, Zscaler, Tor (SOCKS) | Nginx, HAProxy, Envoy, Traefik, Caddy, Cloudflare, IIS ARR, YARP |

## TLS at a reverse proxy

| Mode | Proxy reads HTTP? | Proxy → backend | Use when |
|---|---|---|---|
| Termination (offload) | yes | plain HTTP on a trusted network | default — enables L7 routing, caching, WAF |
| Re-encryption (bridging) | yes | a new TLS session | compliance / zero trust: encrypted on every hop |
| Passthrough | no — routes on the cleartext **SNI** in the ClientHello (L4) | the client's own TLS session, end to end | backend must hold the keys or verify client certs itself |

## Advanced

- **Transparent (intercepting) proxies:** the network redirects traffic to the proxy (policy routing, WCCP, iptables `TPROXY`) with no client config. For HTTPS they see only IP and SNI, unless they do **TLS interception**: terminate TLS with certificates minted on the fly from a corporate root CA installed on managed devices. That breaks certificate pinning and client-cert mTLS, and fails for apps that ship their own trust store.
- **`CONNECT` tunnelling:** for HTTPS through a forward proxy the client sends `CONNECT api.stripe.com:443`; the proxy opens TCP, replies `200`, then blindly relays bytes. Without interception it sees host:port only, so policy is per domain, not per URL.
- **SOCKS5 (RFC 1928):** protocol-agnostic proxy for any TCP (and UDP) with optional auth; no HTTP awareness, no caching. `ssh -D 1080` gives you one; Tor exposes one.
- **Preserving client identity:** the backend's socket peer is the proxy. L7 proxies append `X-Forwarded-For: client, proxy1, …` and set `X-Forwarded-Proto` / `X-Forwarded-Host`; RFC 7239 standardises `Forwarded: for=…;proto=…;host=…`. Clients can forge all of them: trust only entries added by proxies you know, reading from the right. ASP.NET Core: `UseForwardedHeaders` with `KnownProxies` / known networks (only loopback trusted by default, `ForwardLimit` = 1); Express: `trust proxy`; Nginx: `set_real_ip_from` + `real_ip_header`.
- **PROXY protocol (L4):** a TCP proxy can't add HTTP headers (or see them under TLS passthrough), so it prepends a small header with the original source/destination IP:port (v1 text, v2 binary; HAProxy, Envoy, AWS NLB opt-in, Nginx `listen … proxy_protocol`). Both ends must agree, or every connection breaks; the backend must accept it only from the proxy.
- **Slow-client buffering:** the proxy reads whole requests and responses at network speed, so app threads aren't tied up by slow mobile clients or slowloris (headers trickled in to hog connections). Nginx buffers both directions by default; turn it off for SSE/streaming (`X-Accel-Buffering: no`).
- **Pooling, protocol bridging, hygiene:** thousands of short client connections collapse into a few warm keep-alive upstream ones (Nginx needs `keepalive` + `proxy_http_version 1.1`); HTTP/2 and HTTP/3 (QUIC) to clients, HTTP/1.1 to backends; body/header size limits and timeouts; path and header rewrites (HTTP→HTTPS redirect, add HSTS, strip `Server`).
- **Micro-caching:** caching dynamic responses for ~1 s turns 1,000 req/s on a hot URL into ~1 backend req/s; `proxy_cache_lock` collapses concurrent misses into one upstream fetch.
- **Everything at the edge is a reverse proxy:** L7 load balancer (spreads load), API gateway (+ auth, keys, quotas), CDN (geo-distributed cache), WAF (inspects and blocks), Kubernetes ingress, service-mesh sidecar (reverse proxy inbound, forward proxy outbound). Often the same engine (Envoy, Nginx) with different config.
- **Request smuggling:** if proxy and backend disagree on where a request ends (`Content-Length` vs `Transfer-Encoding: chunked`), an attacker hides a second request inside the first; it lingers on a pooled upstream connection and gets glued to the next user's request. Reject ambiguous requests at the edge, patch both parsers, prefer HTTP/2 end to end.
- **Open proxies and SSRF:** an unauthenticated internet-facing forward proxy gets abused for spam, credential stuffing and hiding attackers; require auth or restrict source IPs. A proxy or "fetch this URL" feature with a user-controlled target enables **SSRF** into internal services or cloud metadata (`169.254.169.254`): allow-list destinations, block private/link-local ranges *after* DNS resolution (DNS rebinding), require IMDSv2 on AWS, route server-side fetches through an egress proxy (Stripe's Smokescreen).

## Trade-offs

- ✅ **Reverse:** one hardened entry point; TLS certs, WAF, auth, rate limits, caching and compression configured once; backends can move, scale and stay private.
- ✅ **Forward:** central egress control: allow-lists, audit, a stable egress IP, shared caching.
- ❌ An extra hop and another tier to run redundantly, or it becomes a SPOF.
- ❌ Termination and interception put plaintext and private keys on the proxy; interception breaks end-to-end trust and pinning.
- ❌ Backends lose the real client IP and scheme unless forwarded headers or PROXY protocol are configured, and trusting them blindly lets clients spoof both.

## In the wild

Forward: Squid, Zscaler, Tor, Stripe Smokescreen (egress control) · `HTTP(S)_PROXY` / `NO_PROXY` are honoured by curl and .NET `HttpClient`; Node's built-in `fetch` / `http` ignore them by default (use undici's `ProxyAgent`) · Reverse: Nginx, HAProxy, Envoy, Traefik, Caddy, Apache `mod_proxy` · Cloudflare, AWS ALB / CloudFront, Azure Front Door / Application Gateway · .NET: YARP, IIS ARR + URL Rewrite · Node: `http-proxy-middleware` (dev servers proxying `/api`).

## Gotchas

- **"Reverse proxy vs load balancer" trap:** an L7 LB *is* a reverse proxy whose main job is spreading load; a reverse proxy in front of a single server still earns its keep (TLS, buffering, caching, limits).
- **Scheme lost behind TLS termination:** the app sees `http://` → wrong redirect URLs, OAuth `redirect_uri` mismatches, missing `Secure` cookies, HTTPS-redirect loops. Fix with `X-Forwarded-Proto` + forwarded-headers middleware early in the pipeline.
- **Client IP = proxy IP:** rate limits, geo-blocking and audit logs all see the proxy, so everyone shares one bucket; trusting a raw `X-Forwarded-For` instead lets attackers pick their IP.
- **Corporate TLS interception vs Node:** `SELF_SIGNED_CERT_IN_CHAIN` / `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` → add the corporate root CA via `NODE_EXTRA_CA_CERTS`; never set `NODE_TLS_REJECT_UNAUTHORIZED=0`.
- **Nginx defaults that bite:** `client_max_body_size` 1 MB → `413` on uploads; response buffering breaks SSE; `proxy_read_timeout` 60 s kills idle WebSockets, which also need `Upgrade` / `Connection` forwarded explicitly (they're hop-by-hop headers).

## Related

[Load Balancing](02-load-balancing.md) · [CDN](03-cdn.md) · [Rate Limiting](04-rate-limiting.md) · [API Gateways](../02-apis-and-communication/03-api-gateways.md) · [Service Mesh](../08-microservices-and-reliability/02-service-mesh.md) · [Long Polling vs WebSockets](../02-apis-and-communication/06-long-polling-vs-websockets.md) · [SPOF](../01-foundations/03-spof.md)

## Read more

- [What is a reverse proxy? — Cloudflare Learning Center](https://www.cloudflare.com/learning/cdn/glossary/reverse-proxy/)
- [Configure ASP.NET Core to work with proxy servers and load balancers — Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/proxy-load-balancer)
- [HTTP request smuggling — PortSwigger Web Security Academy](https://portswigger.net/web-security/request-smuggling)

<!-- nav -->

---

[← Long Polling vs WebSockets](../02-apis-and-communication/06-long-polling-vs-websockets.md) · [Index](../README.md) · [Load Balancing →](02-load-balancing.md)
