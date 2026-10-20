# Observability

> **TL;DR** — Observability is how well you can infer a system's internal state from its outputs — logs, metrics, traces, profiles — so you can debug failures nobody predicted (unknown-unknowns) without shipping new code. Monitoring checks known failure modes; observability lets you ask new questions mid-incident.

## How it works

```text
trace 4bf92f35…   each hop sends traceparent: 00-<trace-id>-<parent-id>-01
GET /checkout             gateway     ████████████████████████████  420 ms
└ POST /orders            orders       ██████████████████████████   390 ms
  ├ SELECT stock          SQL Server    ███                          45 ms
  ├ POST /payments        payments          ████████████████        240 ms  ← slow
  └ publish order.created RabbitMQ                           █       15 ms
```

- **Instrument** (auto + manual; see OpenTelemetry below) → **export** over OTLP to an **OpenTelemetry Collector** (batch, sample, redact, route) → **store** per signal → **query, dashboard, alert**.
- **Context propagation** stitches services together: each outbound call carries the W3C `traceparent` header (trace ID, parent span ID, sampled flag); the callee continues the trace and stamps the trace ID on its logs.
- **Monitoring vs observability:** monitoring = predefined dashboards and alerts for anticipated failures ("error rate > 1%"). Observability = rich, high-cardinality context you can slice any way after the fact ("which tenant, build and region do the slow requests share?"). Monitoring says *that*; observability says *why*.

## Signals

| Signal | Shape | Best for | Watch out |
|---|---|---|---|
| **Logs** | timestamped events; structured JSON: level, message template, fields, `trace_id` | the detail of one event; errors with context | priciest per byte; free text can't be queried |
| **Metrics** | numeric time series: **counter** (only increases: requests), **gauge** (current value: queue depth), **histogram** (bucketed distribution: latency) | cheap fixed-cost aggregates → dashboards, alerts, long retention | no per-request detail; each label combination is a new series |
| **Traces** | tree of **spans** (operation, start/end, attributes, status) sharing a trace ID, linked parent → child | where time goes across services; critical path; fan-out | needs sampling, and propagation through every hop |
| **Profiles** | sampled stack traces: CPU, allocations, locks | which function burns CPU/memory | runtime-specific |

## Advanced

- **What to measure:** **RED** (rate, errors, duration) per service/endpoint; **USE** (utilization, saturation, errors) per resource — CPU, disk, connection pools, queues; Google SRE's **four golden signals** (latency, traffic, errors, saturation). Track success and error latency separately — fast failures flatter the numbers.
- **Percentiles, not averages:** an average hides the tail; track p50/p95/p99/p99.9 and alert on the percentile users feel.
- **Histograms aggregate; summaries don't:** averaging per-pod p99s is meaningless. Sum histogram buckets across instances, then take the quantile: `histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_seconds_bucket[5m])))`. Prometheus summaries precompute quantiles per instance and can't be merged; bucket boundaries limit accuracy (native/exponential histograms adapt).
- **Tail amplification under fan-out:** a request touching 100 backends, each slow 1% of the time, is slow 1 − 0.99¹⁰⁰ ≈ 63% of the time — the leaves' p99 becomes the user's median. Watch leaf p99.9; mitigate with hedged requests and tight timeouts.
- **Cardinality explosion:** every unique label set is a separate time series — a `user_id` label turns one metric into millions and topples Prometheus. Keep labels bounded (route template, status class, region); put high-cardinality fields in traces, logs or wide events.
- **Trace sampling:** **head-based** decides at the root (e.g. keep 5%) and propagates via the sampled flag — cheap and consistent, but blind to outcome, so most errors are dropped. **Tail-based** decides after the trace completes (keep every error and slow trace, sample the rest); every span of a trace must reach the same collector, which buffers them (route by trace ID with the load-balancing exporter) — memory and delay cost.
- **Exemplars:** histogram buckets carry a sample trace ID, so you jump from a p99 spike straight to a slow trace (Prometheus/OpenMetrics + Grafana).
- **Log cost control:** sample noisy info/debug logs (keep every error), drop health-check noise, tier retention (hot for days → object storage), never log payloads, secrets or PII.
- **SLOs and alerting:** SLI = good events / valid events (e.g. non-5xx responses under 300 ms); SLO = target (99.9% over 30 days); error budget = the remaining 0.1% (≈ 43 min/month). Alert on **symptoms** (budget burn), not causes like CPU. **Multi-window, multi-burn-rate:** page at burn rate ≥ 14.4 over 1 h *and* 5 min (2% of a 30-day budget in an hour) or ≥ 6 over 6 h; ticket at 1 over 3 days. Every page must be actionable — alert fatigue trains people to ignore the real one.
- **Outside-in:** **synthetic monitoring** (scripted probes from many regions; catches outages at zero traffic) and **RUM** (real-user telemetry from browsers/apps: page loads, Core Web Vitals, JS errors).
- **OpenTelemetry (OTel):** the CNCF vendor-neutral standard — **API** (what code calls), **SDK** (sampling, processors, exporters), **Collector** (receivers → processors → exporters; per-node agent and/or central gateway), **OTLP** protocol (gRPC 4317, HTTP 4318), **semantic conventions** (standard attribute names). **Auto-instrumentation** covers frameworks and clients with zero code; **manual** adds business spans/attributes (`order.id`, `tenant`). Instrument once, switch vendors by config.
- **Across queues:** HTTP propagation is automatic in most frameworks; messaging often isn't — inject `traceparent` into message headers on publish (RabbitMQ message headers, Kafka record headers) and extract on consume; batch consumers use **span links** to many producer spans.
- **Wide events ("observability 2.0", Honeycomb):** one wide structured event per request per service with hundreds of fields (user, build, feature flags, cache hit, DB time); metrics are derived at query time from columnar storage instead of pre-aggregated, so any field can become a new breakdown mid-incident.
- **eBPF telemetry:** kernel-level capture of network flows and HTTP/gRPC calls with no code changes (Cilium Hubble, Pixie, Grafana Beyla) — instant coverage, little business context.
- **Continuous profiling:** always-on, low-overhead sampling profilers in production (Pyroscope, Parca) — flame graphs per deploy; linked to spans, they explain *why* a span was slow. OTel is standardizing profiles as a signal.

## Trade-offs

- ✅ Faster detection and resolution (MTTR); novel failures debuggable without redeploying; SLO-driven decisions; vendor-neutral with OTel.
- ❌ Telemetry volume can rival the cost of the app's own infrastructure — sampling, retention and cardinality discipline are mandatory.
- ❌ Instrumentation overhead (CPU, allocations, egress) and engineering time; auto-instrumentation alone lacks business context.
- ❌ Logs, metrics and traces in separate tools without shared IDs means slow context switching — correlation is the real work.

## In the wild

Prometheus + Grafana (Mimir/Thanos at scale) · Loki, ELK, OpenSearch · Jaeger, Zipkin, Grafana Tempo · Datadog, New Relic, Honeycomb · Azure Monitor / Application Insights, AWS CloudWatch + X-Ray · .NET: `ILogger` (+ Serilog), `Activity` = span (`ActivitySource` = tracer), `Meter` for metrics, OpenTelemetry .NET; ASP.NET Core uses W3C trace context by default and, since .NET 8, emits `http.server.request.duration` · Node: OpenTelemetry JS (`@opentelemetry/sdk-node` + auto-instrumentations), pino for JSON logs · DB side: SQL Server Query Store, Postgres `pg_stat_statements` · RabbitMQ: built-in Prometheus plugin.

## Gotchas

- **Interpolated log messages** (`LogInformation($"Order {id}")`) flatten fields into text and allocate even when the level is off — use message templates (`"Order {OrderId}", id`).
- **Raw paths as labels** (`/orders/123`) explode cardinality — label by route template (`/orders/{id}`).
- **Broken propagation:** one hop (a queue, a background job, a proxy) that drops `traceparent` splits every trace in two.
- **Gauges show only the value at scrape time:** a queue-depth burst between scrapes never appears — also export counters (enqueued/dequeued totals) or a max over the interval.
- **Clock skew** between hosts can draw a child span before its parent — trust durations, not cross-host timestamps.

## Related

[Circuit Breaker](01-circuit-breaker.md) · [Service Mesh](02-service-mesh.md) · [Availability](../01-foundations/02-availability.md) · [Message Queues](../06-async-and-events/01-message-queues.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [API Gateways](../02-apis-and-communication/03-api-gateways.md)

## Read more

- [Monitoring Distributed Systems — Google SRE book](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Alerting on SLOs — Google SRE workbook](https://sre.google/workbook/alerting-on-slos/)
- [Observability primer — OpenTelemetry docs](https://opentelemetry.io/docs/concepts/observability-primer/)

<!-- nav -->

---

[← Service Mesh](02-service-mesh.md) · [Index](../README.md)
