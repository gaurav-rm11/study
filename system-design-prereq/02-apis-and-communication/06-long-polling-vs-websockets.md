# Long Polling vs WebSockets

> **TL;DR** — HTTP is request/response, so the server can't speak first. To push updates you keep asking (short polling), park a request until there's news (long polling), stream one never-ending response (Server-Sent Events), or upgrade to a persistent full-duplex socket (WebSockets).

## How it works

```text
LONG POLLING
C ── GET /events?after=41 ─────────► S   held open until data arrives or ~30 s pass
C ◄── 200 [{seq:42}, {seq:43}] ───── S
C ── GET /events?after=43 ─────────► S   re-issued at once with the new cursor

WEBSOCKET HANDSHAKE
C ── GET /ws  Upgrade: websocket ──► S   + Connection: Upgrade, Sec-WebSocket-Key
C ◄── 101 Switching Protocols ────── S   Sec-WebSocket-Accept = b64(SHA1(key+GUID))
C ◄═════ frames, both directions ══► S   same TCP connection, no more HTTP
```

- **Server-Sent Events (SSE):** one `GET` answered with `Content-Type: text/event-stream` that never ends; the server writes `id:` / `event:` / `data:` lines as events happen.
- **WebSockets (RFC 6455):** after the `101`, the TCP connection carries small frames both ways — text, binary, ping/pong, close. Client→server frames are masked with a 4-byte key so proxies can't be cache-poisoned.

## Comparison

| | Short polling | Long polling | Server-Sent Events | WebSockets |
|---|---|---|---|---|
| Direction | client pulls | server → client; client sends via normal requests | server → client | full duplex |
| Connection | new request every N s | one held request per wait, then re-opened | one long-lived HTTP response | one long-lived TCP connection |
| Overhead per message | full request + headers, mostly empty replies | full request + headers | a few bytes of framing | 2–14-byte frame header |
| Latency | up to the interval | near-immediate (+1 RTT between polls) | immediate | immediate |
| Payload | anything | anything | UTF-8 text only | text or binary |
| Infra compatibility | any HTTP stack; cacheable | any HTTP stack; hold < proxy timeouts | plain HTTP; buffering proxies delay it | every hop must allow `Upgrade` + long idle |
| Browser API | `fetch` | `fetch` | `EventSource` (auto-reconnect) | `WebSocket` |

## Advanced

- **Long polling without gaps or dupes:** events that land between polls must not be lost → keep a per-user/topic log; the client polls with its last sequence id (`after=43`) and dedupes by id.
- **Long polling holds a request per client:** thread-per-request servers starve → use async I/O (Node, async ASP.NET Core). Answer empty (e.g. 204) before any proxy/LB idle timeout (often 60 s) or clients see 502/504s.
- **SSE details:** the browser auto-reconnects (server tunes the delay via `retry:`) and sends `Last-Event-ID` so the server replays from there. One-way and text-only (binary → base64). Over HTTP/1.1 browsers allow ~6 connections per domain, shared across tabs → the 7th tab's stream stalls; HTTP/2 multiplexes streams on one connection. Disable proxy buffering (Nginx: `X-Accel-Buffering: no`) or events arrive in bursts.
- **WebSockets are stateful infrastructure:** every hop (CDN, LB, ingress) must pass `Upgrade` and tolerate long idle periods — Nginx needs `proxy_http_version 1.1` plus `Upgrade`/`Connection` headers, and its `proxy_read_timeout` and AWS ALB's idle timeout both default to 60 s. Autoscaling doesn't move existing connections; new instances only get new ones.
- **Per-server limits:** each socket costs a file descriptor plus kernel and app buffers → raise `ulimit -n`, budget memory per connection, watch LB ephemeral ports. Idle sockets are cheap (Phoenix demoed 2M on one server); message rate and fan-out are the real limits. `permessage-deflate` (RFC 7692) keeps zlib state per socket, so Node `ws` leaves it off server-side by default.
- **User → connection mapping:** a user has N sockets (tabs, devices) across M servers → keep a presence registry (Redis `user → {server, connId}` with a TTL refreshed by heartbeats) or address users via pub/sub channels.
- **Cross-server fan-out:** the sender's server rarely holds the recipient's socket → publish to a **backplane** every node subscribes to: Redis Pub/Sub, Kafka, NATS, RabbitMQ topic exchanges; SignalR's Redis backplane, or Azure SignalR Service, which also hosts the connections. Redis Pub/Sub is fire-and-forget — pair it with a durable log for catch-up.
- **Heartbeats:** ping/pong control frames every ~15–30 s detect half-open TCP — a phone that switched networks never sends FIN — and keep NAT/proxy idle timers alive. Browsers auto-answer server pings but the JS API can't send them → clients need app-level heartbeats to notice a dead server (SignalR: its own ping message every 15 s, 30 s timeout).
- **Reconnect storms:** a deploy or LB failure drops 100k sockets at once and they all reconnect together, hammering auth, DB and presence. Clients use exponential backoff + full jitter, then **resume** from the last sequence id instead of reloading full state.
- **Draining on deploy:** stop routing new upgrades to old instances, ask clients to reconnect in staggered waves (close code 1001 "going away" or an app message with a random delay), and give the LB a long deregistration delay.
- **Auth at the handshake:** the browser `WebSocket` constructor can't set custom headers → use the cookie (and validate `Origin` to stop **cross-site WebSocket hijacking**), a short-lived token in the query string (SignalR's `access_token`; scrub it from logs), or a token in the first message (close if absent within seconds). Re-authorize long-lived sockets when the token expires.
- **Backpressure:** a slow client makes the server's send queue grow until memory runs out → bounded per-connection buffers; coalesce (latest-value-wins for prices/presence), drop, or disconnect. Watch `bufferedAmount` (browser, Node `ws`).
- **Transport security:** always `wss://` (TLS also keeps middleboxes from mangling the upgrade); cap message size and per-connection message rate.
- **Successors and side channels:** **WebTransport** (HTTP/3 over QUIC) adds multiple streams and unreliable datagrams without TCP head-of-line blocking; support is still maturing. WebSockets can also be bootstrapped over HTTP/2 (RFC 8441) and HTTP/3 (RFC 9220). Backgrounded mobile apps lose their sockets → notify via **APNs/FCM** push and resync on foreground.

## When to use which

| Situation | Pick |
|---|---|
| Updates every minute or slower; simplest stack | Short polling (cacheable, stateless) |
| Server → client stream: notifications, feeds, dashboards, LLM token streaming | SSE |
| Bidirectional, frequent, low-latency: chat, multiplayer, collaborative editing, trading | WebSockets |
| Proxies that block `Upgrade`, legacy clients, a fallback path | Long polling |
| App backgrounded or closed | Mobile push (APNs/FCM) |
| Server-to-server events | Webhooks or a message queue |

## In the wild

.NET **SignalR** negotiates WebSockets → SSE → long polling and scales out via a Redis backplane or Azure SignalR Service · **Socket.IO** (Node) opens with HTTP long polling, then upgrades to WebSocket — its own protocol, so plain WebSocket clients can't connect; Redis adapter for multiple nodes · Node `ws`, uWebSockets.js · Managed: AWS API Gateway WebSocket APIs, Azure Web PubSub, Pusher, Ably · Slack and Discord clients run on WebSockets (Discord's gateway is Elixir) · OpenAI and Anthropic APIs stream tokens over SSE.

## Gotchas

- **Idle timeouts in the path** (LB, proxy, NAT, corporate firewall) silently kill quiet connections → the heartbeat interval must beat the smallest one.
- **SignalR scale-out needs sticky sessions** unless every client uses WebSockets with `SkipNegotiation` or you use Azure SignalR Service — negotiation and fallback transports span several HTTP requests.
- **A socket is not a delivery guarantee:** messages sent while a client reconnects are gone unless you add sequence ids, a replay buffer and app-level acks.
- **Don't default request/response APIs to WebSockets:** you lose HTTP caching, status codes, per-request load balancing and standard auth/rate-limit middleware.

## Related

[Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Proxy vs Reverse Proxy](../03-traffic-and-edge/01-proxy-vs-reverse-proxy.md) · [Stateful vs Stateless](../01-foundations/04-stateful-vs-stateless.md) · [Webhooks](05-webhooks.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [JWTs](04-jwts.md) · [Scalability](../01-foundations/01-scalability.md)

## Read more

- [RFC 6455 — The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [ASP.NET Core SignalR hosting and scaling — Microsoft Learn](https://learn.microsoft.com/en-us/aspnet/core/signalr/scale)
- [Real-time Messaging — Slack Engineering](https://slack.engineering/real-time-messaging/)

<!-- nav -->

---

[← Webhooks](05-webhooks.md) · [Index](../README.md) · [Proxy vs Reverse Proxy →](../03-traffic-and-edge/01-proxy-vs-reverse-proxy.md)
