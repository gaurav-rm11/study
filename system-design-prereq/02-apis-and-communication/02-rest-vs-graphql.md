# REST vs GraphQL

> **TL;DR** — REST exposes many resource URLs whose response shape the server decides; GraphQL exposes one typed endpoint where each client asks for exactly the fields it needs. REST wins on simplicity and HTTP caching; GraphQL wins when many clients need different slices of a connected data graph.

## How it works

```text
REST — several round trips, server-shaped payloads
  GET /users/42           → full user object         (over-fetching)
  GET /users/42/orders    → orders, no line items    (under-fetching:
  GET /orders/7/items     → one call per order        N more calls)

GraphQL — one round trip, client-shaped payload
  POST /graphql  { user(id: 42) { name orders { total items { sku } } } }
```

- **REST:** resources named by URLs, manipulated with HTTP verbs, stateless requests. Each endpoint returns a server-defined representation, so clients over-fetch unused fields or under-fetch and make extra round trips.
- **GraphQL:** a typed schema in SDL (`type User { id: ID! orders: [Order!]! }`) behind one endpoint. **Queries** read (fields may resolve in parallel), **mutations** write (top-level fields run serially), **subscriptions** push events. The server validates the query against the schema, runs a **resolver** per field and returns JSON mirroring the query's shape.

## Side by side

| | REST | GraphQL |
|---|---|---|
| Endpoints | many, one per resource/collection | one, usually `POST /graphql` |
| Response shape | fixed by the server | chosen by the client per query |
| Contract | optional (OpenAPI) | mandatory typed schema, introspectable |
| Versioning | `/v2`, headers, media types | none — additive changes + `@deprecated` |
| HTTP/CDN caching | native: GET + URL + `Cache-Control`/`ETag` | defeated by POST to one URL; needs persisted queries or client caches |
| Errors | HTTP status codes | usually HTTP 200 + `errors` array, possibly with partial data |
| Real-time | bolt on WebSockets/SSE/webhooks | subscriptions in the spec |
| File uploads | native (multipart, pre-signed URLs) | not in the spec; multipart extension or pre-signed URLs |
| Tooling | curl, Postman, every proxy/cache/WAF | GraphiQL, codegen, schema registries; needs GraphQL-aware monitoring |
| Learning curve | low | higher: schema design, resolvers, DataLoader, cost limits |

## Advanced

- **N+1 resolvers:** 50 posts × `author` = 1 posts query + 50 author queries. **DataLoader** collects the keys requested during one execution tick, issues one batched call (`WHERE id IN (…)`) and caches per request to dedupe. Create loaders per request so cached data never leaks across users. Built into Hot Chocolate (.NET); `dataloader` package in Node.
- **Caching:** a POST to one URL is invisible to HTTP caches. **Persisted queries** send a hash instead of the query text; with Apollo's **APQ** the server learns the hash on first miss, after which clients can send it via GET, which CDNs cache. Clients add **normalised caches** (Apollo Client, Relay) that store each object once by identity (e.g. `__typename:id`), so a mutation result updates every view showing that object.
- **Abuse protection:** one request can nest `friends { friends { friends … } }` or alias the same `login` mutation 1,000 times. Defences: depth and alias limits; **cost analysis** (per-field cost × requested list sizes, reject over budget, rate-limit by cost, as GitHub's GraphQL API does); mandatory `first:` with a max; execution timeouts; introspection off in production (hygiene, not security); for first-party apps a **persisted-query allowlist** (trusted documents) so the server runs only pre-registered operations.
- **Errors:** a failing field becomes `null` plus an `errors` entry (`message`, `path`, `extensions.code`); nulls bubble up to the nearest nullable parent; siblings still return data. Status-code monitoring sees only 200s, so parse response bodies. Model expected business failures in the schema (`union PlaceOrderResult = Order | OutOfStock`) instead of `errors`.
- **Schema evolution:** clients get only the fields they select, so adding is free; mark old fields `@deprecated(reason: …)`, track per-field usage in a schema registry, delete at zero. Breaking: removing fields, adding required arguments, making a non-null output field nullable.
- **Federation:** each team owns a **subgraph**; a router (Apollo Federation's router, Hot Chocolate Fusion, WunderGraph Cosmo) composes them into a **supergraph**, joins entities by key (`@key(fields: "id")`) and plans the cross-service fan-out. Run composition checks in CI; each extra hop adds latency. Replaced schema stitching.
- **Subscriptions:** long-lived connections over WebSockets (`graphql-ws` protocol) or SSE; each server holds connection state, so scaling out needs a pub/sub backbone (Redis, Kafka) and connection-aware load balancing.
- **Relay connections:** the standard cursor pagination shape — `orders(first: 10, after: $cursor) { edges { cursor node { … } } pageInfo { hasNextPage endCursor } }` — maps directly onto keyset pagination.
- **REST maturity (Richardson):** L0 one endpoint, RPC-style (SOAP); L1 resources; L2 HTTP verbs + status codes, where most "REST" APIs stop; L3 **HATEOAS**: responses carry links to valid next actions (HAL `_links`) so clients discover transitions instead of hard-coding URLs. Rare in practice.
- **gRPC, the third option:** Protobuf over HTTP/2, codegen, streaming, deadlines; the usual pick for internal service-to-service calls, not browser-native.
- **BFF (Backend for Frontend):** one API per client type that aggregates downstream calls and shapes responses; it fixes over-/under-fetching without GraphQL, and a GraphQL server is often the BFF.

## When to use which

- **REST:** public/partner APIs with unknown consumers; resource-shaped CRUD; file transfer; heavy reliance on HTTP/CDN caching; small teams wanting the fewest moving parts.
- **GraphQL:** several clients (web, iOS, Android, TV) needing different shapes of the same data; high-latency mobile networks where round trips hurt; a UI layer aggregating many services; frontend teams iterating faster than backend endpoints can change.
- **gRPC:** internal, latency-sensitive, polyglot service-to-service calls and streaming.
- **Mix:** GraphQL or REST at the edge with gRPC or messaging inside is the common end state at scale.

## In the wild

GitHub (REST plus a GraphQL API with cost-based rate limits) · Shopify (GraphQL-first; REST Admin API now legacy) · Facebook (GraphQL's origin, open-sourced 2015) · Netflix (federated GraphQL on its DGS framework) · AWS AppSync (managed GraphQL) · Node: Apollo Server, GraphQL Yoga, Mercurius, `dataloader` · .NET: Hot Chocolate (+ Fusion, Strawberry Shake client), GraphQL.NET · Postgres-to-GraphQL engines: Hasura, PostGraphile · Federation routers: Apollo Router, WunderGraph Cosmo.

## Gotchas

- **GraphQL isn't a graph database** — it's an API query language; resolvers can read SQL, REST, gRPC, anything.
- **GraphQL moves N+1 to the server, it doesn't remove it** — without DataLoader one innocent query fans out into thousands of DB calls.
- **HTTP 200 ≠ success:** dashboards and alerts keyed on status codes show 0% errors while users see failures.
- **Authorize per type/field (or in the business layer), not per endpoint** — `order.customer.paymentMethods` reaches data through paths a single `/graphql` check never sees.
- **Request-count rate limits are meaningless for GraphQL** — one request can cost 1 or 1,000,000; budget by computed cost.
- **Don't adopt GraphQL just for over-fetching** — REST sparse fieldsets (`?fields=`), `?expand=` and a BFF solve most of it.

## Related

[APIs](01-apis.md) · [API Gateways](03-api-gateways.md) · [Long Polling vs WebSockets](06-long-polling-vs-websockets.md) · [Caching](../05-caching/01-caching.md) · [CDN](../03-traffic-and-edge/03-cdn.md) · [Rate Limiting](../03-traffic-and-edge/04-rate-limiting.md)

## Read more

- [Introduction to GraphQL — graphql.org](https://graphql.org/learn/)
- [Richardson Maturity Model — Martin Fowler](https://martinfowler.com/articles/richardsonMaturityModel.html)
- [GraphQL Cheat Sheet — OWASP](https://cheatsheetseries.owasp.org/cheatsheets/GraphQL_Cheat_Sheet.html)

<!-- nav -->

---

[← APIs](01-apis.md) · [Index](../README.md) · [API Gateways →](03-api-gateways.md)
