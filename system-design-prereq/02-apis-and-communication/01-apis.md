# APIs

> **TL;DR** — An API is a contract: the operations a service exposes, their inputs, outputs and errors, plus promises about auth, limits and retries. Clients use the system without knowing its internals, so both sides can change independently as long as the contract holds.

## How it works

```text
POST /v1/orders HTTP/1.1                 ← method + resource + version
Authorization: Bearer eyJhbGciOi...      ← who is calling
Idempotency-Key: 5f1c9e2a-...            ← makes a retry safe
Content-Type: application/json

{"sku": "A-100", "qty": 2}               ← validated against the schema

HTTP/1.1 201 Created                     ← outcome as a status code
Location: /v1/orders/8412                ← where the new resource lives
```

- The contract is everything a client can observe: operations, request/response schemas, the error model and behaviour (auth, rate limits, idempotency, pagination, versioning policy). Write it down (OpenAPI, `.proto`, GraphQL SDL) or clients will reverse-engineer it.

## Styles

| Style | Mechanism | Best for | Weak spots |
|---|---|---|---|
| REST | resources + HTTP verbs, usually JSON | public/partner APIs, HTTP caching | over-/under-fetching, chatty UIs |
| GraphQL | one endpoint, typed schema, client picks fields | UIs over many services, mobile | caching, N+1, query-cost abuse |
| gRPC | Protobuf over HTTP/2, generated stubs, streaming | internal service-to-service | not browser-native, binary payloads |
| WebSockets | full-duplex frames on one upgraded HTTP connection | chat, live dashboards, games | stateful connections, harder to scale |
| Webhooks | provider POSTs events to your URL | payment/SaaS event notifications | receiver owns retries, dedup, signature checks |
| SOAP (legacy) | XML envelope + WSDL + WS-* standards | old enterprise/banking integrations | verbose, heavy tooling |

## HTTP essentials

- **GET, HEAD, OPTIONS** are *safe* (no intended state change) and idempotent; **PUT, DELETE** are idempotent; **POST** and **PATCH** are neither by default. *Idempotent* = N identical requests leave the same server state as one (responses may differ: the second DELETE gets 404). Clients and proxies should auto-retry only idempotent methods; Nginx won't resend a POST/PATCH to another upstream unless told to.

| Class | Codes that matter |
|---|---|
| 2xx | **200** OK · **201** Created + `Location` · **202** Accepted, processing async · **204** No Content |
| 3xx | **301** / **302** permanent / temporary redirect (**308** / **307** keep the method) · **304** Not Modified |
| 4xx | **400** malformed · **401** not authenticated · **403** authenticated, not allowed · **404** not found · **409** conflicts with current state · **412** precondition (`If-Match`) failed · **422** well-formed but invalid · **429** rate-limited |
| 5xx | **500** bug · **502** bad response from upstream · **503** overloaded/down, retry later · **504** upstream timed out |

## Advanced

- **Resource naming:** plural nouns, IDs in the path, shallow nesting only for ownership (`/customers/42/orders`), filters in the query string, no verbs. Non-CRUD actions become sub-resources (`POST /orders/42/cancellation`) or custom methods (`POST /orders/42:cancel`, Google AIP-136).
- **Versioning:** URI (`/v2/orders`) — explicit, easy to route and cache; header or query (`Stripe-Version`, Azure's `?api-version=2024-05-01`) — clean URLs; media type (`Accept: application/vnd.acme.v2+json`) — purist, awkward to test. Avoid new versions by shipping additive changes; when you must break, run both and announce the end with `Deprecation` and `Sunset` (RFC 8594) headers.
- **Compatibility rules:** safe — new endpoints, optional fields and params. Breaking — removing/renaming fields, changing types or defaults, making a field required, tightening validation, new enum values for clients that switch exhaustively. Clients should be **tolerant readers** (ignore unknown fields, default branch for unknown enums). Protobuf: the field **number** is the wire identity — never change or reuse one, mark deleted ones `reserved`; renames are wire-safe but break generated code and JSON mapping.
- **Contract-first:** write the OpenAPI / `.proto` before code, review it like code, generate server stubs, typed clients, docs and mocks; lint (Spectral) and fail CI on breaking diffs (`oasdiff`, `buf breaking`).
- **Pagination:** `OFFSET 100000` makes the DB walk and discard 100k rows (cost grows with depth), and inserts/deletes between requests shift rows → duplicates and gaps. **Keyset/cursor:** `WHERE (created_at, id) < (@ts, @id) ORDER BY created_at DESC, id DESC LIMIT 50` seeks via an index — constant cost, stable under writes; return an opaque `next_cursor` (encoded last key); needs a unique tiebreaker; no jumping to page 37. SQL Server lacks row-value comparison: `created_at < @ts OR (created_at = @ts AND id < @id)`.
- **Filtering, sorting, fields:** `?status=paid&sort=-created_at&fields=id,total&expand=customer` — allow-list filterable/sortable fields (each needs an index), cap `limit`; sparse fieldsets shrink payloads; `expand` (Stripe) inlines related objects instead of extra calls.
- **Errors:** one shape everywhere — RFC 9457 Problem Details (`application/problem+json`: `type`, `title`, `status`, `detail`, `instance`, plus extensions such as per-field `errors`); stable machine-readable codes; no stack traces. ASP.NET Core: `AddProblemDetails()`.
- **Idempotency keys:** client sends a UUID per logical operation in `Idempotency-Key` on POST; server stores key → request hash + response (Stripe: at least 24 h), committed atomically with the side effect; a retry gets the stored response. IETF draft: same key, different body → 422; original still in flight → 409.
- **Conditional requests:** responses carry an `ETag`. GET + `If-None-Match: "v7"` → **304**, no body, if unchanged. PUT/PATCH + `If-Match: "v7"` → applied only if still current, else **412** → re-read, merge, retry: optimistic concurrency that prevents lost updates. Back it with a version column (SQL Server `rowversion`, EF Core concurrency token); APIs that demand `If-Match` return **428** without it.
- **Long-running operations:** don't hold a request open for minutes (LB/gateway idle timeouts); return **202** + `Location: /operations/123`, client polls (honouring `Retry-After`) until `succeeded`/`failed` plus a result link, or gets a webhook callback.
- **Bulk endpoints:** `POST /orders:batchCreate` with a capped item count; state whether it's all-or-nothing or partial, and for partial return a per-item status so clients retry only the failures.
- **Rate-limit signalling:** **429** + `Retry-After`; publish budgets via `X-RateLimit-Limit/-Remaining/-Reset` (de facto) or the IETF draft `RateLimit`/`RateLimit-Policy` headers; clients back off with jitter.
- **Auth:** **API keys** identify an *app* — simple, long-lived bearer secrets; fine for server-to-server and metering. **OAuth 2.0** issues short-lived scoped access tokens — Authorization Code + PKCE for user-facing apps, Client Credentials for services; **OIDC** adds an ID token saying who the user is. **mTLS** authenticates both ends with certificates — strong workload identity (meshes, open banking) at the cost of cert issuance and rotation.
- **gRPC specifics:** HTTP/2 multiplexes calls over one long-lived connection (an L4 LB pins them all to one backend → use L7 or client-side LB). Modes: unary, server streaming, client streaming, bidirectional. Set a **deadline** on every call and propagate it so downstream work stops when the caller gives up (`DEADLINE_EXCEEDED`). Status codes (`UNAVAILABLE`, `RESOURCE_EXHAUSTED`…) travel in HTTP/2 trailers, which browsers can't read → **gRPC-Web** via a proxy (Envoy, `Grpc.AspNetCore.Web`) or JSON transcoding.
- **Internal vs external:** external → REST/JSON + OpenAPI (every language, browser, cache and gateway speaks it); internal → gRPC (smaller payloads, typed clients, streaming, deadlines), or a queue when the caller doesn't need the answer now.
- **Hyrum's law:** with enough users, every observable behaviour — error text, field order, default sort, timing — is depended on by someone. Document guarantees, randomise what isn't guaranteed (why Go randomises map iteration), treat any visible change as potentially breaking.

## Trade-offs

- ✅ A stable contract decouples teams: implementation, storage and language can change behind it.
- ❌ Everything exposed is a long-term liability — removal costs a deprecation cycle, so expose less.
- ✅ REST/JSON is universal, debuggable and cacheable. ❌ Verbose, chatty for UIs, schema-less without OpenAPI.
- ✅ gRPC gives compact payloads, codegen, streaming and deadlines. ❌ Needs HTTP/2 end-to-end, weak browser story, hard to eyeball.
- ✅ Cursor pagination is fast and stable at any depth. ❌ No random page access or cheap totals.

## In the wild

Stripe (idempotency keys, `expand`, cursor pagination via `starting_after`, date-based versions pinned per account) · GitHub (REST with `Link`-header pagination and `X-GitHub-Api-Version`, plus GraphQL) · Google AIPs (resource-oriented design, custom methods, long-running operations) · Microsoft REST API Guidelines / Azure (`api-version`, async operation polling) · gRPC (open-sourced successor to Google's internal Stubby) · .NET: ASP.NET Core Minimal APIs, `ProblemDetails`, `Microsoft.AspNetCore.OpenApi`, `Asp.Versioning`, `Grpc.AspNetCore` · Node: Express, Fastify, NestJS, `@grpc/grpc-js`.

## Gotchas

- **401 vs 403:** 401 = "who are you?" (must send `WWW-Authenticate`); 403 = "I know you — no". Return 404 when even admitting the resource exists leaks information.
- **`200 OK` with `{"success": false}`** breaks retries, caches, client libraries and error-rate dashboards.
- **Retry only what's retryable:** 429/503 after `Retry-After`; 502/504 and network errors only for idempotent calls or with an idempotency key; never resend 400/401/403/404/409/422 unchanged.
- **PATCH idempotency depends on the format:** JSON Merge Patch (RFC 7396) setting fields is effectively idempotent; a JSON Patch (RFC 6902) `add` to `/items/-` appends again on every retry.
- **JavaScript loses integer precision above 2^53** — send 64-bit IDs as strings (Twitter's `id_str`), money as integer minor units or decimal strings, timestamps as ISO 8601 UTC.
- **Offset pagination on a live feed** shows duplicates and misses, and `?page=100000` is a cheap DoS on your database.

## Related

[REST vs GraphQL](02-rest-vs-graphql.md) · [API Gateways](03-api-gateways.md) · [JWTs](04-jwts.md) · [Webhooks](05-webhooks.md) · [Long Polling vs WebSockets](06-long-polling-vs-websockets.md) · [Idempotency](../06-async-and-events/02-idempotency.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md)

## Read more

- [RFC 9110 — HTTP Semantics (methods, status codes, conditional requests)](https://www.rfc-editor.org/rfc/rfc9110)
- [Google API Improvement Proposals (AIPs)](https://google.aip.dev/general)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)

<!-- nav -->

---

[← Concurrency vs Parallelism](../01-foundations/05-concurrency-vs-parallelism.md) · [Index](../README.md) · [REST vs GraphQL →](02-rest-vs-graphql.md)
