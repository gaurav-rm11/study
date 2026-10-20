# Availability

> **TL;DR** — Availability is the fraction of time — or of requests — a system serves correct, fast-enough responses. It's bought with redundancy and fast recovery, and each extra nine costs more than the last, so the target is a business decision.

## How it works

- **Time-based:** uptime ÷ (uptime + downtime). Simple, but ambiguous for partial outages — is "10% of requests failing" down?
- **Request-based:** good requests ÷ valid requests, where good = non-5xx *and* under a latency threshold. Weights outages by traffic; Google SRE's preferred measure.
- **Core techniques:** redundancy (N+1, no SPOF), automatic failover, health checks that pull bad instances from rotation, spreading across AZs and regions, graceful degradation — plus safe change: Google SRE attributes ~70% of outages to changes, so canary and roll back fast.

## The nines

| Availability | Downtime / year | Downtime / month (30 d) |
|---|---|---|
| 99% | 3.65 days | 7.2 h |
| 99.9% | 8.76 h | 43.2 min |
| 99.95% | 4.38 h | 21.6 min |
| 99.99% | 52.6 min | 4.32 min |
| 99.999% | 5.26 min | 25.9 s |

## SLI, SLO, SLA and error budgets

| Term | Meaning | Example |
|---|---|---|
| **SLI** (indicator) | a measured ratio of good events | % of checkout requests returning non-5xx in < 300 ms |
| **SLO** (objective) | internal target for an SLI over a window | 99.9% over a rolling 30 days |
| **SLA** (agreement) | contract with penalties, usually service credits | 99.5% — looser than the SLO, so you breach the SLO first |
| **Error budget** | 1 − SLO | 0.1% ≈ 43 min per 30 days for deploys, experiments, incidents; when it's spent, freeze risky releases |

## Advanced

- **Series vs parallel:** hard dependencies in **series** multiply — A = A₁ × A₂ × …; five 99.9% dependencies cap you at 0.999^5 ≈ 99.5%. Redundant copies in **parallel** (any one suffices): A = 1 − Π(1 − Aᵢ); two 99% nodes ≈ 99.99%. Example: LB 99.99% → two app nodes at 99.5% each (1 − 0.005² = 99.9975%) → one DB at 99.95% ⇒ 0.9999 × 0.999975 × 0.9995 ≈ **99.94%**. The single DB dominates; you can't beat your weakest hard dependency.
- **MTBF vs MTTR:** A = MTBF ÷ (MTBF + MTTR). One failure a month (MTBF ≈ 720 h) with 1 h to recover → 99.86%; same failure rate, 5-min recovery → 99.99%. Cutting MTTR (detection via SLO alerts, one-click rollback, automated failover) usually beats making failures rarer.
- **Active-active vs active-passive:** active-active serves from every site, so failover is just shifting traffic — but each site needs headroom (2 regions → each ≤ 50% busy) and concurrent writes need conflict handling or a home region per key. Active-passive standbys: **hot** (running, data in sync; seconds–minutes), **warm** (scaled-down copy running; minutes), **pilot light** (data replicated, compute off; tens of minutes), **cold** (restore from backups + IaC; hours).
- **RTO vs RPO:** RTO = how long until service is back; RPO = how much data, measured in time, you can lose. Async replication → RPO > 0 (unreplicated writes are lost on failover); sync replication → RPO ≈ 0 at the cost of write latency.
- **Failover pitfalls:** *split-brain* — both sides believe they're primary and accept writes (prevent with quorum + fencing); *never-tested failover* — the standby lacks capacity, config or data, and the runbook has rotted (drill it); *thundering herd* — on recovery every client reconnects and retries at once into cold caches (jittered backoff, ramp traffic back gradually).
- **Correlated failures break the math:** the parallel formula assumes independence, but replicas share the same deploy, config, AZ, power, certificate and kernel — one bad config push takes out every "redundant" copy. Stagger rollouts by AZ/region with bake time; validate config like code.
- **Blast-radius reduction:** *cells* — independent stack copies, each serving a slice of customers; *shuffle sharding* — each customer gets a random subset of workers (8 workers, 2 each → 28 combinations), so a poison customer fully takes down only those sharing its exact pair; *bulkheads* — separate thread/connection pools per dependency so one slow dependency can't exhaust them all.
- **Static stability:** the data plane keeps serving on last-known-good state when the control plane (config, discovery, autoscaling) is down, and capacity is pre-provisioned — run 3 AZs at ≤ 67% so losing one needs no new instances.
- **Degradation tactics:** serve stale cache when the origin fails (`stale-if-error`), switch off non-critical features with kill switches (recommendations, reviews), go read-only, shed low-priority load with 503s so core paths survive.
- **Availability ≠ reliability ≠ durability:** availability = can I use it now; reliability = does it work correctly over time (failure rate, MTBF); durability = will stored data survive. S3 Standard is designed for 11 nines (99.999999999%) durability but 99.99% availability, and its SLA promises 99.9%.
- **CAP's "A" ≠ SLA availability:** CAP availability means every non-failed node eventually answers during a partition — no time bound, no SLA. CP systems still reach high SLAs: Spanner picks C under partition yet offers 99.999% for multi-region instances.
- **Measure from the client's side:** server metrics miss DNS, TLS, CDN, LB and network failures and requests that never arrived. Add black-box synthetic probes from several regions plus real-user monitoring (RUM).
- **Burn-rate alerting:** page on how fast the error budget is burning, not on raw error spikes — a 14.4× burn over 1 h consumes 2% of a 30-day budget (Google SRE workbook).

## Trade-offs

- ✅ Each extra nine cuts allowed downtime 10×, protecting revenue and trust.
- ❌ Each extra nine costs disproportionately more: more redundancy, multi-region data, automation, slower releases.
- ❌ Redundancy and failover logic add failure modes of their own (false failovers, split-brain, flapping).
- ✅ Error budgets turn "ship faster vs stay up" into a number product and ops both accept.
- ❌ Nines beyond what users can perceive are wasted spend — their own Wi-Fi and mobile networks fail more often; derive the SLO from user needs and dependency limits.

## In the wild

Google SRE (SLOs, error budgets) · AWS Regions/AZs, Route 53 health-check failover, RDS/Aurora Multi-AZ · Azure Availability Zones, Front Door, Azure SQL auto-failover groups · SQL Server Always On Availability Groups · Postgres: Patroni, pg_auto_failover · MySQL InnoDB Cluster (Group Replication) · RabbitMQ quorum queues (Raft-replicated) · Netflix Chaos Monkey / Chaos Kong (region evacuation) · .NET: ASP.NET Core health checks (`MapHealthChecks`), Polly / `Microsoft.Extensions.Http.Resilience` · Datadog Synthetics, CloudWatch Synthetics, Pingdom for client-side probes.

## Gotchas

- 99.99% allows 4.3 min/month — less than it takes a human to get paged and log in; at four nines and up, only automated detection and failover count.
- A 99.9% month can be one 43-min outage at peak; request-based SLIs weight it by traffic, time-based ones don't.
- Every new hard dependency lowers your ceiling (series math) — make it soft with a timeout and a fallback.
- Retries multiply across layers: 3 attempts at each of 3 layers = 27 calls to the bottom service during an outage. Use retry budgets and backoff.
- A process that answers `/health` can still fail every real request (gray failure) — alert on SLIs, not liveness.
- Planned maintenance is downtime to users even when the SLA excludes it.

## Related

[SPOF](03-spof.md) · [Scalability](01-scalability.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Replication](../04-databases/04-replication.md) · [CAP Theorem](../04-databases/05-cap-theorem.md) · [Circuit Breaker](../08-microservices-and-reliability/01-circuit-breaker.md) · [Observability](../08-microservices-and-reliability/03-observability.md)

## Read more

- [Embracing Risk — Google SRE book](https://sre.google/sre-book/embracing-risk/)
- [Service Level Objectives — Google SRE book](https://sre.google/sre-book/service-level-objectives/)
- [Static stability using Availability Zones — AWS Builders' Library](https://aws.amazon.com/builders-library/static-stability-using-availability-zones/)

<!-- nav -->

---

[← Scalability](01-scalability.md) · [Index](../README.md) · [SPOF →](03-spof.md)
