# Single Point of Failure (SPOF)

> **TL;DR** — A single point of failure is any component whose failure alone takes down the system or one of its critical paths. Removing it takes redundancy plus tested, automatic failover — and the most dangerous ones are dependencies nobody drew on the diagram.

## How it works

```text
SPOF:       Client ─► LB ─► App ─► DB          any one box dies → outage
Redundant:  Client ─► LB ×2 ─► App ×N ─► DB primary ══► standby
                      (VIP)    (2+ AZs)  failover: quorum + fencing
```

- **The test:** "if this one thing dies — or just gets slow — does the request fail?" Yes → SPOF.
- **Redundancy:** with N = instances needed at peak, run **N+1** to survive one failure, **N+2** to survive a failure during a deploy or maintenance.
- **Stateless parts:** run copies across failure domains (host → rack → AZ → region) behind health-checked LBs.
- **Stateful parts:** replicate the data and automate failover — promote a replica, re-point clients.
- **Remove the dependency:** cache it, make it async, or degrade without it.

## Classic SPOFs

| On the request path | Fix |
|---|---|
| Single load balancer | active-passive pair on a floating VIP (keepalived/VRRP), or a managed multi-AZ LB |
| Single app server | N+1 stateless instances across AZs, health-checked |
| Single DB primary | standby + automatic failover; writes still stop for the failover window |
| Single cache node | replicas + Redis Sentinel/Cluster; size the DB to survive a cold cache |
| DNS provider | two authoritative DNS providers (the 2016 Dyn DDoS took down single-provider sites), sane TTLs |
| One AZ / region | multi-AZ by default; multi-region for critical paths |
| Message broker | 3+ node cluster — RabbitMQ quorum queues; Kafka RF=3 with `min.insync.replicas=2` |
| Auth service | validate JWTs locally against cached signing keys (JWKS) so the IdP isn't on every request |

## Advanced

- **Hidden SPOFs (infra):** config/feature-flag service (services can't start without it), secrets manager/KMS (can't decrypt → can't boot), **TLS certificate expiry** (a full outage on a date known in advance — automate renewal, alert weeks ahead), DNS, **NTP/time** (skew breaks TLS validation, JWT `exp`/`nbf` checks and lease-based locks), a single network link/NAT gateway/VPN, the Kubernetes control plane/etcd (running pods keep serving, but nothing schedules, scales or rolls back).
- **Hidden SPOFs (process and people):** CI/CD (you can't ship the fix), a shared library or base image (one bad version everywhere), cloud account and per-region quotas (failover can't launch capacity), operator SSO, and the one person who understands the system (**bus factor**). Facebook's 2021 outage: a backbone maintenance command cut off its DNS — and the internal tools needed to fix it ran on the same network.
- **Correlated failures:** identical replicas are N copies of the same bug — a poison request, bad deploy or config push hits them all at once (Cloudflare 2019: one WAF regex pushed globally pinned CPUs everywhere; CrowdStrike 2024: one content update crashed Windows hosts worldwide). Defend with staggered rollouts (one cell/AZ/region at a time, with bake time), canaries with automatic rollback, config validated like code, and diversity (versions, vendors) where it's worth the cost.
- **Failover needs quorum + fencing:** a standby that promotes itself because it *can't see* the primary causes split-brain — two primaries, diverging writes. Decide leadership by majority (3 or 5 nodes, or 2 + a witness) via consensus (etcd, ZooKeeper, Raft), then fence the old primary: **STONITH** ("shoot the other node in the head" — power it off via IPMI or the cloud API), revoke its storage/network access, or **fencing tokens** (monotonically increasing epochs that storage checks to reject a stale leader).
- **Untested redundancy is false comfort:** standbys drift (capacity, config, data) and failover scripts rot. Run game days, chaos experiments (Netflix Chaos Monkey kills instances; Chaos Kong evacuates a region) and routine planned failovers.
- **Algorithmic SPOFs:** a 2PC coordinator that dies after *prepare* leaves participants blocked, holding locks, until it returns; a single leader for writes, scheduling or ID generation is a SPOF unless its state is replicated via consensus (Raft/Paxos) so a new leader resumes from the log. Singleton cron jobs → leader election with leases.
- **Hard vs soft dependencies:** hard = the request fails when the dependency does; soft = the request succeeds, degraded. Convert hard to soft with timeouts + fallbacks (cached or default values), circuit breakers, queues (accept now, process later) and last-known-good config. Slow is worse than dead: without timeouts a hung dependency ties up every thread.
- **Finding SPOFs:** draw the request path and the full dependency graph — DNS, auth, config, secrets, deploy pipeline, people. For each node ask: what if it dies? what if it's slow? how long does failover take, and when was it last tested? **FMEA** formalizes this: list failure modes, score severity × occurrence × detectability (risk priority number), fix the highest first.
- **Accepting SPOFs:** redundancy costs money and complexity. A single-instance internal tool, a nightly batch job or a single-cloud setup can be fine when expected downtime cost < redundancy cost — document it, know its RTO, test the restore.

## Trade-offs

- ✅ No SPOF on the critical path turns host failures, AZ loss and maintenance (rolling deploys, patching) into non-events.
- ❌ Redundancy multiplies cost — a hot standby doubles the stateful tier; multi-region roughly doubles everything.
- ❌ Failover machinery adds failure modes of its own: false failovers on network blips, split-brain, flapping.
- ❌ Zero-data-loss failover needs synchronous replication (higher write latency); async failover can lose the last seconds of writes.

## In the wild

keepalived/VRRP LB pairs · AWS ALB (spans ≥ 2 AZs), Route 53 failover records · RDS Multi-AZ, Aurora (6 copies across 3 AZs) · SQL Server Always On Availability Groups (WSFC quorum with a file-share or cloud witness) · Postgres: Patroni (etcd/Consul/ZooKeeper), pg_auto_failover · MySQL InnoDB Cluster · Redis Sentinel / Redis Cluster · RabbitMQ quorum queues · Pacemaker + Corosync (STONITH) · cert-manager / Let's Encrypt (ACME) auto-renewal · Chaos Monkey, AWS FIS, Azure Chaos Studio, Gremlin.

## Gotchas

- Two replicas in the same rack, AZ or power circuit, or behind one switch, are one failure domain.
- "Managed" ≠ redundant: a single-AZ RDS instance or single-node Redis is still a SPOF.
- Failover isn't instant: detection + election + DNS/connection re-routing (RDS Multi-AZ documents typically 1–2 min). Clients must retry and evict dead pooled connections.
- A cache becomes a SPOF once the DB can't carry full load without it — a cold cache after failover stampedes the DB.
- Interview trap: drawing one LB and one DB without saying how each is made redundant — expect "what happens when this dies?" for every box.

## Related

[Availability](02-availability.md) · [Load Balancing](../03-traffic-and-edge/02-load-balancing.md) · [Replication](../04-databases/04-replication.md) · [Consensus Algorithms](../07-distributed-coordination/02-consensus-algorithms.md) · [Distributed Locks](../07-distributed-coordination/01-distributed-locks.md) · [Two-Phase Commit](../07-distributed-coordination/03-two-phase-commit.md) · [Circuit Breaker](../08-microservices-and-reliability/01-circuit-breaker.md)

## Read more

- [More details about the October 4 outage — Meta Engineering](https://engineering.fb.com/2021/10/05/networking-traffic/outage-details/)
- [Details of the Cloudflare outage on July 2, 2019](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)

<!-- nav -->

---

[← Availability](02-availability.md) · [Index](../README.md) · [Stateful vs Stateless →](04-stateful-vs-stateless.md)
