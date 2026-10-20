# Geohashing

> **TL;DR** — A geohash encodes a (lat, lon) point as a short base32 string naming a rectangular cell; points that share a prefix share a cell. It turns 2-D "what's near me?" into 1-D prefix range scans on an ordinary B-tree, sorted set or partition key.

## How it works

```text
lon  -5.6  in [-180, 180] → 0 1 1 1 1 1 0 0 0 0 0 0 0
lat  42.6  in [ -90,  90] → 1 0 1 1 1 1 0 0 1 0 0 1
interleave (lon first)    → 01101 11111 11000 00100 00010
base32 (5 bits/char)      →   e     z     s     4     2    = "ezs42"
```

- **Bisect** each range: value ≥ midpoint → emit 1 and keep the upper half, else emit 0 and keep the lower half; repeat.
- **Interleave** the bits starting with longitude, then map every 5 bits to `0123456789bcdefghjkmnpqrstuvwxyz` (base32 without `a i l o`).
- **Each extra char = 5 more bits = a 32× smaller cell**, split alternately 8×4 and 4×8: odd lengths are square in degrees, even lengths 2:1 wide.
- **A geohash names a cell, not a point:** decoding returns a bounding box; centre ± half a cell is the error.
- **Prefix = parent cell:** `ezs42` ⊂ `ezs4` ⊂ `ezs`, so truncating zooms out — free aggregation by area.
- **Proximity search = prefix range scan:** B-tree index on a `geohash` column + `WHERE geohash LIKE 'ezs4%'` (i.e. `>= 'ezs4' AND < 'ezs5'`), or Redis GEO: `GEOADD` stores a 52-bit integer geohash as the sorted-set score (exact in a double) and `GEOSEARCH key FROMLONLAT … BYRADIUS 2 km` queries it.

## Precision

| Length | Cell w × h at the equator | ≈ Scale |
|---|---|---|
| 1 | 5,000 × 5,000 km | continent |
| 2 | 1,250 × 625 km | large country |
| 3 | 156 × 156 km | region |
| 4 | 39 × 19.5 km | city |
| 5 | 4.9 × 4.9 km | district |
| 6 | 1.2 × 0.61 km | neighbourhood |
| 7 | 153 × 153 m | block |
| 8 | 38 × 19 m | building |
| 9 | 4.8 × 4.8 m | phone-GPS accuracy — longer adds nothing real |

## Geohash vs alternatives

| Scheme | Cells | Key / order | Best at | Weak spot |
|---|---|---|---|---|
| Geohash | lat/lon rectangles, fixed size per length | base32 string or int; Z-order | prefix scans on any ordered store | edges, distortion, ignores density |
| Quadtree | rectangle split in 4 once it holds > N points | tree path | uneven density (deep in cities, shallow in deserts); in-memory nearby search | rebalancing as points move; lives in RAM |
| Google S2 | quads on the 6 faces of a cube, levels 0–30 | 64-bit cell ID; Hilbert curve | global indexing, low distortion, mixed-level region coverings | opaque IDs, heavier library |
| Uber H3 | hexagons (+12 pentagons per resolution), resolutions 0–15 | 64-bit index | 6 equidistant neighbours → clean k-rings, smoothing, aggregation (surge pricing, demand heat-maps) | children only approximately tile their parent |
| R-tree | bounding boxes of the data itself | tree | polygons, lines, exact KNN | needs DB support; slower under heavy churn |

## Advanced

- **Edge problem:** points 1 m apart can share no prefix — just north vs south of the equator at lon 0+ they hash to `s0000…` vs `kpbpb…`. Query the cell **plus its 8 neighbours** (bit arithmetic or lookup tables), then filter by exact distance and sort — haversine `d = 2R·asin(√(sin²(Δφ/2) + cos φ₁·cos φ₂·sin²(Δλ/2)))`, R ≈ 6,371 km.
- **Picking precision:** the 3×3 block reaches at least one full cell beyond the query point, so use the longest hash whose cell's *shorter* side ≥ the radius (2 km → length 5; 500 m → 6). Over-fetch grows with cell area; for a poor fit, cover the circle with many finer cells instead (what S2's `RegionCoverer` does).
- **Z-order underneath:** bit interleaving *is* a Z-order (Morton) curve. Locality holds inside a cell, but the curve jumps between quadrants, so adjacent cells can sit far apart in key order — a 9-cell search is up to 9 separate range scans. S2's Hilbert curve never jumps (consecutive cells always touch), though no 1-D order keeps every 2-D neighbour close.
- **Distortion:** cells are fixed in degrees, so their width in km shrinks with cos(latitude) — half at 60°, slivers near the poles. The grid also wraps at ±180° and has no neighbours across the poles; hand-rolled neighbour code tends to break at both.
- **Uneven density:** one fixed length gives packed city cells and empty desert ones → hot keys, hot shards, uneven query cost. Use a quadtree (split at N points) or mixed-level coverings (S2).
- **Live locations at scale:** drivers ping every few seconds — keep current positions in memory (Redis GEO or an in-process grid) partitioned by city/cell, move an entry only when its cell changes, and stream raw pings to Kafka for history. Redis GEO members can't expire individually: store last-seen and sweep, or write to time-bucketed keys.
- **Geo-sharding:** shard by region or geohash prefix so most queries hit one shard; queries near a shard edge fan out to the neighbour; split hot prefixes into longer ones. Hashing cell IDs (consistent hashing) balances load but scatters neighbours across shards.
- **Privacy:** truncate before storing, logging or sharing (length 5 ≈ 5 km, 6 ≈ 1 km). Snap to a coarse cell *before* computing "2.3 km away", or repeated readings trilaterate the exact spot; in sparse areas even a coarse cell can identify a person.

## Trade-offs

- ✅ Any ordered store becomes a spatial index — B-tree, Redis sorted set, Cassandra clustering key, DynamoDB sort key; no spatial engine needed.
- ✅ Hierarchical and compact: truncate to aggregate or coarsen; a prefix doubles as a shard key.
- ❌ Approximate by design: 9-cell query plus distance post-filter, and over-fetch grows with cell size.
- ❌ Fixed rectangular cells: latitude distortion, unequal neighbour distances (diagonal vs edge), no density adaptivity.
- ❌ Points only — polygons, routes and "inside this delivery zone?" need an R-tree or S2 coverings.

## In the wild

Redis GEO (`GEOADD` / `GEOSEARCH` / `GEODIST` over a sorted set) · PostGIS `geography` + GiST (R-tree) index: `ST_DWithin` for radius, `ORDER BY geom <-> point` for KNN, `ST_GeoHash` · MongoDB `2dsphere` (S2 cells) with `$near` / `$geoWithin` · Elasticsearch / OpenSearch `geo_point` (BKD tree) + `geo_distance` query + `geohash_grid` aggregation for map clustering · SQL Server `geography` + spatial index (a multi-level grid stored in a B-tree — the geohash idea) · MySQL InnoDB `SPATIAL` index (R-tree) · .NET: NetTopologySuite via EF Core spatial · Node: `ngeohash`, `h3-js` · Uses: ride-hailing (nearest drivers), food delivery (rider/restaurant matching, delivery zones), nearby search (Yelp-style), geofencing, map heat-maps.

## Gotchas

- **Shared prefix ⇒ near, but near ⇏ shared prefix** — never answer "nearby" with a single `LIKE 'prefix%'`.
- **Keep raw lat/lon** next to the hash: the hash is the index key; distances computed from cell centres are off by up to half a cell.
- **Coordinate order:** Redis `GEOADD key lon lat`, GeoJSON `[lon, lat]` and PostGIS `ST_MakePoint(lon, lat)` vs SQL Server `geography::Point(lat, lon, 4326)` and most humans; swapped values fail silently unless |lat| > 90.
- **Degrees aren't distances:** Euclidean distance on raw lat/lon is wrong away from the equator. Use haversine (spherical, within ~0.5% of the WGS-84 ellipsoid) or a `geography` type.
- **Postgres prefix `LIKE`** uses a B-tree only with `text_pattern_ops` or the C collation — otherwise write the range explicitly.
- **Redis GEO rejects latitudes beyond ±85.05°** (the Web Mercator limit), so it can't index polar points.

## Related

[Database Indexes](03-database-indexes.md) · [Database Sharding](06-database-sharding.md) · [Consistent Hashing](07-consistent-hashing.md) · [SQL vs NoSQL](01-sql-vs-nosql.md) · [Caching](../05-caching/01-caching.md) · [Long Polling vs WebSockets](../02-apis-and-communication/06-long-polling-vs-websockets.md) · [Batch vs Stream Processing](../06-async-and-events/03-batch-vs-stream-processing.md)

## Read more

- [Geohash — Wikipedia (algorithm, precision table)](https://en.wikipedia.org/wiki/Geohash)
- [S2 cell hierarchy — S2 Geometry docs](https://s2geometry.io/devguide/s2cell_hierarchy)
- [H3: Uber's hexagonal hierarchical spatial index — Uber Engineering](https://www.uber.com/blog/h3/)

<!-- nav -->

---

[← CDC](08-cdc.md) · [Index](../README.md) · [Caching →](../05-caching/01-caching.md)
