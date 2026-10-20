# Content Delivery Network (CDN)

> **TL;DR** — A CDN is a worldwide fleet of caching reverse proxies (points of presence, **PoPs**) that answers requests close to users. Most requests never cross an ocean or reach your origin: lower latency, less origin load and bandwidth, and edge capacity that soaks up DDoS traffic.

## How it works

```text
User ──► nearest PoP  (picked by GeoDNS or anycast)
           ├─ HIT  ✓ ──► served from the edge cache, no origin trip
           └─ MISS ✗ ──► shield (mid-tier) PoP ──► origin
                          ◄── response cached at each tier per its headers
```

- **Routing to a PoP:** *GeoDNS*: the CDN's DNS answers with a PoP near the user's resolver (EDNS Client Subnet sharpens this for public resolvers). *Anycast*: every PoP announces the same IP and BGP delivers packets to the network-nearest one, which also spreads attack traffic across all PoPs.
- **Hit:** served from PoP memory/SSD. **Miss:** fetched from origin, stored if the response headers and CDN rules allow, then served; later users at that PoP hit until the TTL expires.
- **Cache:** static assets (JS, CSS, images, fonts), downloads, video segments, public HTML and API responses with short TTLs. **Don't** (or key very carefully): authenticated/personalised responses, anything with `Set-Cookie`.
- **Why:** TCP + TLS handshakes complete over a short RTT to a nearby PoP instead of 100–250 ms across continents; origin offload; lower origin egress; DDoS absorption; TLS terminated near users.

## HTTP caching headers

| Header / directive | Effect |
|---|---|
| `Cache-Control: max-age=N` | fresh for N s in every cache (browser and CDN) |
| `s-maxage=N` | overrides `max-age` in shared caches (CDN, proxies) only |
| `public` / `private` | `private`: browser only, CDN must not store. `public`: shared caches may store it even for requests carrying `Authorization` |
| `no-cache` | may store, but must revalidate with the origin before every reuse |
| `no-store` | never store anywhere |
| `immutable` | won't change while fresh; browsers skip revalidation on reload |
| `stale-while-revalidate=N` | serve stale for up to N s while refetching in the background |
| `stale-if-error=N` | serve stale for up to N s while the origin errors or is down |
| `ETag` / `Last-Modified` | validators: cache sends `If-None-Match` / `If-Modified-Since`; unchanged → `304 Not Modified`, no body |
| `Vary: <header>` | adds that request header to the cache key |
| `CDN-Cache-Control` (RFC 9213), `Surrogate-Control` | CDN-only directives; browsers ignore them |

## Invalidation

| Approach | How | Trade-off |
|---|---|---|
| Fingerprinted names | `app.3f9a1c.js` + `max-age=31536000, immutable`; each deploy emits new names | no purge, atomic switch; the HTML that references them must stay short-TTL or revalidated |
| Purge by URL | API call per URL | precise, but you must enumerate every URL and variant |
| Purge by tag / surrogate key | origin tags responses (`Surrogate-Key` on Fastly, `Cache-Tag` on Cloudflare); purge the tag | "product 42 changed" clears every page that shows it |
| Wildcard / purge-all | prefix or everything | blunt; a cold cache stampedes the origin |
| Short TTL + SWR | `s-maxage=60, stale-while-revalidate=600` | no purge plumbing; bounded staleness |

Purges fan out to PoPs asynchronously (sub-second to minutes, depending on vendor) and never touch copies already in browsers.

## Advanced

- **Pull vs push:** *pull*: the CDN fetches on the first miss; zero upload work; the default. *Push*: you upload to CDN storage ahead of time, for large, rarely-changing files (installers, game patches, VOD) where first-miss latency or a launch-day stampede is unacceptable; you own versioning and cleanup.
- **Cache-key design:** default key ≈ scheme + host + path + query. Normalise (sort params, lowercase), drop tracking params (`utm_*`, `gclid`, `fbclid`), ignore cookies, and vary only on what changes the body (`Accept-Encoding`, a coarse device class, never raw `User-Agent`). Every extra variant splits the hit ratio.
- **Tiered caching / origin shield:** edge PoPs miss to a regional tier instead of the origin, so the origin sees ~one fetch per object per shield, not per PoP. **Request collapsing:** concurrent misses for one key share a single origin fetch, which stops thundering herds after a purge or on a viral object.
- **Metrics:** hit ratio, byte hit ratio and **origin offload** (requests/bytes that never reach origin). Raising hit ratio from 95% to 99% cuts origin traffic 5×. Debug with `Age` and `X-Cache` / `CF-Cache-Status` response headers.
- **Dynamic acceleration:** uncacheable API calls still gain. TLS terminates nearby, then requests ride warm, pooled edge→origin connections, sometimes over optimised routes (Cloudflare Argo, Akamai SureRoute), skipping handshake round trips over the long path.
- **Edge compute:** code in every PoP: Cloudflare Workers (V8 isolates), Fastly Compute (Wasm), Lambda@Edge / CloudFront Functions, Akamai EdgeWorkers. Good for token checks, redirects, A/B bucketing, header and cache-key tweaks, geo logic; limited by CPU/memory caps and data that still lives in one region.
- **Video:** HLS/DASH cut streams into few-second segments plus a manifest, each an ordinary cacheable object; adaptive-bitrate players switch renditions per segment. Live manifests get second-scale TTLs, segments long ones.
- **Security:** WAF, bot management and rate limiting at the edge. **Signed URLs/cookies** (expiry + path, signature verified at the edge) protect paid or private content without an origin auth hop. **Lock the origin** to the CDN: IP allow-lists, a secret header (Azure Front Door's `X-Azure-FDID`), mTLS (Cloudflare Authenticated Origin Pulls), private connectivity (Cloudflare Tunnel, CloudFront OAC for S3). Keep the origin IP out of DNS history and email headers, or attackers simply bypass the WAF.
- **Cache poisoning and deception:** *poisoning*: the response depends on an **unkeyed** input (e.g. `X-Forwarded-Host` reflected into a script URL), so one malicious request is cached and served to everyone. *Deception*: `/account/settings/x.css` returns the private page and the CDN caches it because it looks static. Key on everything that changes the response, cache by explicit rules, mark personalised pages `private` / `no-store`.
- **Multi-CDN:** DNS-based steering across 2+ vendors by health, performance or cost survives a CDN-wide outage (Fastly, June 2021). Costs: lowest-common-denominator features, duplicated config and WAF rules, purging everywhere, a lower hit ratio per vendor.

## Trade-offs

- ✅ Lower global latency, cheap read scaling via origin offload, DDoS/WAF at the edge, TLS near users; `stale-if-error` rides out origin blips.
- ❌ Staleness and invalidation complexity; one cache-key mistake leaks private data or craters the hit ratio.
- ❌ Per-GB/per-request cost, lock-in (edge-compute, purge and config APIs), and a dependency whose outage is your outage.
- ❌ Little help for personalised or write-heavy traffic beyond connection acceleration; one more layer to debug.

## In the wild

Cloudflare (anycast, Workers), Akamai (servers inside ISP networks), Amazon CloudFront, Fastly (Varnish heritage, fast purge), Azure Front Door, Google Cloud CDN / Media CDN · Netflix Open Connect (its own appliances inside ISPs) · self-hosted layers: Varnish, Nginx `proxy_cache` · fingerprinting: webpack `[contenthash]`, Vite by default · ASP.NET Core: `[ResponseCache]` sets `Cache-Control`; `asp-append-version` and .NET 9 `MapStaticAssets` fingerprint static files.

## Gotchas

- **Caching personalised responses** (or a `Set-Cookie`) under a shared key serves user A's page or session to user B; Steam did exactly this on Christmas 2015. Don't trust CDN defaults to bypass cookies or `Authorization`; send `private` / `no-store`.
- **Purges don't reach browsers:** a long `max-age` on HTML pins old copies in browsers whatever you purge. Use a long `s-maxage` (purgeable) with a short `max-age` for anything that may change.
- **Stale after deploy:** cached old HTML references assets you deleted → 404s; new HTML references assets not yet uploaded. Upload assets first and keep old fingerprints around for a while.
- **Query strings:** `?v=2` busting fails if the CDN ignores query strings in the key; random query strings (`?_=<random>`) force misses, an easy way to hammer the origin *through* the CDN.
- **`Vary: User-Agent` / `Vary: Cookie`** means near-unique cache keys and a ~0% hit ratio; some CDNs ignore most `Vary` values or refuse to cache.
- **Interview trap:** "put a CDN in front" is half an answer. Say what's cacheable, the TTLs, how you invalidate, and what a miss storm does to the origin.

## Related

[Caching](../05-caching/01-caching.md) · [Caching Strategies](../05-caching/02-caching-strategies.md) · [Cache Eviction Policies](../05-caching/03-cache-eviction-policies.md) · [Proxy vs Reverse Proxy](01-proxy-vs-reverse-proxy.md) · [Load Balancing](02-load-balancing.md) · [Rate Limiting](04-rate-limiting.md) · [Availability](../01-foundations/02-availability.md)

## Read more

- [HTTP caching — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [Web cache poisoning — PortSwigger Web Security Academy](https://portswigger.net/web-security/web-cache-poisoning)
- [What is a CDN? — Cloudflare Learning Center](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/)

<!-- nav -->

---

[← Load Balancing](02-load-balancing.md) · [Index](../README.md) · [Rate Limiting →](04-rate-limiting.md)
