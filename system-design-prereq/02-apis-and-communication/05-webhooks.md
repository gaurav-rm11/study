# Webhooks

> **TL;DR** — A webhook is an HTTP POST a provider sends to a URL you registered whenever an event happens (payment captured, PR opened). It replaces polling with push, but delivery is at-least-once, unordered and arrives from the open internet — so both sides need a hardening checklist.

## How it works

```text
PROVIDER                                        CONSUMER
event ─► outbox ─► sender ── POST /webhooks ──► verify signature + timestamp
                     ▲       (signed JSON)      persist / enqueue raw event
                     └── 2xx within seconds ◄── return 2xx
                                                worker (async): dedupe on id ─► process
no 2xx / timeout ─► retry w/ backoff + jitter (hours–days) ─► DLQ, disable endpoint
```

- **Registration:** the consumer registers an HTTPS URL plus the event types it wants; the provider issues a per-endpoint **signing secret**. Some prove URL ownership with a challenge first (Slack `url_verification`, Azure Event Grid validation handshake).
- **Payload:** `id` (unique event id — your dedupe key) · `type` (`invoice.paid`) · `created` timestamp · `data` (object snapshot or just its id) · often a schema/API version. Headers carry signature, timestamp and delivery id (`Stripe-Signature`; GitHub's `X-Hub-Signature-256` + `X-GitHub-Delivery`).
- **Examples:** Stripe and Razorpay (payment succeeded/failed), GitHub (push → CI), Slack Events API (message posted), Twilio (SMS status, incoming call).

## Receiver checklist

- **Verify the signature over the raw body:** recompute `HMAC-SHA256(secret, raw bytes)` — often prefixed with id/timestamp — and compare in constant time (`CryptographicOperations.FixedTimeEquals` in .NET, `crypto.timingSafeEqual` in Node).
- **Reject stale timestamps:** the timestamp is inside the signed string; refuse anything older than ~5 min (Stripe's library default) to block replays. GitHub sends no timestamp — dedupe is your only replay defence.
- **Ack fast, process async:** durably persist or enqueue the raw event (DB table, SQS, RabbitMQ with publisher confirms), then return 2xx — GitHub fails a delivery after 10 s, Slack wants a response within 3 s. Slow inline work → timeouts → retries → duplicates.
- **Dedupe by event id:** delivery is at-least-once; record processed ids under a unique constraint in the same transaction as the side effect (Postgres `INSERT … ON CONFLICT DO NOTHING`; SQL Server: unique index + catch the duplicate-key error) so handlers are idempotent.
- **Don't rely on ordering:** retries interleave (`updated` can beat `created`); compare the object's version/`updated_at` and drop older snapshots, or re-fetch the resource.
- **Ignore unknown event types:** return 2xx and skip; a 4xx/5xx triggers retries and can get the endpoint disabled. Providers add types over time, and wildcard subscriptions deliver them.

## Sender checklist

- **Never lose the event:** write it to a transactional outbox in the same DB transaction as the state change, then dispatch.
- **Retries:** exponential backoff + jitter spread over hours to days (Stripe: up to 3 days in live mode); only 2xx counts as delivered.
- **Per-endpoint isolation:** a queue and concurrency cap per endpoint, plus short timeouts, so one slow or dead endpoint can't stall everyone else's deliveries.
- **DLQ + manual replay:** after the last retry, park the event; offer "resend" in the dashboard/API and an events-list endpoint for catch-up.
- **Auto-disable failing endpoints:** after sustained failure, disable and email the owner instead of retrying forever.
- **Event versioning:** pin a payload schema version per endpoint; additive changes only; breaking changes ship as a new version or event type.
- **Secret rotation:** keep two secrets active during the overlap and sign with both (Stripe and Standard Webhooks allow multiple signatures in one header).
- **SSRF protection:** URLs are user-supplied → resolve DNS and block private, loopback and link-local ranges (cloud metadata `169.254.169.254`) at connect time to defeat DNS rebinding, don't follow redirects, send through an isolated egress proxy (Stripe open-sourced Smokescreen for this).
- **Static egress IPs:** send from a fixed, published IP set (NAT gateway) so customers can allowlist you.

## Advanced

- **Fat vs thin payloads:** *fat* = full object snapshot — no extra call, but may be stale or out of order and ships data on every delivery; *thin* = id + type, the receiver fetches current state — ordering stops mattering, at the cost of an API call per event (watch rate limits). Stripe's v2 "thin events" and Microsoft Graph's default change notifications are thin.
- **Reconciliation job:** webhooks get lost — endpoint down past the retry window, endpoint auto-disabled, handler bug; GitHub doesn't auto-retry failed deliveries at all. Periodically list recent objects/events via the API and repair drift. For payments, never mark an order paid from the browser redirect alone.
- **Sender at scale:** outbox → event bus (Kafka) → dispatcher workers with per-endpoint queues and rate limits → a delivery-attempt log customers can inspect.
- **Stronger auth:** HMAC proves the sender knows a shared secret; asymmetric signatures (Ed25519 in Standard Webhooks) remove the shared secret; mTLS or OAuth client credentials on the callback for high-assurance consumers. A "secret URL" alone is not auth — URLs leak into logs.
- **Local testing:** tunnels (ngrok, Cloudflare Tunnel, Visual Studio dev tunnels) expose localhost; provider CLIs forward and trigger events (`stripe listen --forward-to localhost:5000/webhooks`, `stripe trigger payment_intent.succeeded`); replay real deliveries from the provider dashboard.
- **Standards:** **Standard Webhooks** — `webhook-id`, `webhook-timestamp`, `webhook-signature` headers, HMAC-SHA256 over `id.timestamp.body`, reference libraries in many languages; **CloudEvents** (CNCF) — a common envelope (`id`, `source`, `type`, `specversion`) with an HTTP binding, used by Azure Event Grid and Knative.
- **Webhooks vs alternatives:** *polling* — no public endpoint needed and you control the pace, but latency = interval and most calls return nothing; *WebSockets/SSE* — push to clients that hold a connection (browsers, mobile), not cross-org server integration; *message queues* — durable, acked, replayable, but both sides need broker access, so they fit inside one trust boundary. Common combo: a webhook handler that publishes straight into your RabbitMQ/SQS.

## Trade-offs

- ✅ Near-real-time push with no wasted polling; the consumer needs only an HTTPS endpoint, in any language, across organisations.
- ❌ The consumer must expose a public, highly available endpoint — awkward behind corporate firewalls or on-prem.
- ❌ At-least-once, unordered and occasionally lost → idempotency, version checks and reconciliation are mandatory.
- ❌ The sender owns a delivery system: retries, per-tenant isolation, SSRF defence, secret management and a customer-facing delivery log.

## In the wild

Stripe, Razorpay, PayPal, Adyen (payments) · GitHub, GitLab (push → CI) · Slack Events API, Twilio callbacks, Shopify, Zoom · Webhooks-as-a-service: Svix, Hookdeck, Convoy · Azure Event Grid webhook subscriptions, AWS EventBridge API destinations, SNS HTTP/S subscriptions · .NET: Stripe.net `EventUtility.ConstructEvent` · Node: `stripe.webhooks.constructEvent`.

## Gotchas

- **Body parsers break signatures:** re-serialised JSON has different bytes, so a global `express.json()` or ASP.NET Core model binding → "signature mismatch" that only shows up in prod. Mount `express.raw()` on the webhook route; in ASP.NET Core read the raw `Request.Body` yourself.
- **Ack ≠ processed:** a 2xx for an event not yet durably stored (fire-and-forget `Task.Run`, an un-awaited promise) is lost on a crash or redeploy — the provider won't retry.
- **Duplicates arrive concurrently:** if your handler outlives the sender's timeout, the retry runs while the first attempt is still in flight — `SELECT`-then-`INSERT` dedupe races; let the unique constraint decide.
- **Retry floods after your outage:** hours of backlogged retries arrive at once; size the queue and workers, never process inline.
- **Twilio signs the full URL + POST params (HMAC-SHA1):** behind a TLS-terminating proxy your app sees `http://` or an internal host, recomputes a different signature and rejects every request — rebuild the public URL first.

## Related

[Idempotency](../06-async-and-events/02-idempotency.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [APIs](01-apis.md) · [Long Polling vs WebSockets](06-long-polling-vs-websockets.md) · [JWTs](04-jwts.md) · [Change Data Capture (CDC)](../04-databases/08-cdc.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md)

## Read more

- [Receive Stripe events in your webhook endpoint — Stripe docs](https://docs.stripe.com/webhooks)
- [Standard Webhooks specification](https://github.com/standard-webhooks/standard-webhooks/blob/main/spec/standard-webhooks.md)
- [webhooks.fyi — webhook security and design patterns (ngrok)](https://webhooks.fyi/)

<!-- nav -->

---

[← JWTs](04-jwts.md) · [Index](../README.md) · [Long Polling vs WebSockets →](06-long-polling-vs-websockets.md)
