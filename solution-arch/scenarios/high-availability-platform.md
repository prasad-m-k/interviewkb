# Scenario: Designing a High-Availability Platform (99.99%)

**Concepts:** [[solution-arch/topics/nfr-quality-attributes]], [[solution-arch/patterns/circuit-breaker]], [[solution-arch/patterns/bulkhead]], [[solution-arch/patterns/blue-green-canary]]

---

## The Problem

Design an e-commerce platform with:
- 99.99% availability SLA (< 52 minutes downtime per year)
- 500k concurrent users at peak
- Global customers
- Zero-downtime deployments

---

## 1. Eliminate Single Points of Failure (SPOFs)

```
Every component must have redundancy:

Load Balancer:
  → 2+ LB instances; anycast DNS or floating VIP
  → Health checks every 5 seconds

Application Tier:
  → Minimum 3 instances per service
  → Deploy across 3 Availability Zones (AZ)
  → Auto-scaling: scale out when CPU > 60%

Database:
  → Primary + Synchronous standby (same AZ: very low lag)
  → Async replica in second AZ (read queries)
  → Automated failover (< 30 seconds)

Cache:
  → Redis Cluster (3 primary + 3 replica shards)
  → If one shard fails: failover to replica automatically

Message Broker:
  → Kafka: 3 brokers, replication factor = 3
  → min.insync.replicas = 2 (write must be confirmed by 2 brokers)

DNS:
  → TTL 60 seconds (fast failover)
  → Health-check-based DNS failover (Route 53 / CloudDNS)
```

---

## 2. Multi-AZ + Multi-Region Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                         Global DNS / GeoDNS                       │
└───────────────┬─────────────────────────────┬─────────────────────┘
                │                             │
    ┌───────────▼──────────┐      ┌───────────▼──────────┐
    │    Region: US-East    │      │    Region: EU-West    │
    │                      │      │                      │
    │  AZ-1  AZ-2  AZ-3   │      │  AZ-1  AZ-2  AZ-3   │
    │  [App] [App] [App]  │      │  [App] [App] [App]  │
    │  [DB]  [DB*] [DB*]  │      │  [DB]  [DB*] [DB*]  │
    │  *=replica           │      │  *=replica           │
    │         │            │      │         ▲            │
    └─────────┼────────────┘      └─────────┼────────────┘
              │  async replication          │
              └────────────────────────────┘
```

**Active-Active:** Both regions serve traffic. Writes replicate asynchronously.
**Active-Passive:** One region serves; other is hot standby. Simpler consistency; failover takes ~minutes.

**Decision:** Active-active for reads, active-passive for writes (only US primary accepts writes → EU reads from replica) unless global write throughput demands it.

---

## 3. Graceful Degradation

Design the system to work in a degraded state when components fail:

```
Normal:       Full feature set, real-time data
Degraded-1:   Cache fails → serve from DB (slower but functional)
Degraded-2:   DB read replica fails → serve from primary (higher load)
Degraded-3:   Recommendation engine down → show trending items (generic)
Degraded-4:   Payment gateway slow → queue payment async, confirm by email
Degraded-5:   Search service down → show category pages instead
Catastrophic: Show static maintenance page with status link
```

Feature flags control which features can be disabled under load.

---

## 4. Resilience Patterns

```
Circuit Breaker on all external dependencies:
  Payment Gateway: 5 failures in 60s → open circuit; return "try later" UX
  Inventory Service: circuit open → show "limited availability" message
  Email Provider: circuit open → queue emails; send when recovered

Bulkhead isolation:
  Payment thread pool: 20 threads
  Inventory thread pool: 30 threads
  Search thread pool: 50 threads
  Slow search doesn't consume payment threads

Retry with exponential backoff + jitter:
  Attempt 1: immediate
  Attempt 2: wait 100ms + rand(0-50ms)
  Attempt 3: wait 200ms + rand(0-50ms)
  Give up: return graceful error

Timeout on every network call:
  Internal services: 200ms timeout
  External APIs: 2s timeout
  DB queries: 5s timeout (slow query alert at 1s)
```

---

## 5. Deployment Strategy

```
Goal: deploy without any downtime

Strategy: Canary Deployment
  Deploy to 5% of traffic → monitor for 10 minutes
  Error rate or P99 latency regresses → auto-rollback to 0%
  Metrics look good → promote to 25% → 50% → 100%

Database migrations must be backward compatible:
  Never: rename/drop column in same deploy that switches traffic
  Use: expand-migrate-contract pattern over 3 deploys

Zero-downtime DB schema change example:
  Deploy 1: Add new column (nullable)
  Deploy 2: Backfill + application writes to both columns
  Deploy 3: Remove old column (after all traffic reads new column)
```

---

## 6. Observability for HA

```
You can't maintain what you can't observe.

SLOs (Service Level Objectives):
  Availability:  99.99% requests return non-5xx in rolling 30 days
  Latency:       P99 < 500ms for checkout API
  Error rate:    < 0.1% of requests result in errors

Alerts:
  Page immediately:  Error rate > 1% for 5 minutes
  Page immediately:  P99 latency > 2s for 5 minutes
  Ticket:            Error budget > 20% consumed this week

Dashboards:
  Golden signals: Latency, Traffic, Errors, Saturation (USE/RED)
  Per service, per AZ, per region
  Real-time: Grafana + Prometheus or equivalent

Runbooks:
  Every alert has a runbook: what is it? why does it fire? how to fix?
  Review runbooks quarterly — stale runbooks are dangerous
```

---

## 7. Chaos Engineering

Test resilience by intentionally injecting failures in production (or staging that mirrors prod):

```
Experiments:
  Kill a random pod → does traffic shift? Does auto-heal work?
  Inject 500ms latency to DB → does circuit breaker protect other services?
  Kill an AZ → does traffic shift to surviving AZs?
  Fill disk on one node → does alerting fire before disk is full?
  Corrupt a Redis key → does cache miss fallback to DB work?

Process:
  1. Hypothesis: killing one pod should have < 0.1% error impact
  2. Run experiment in staging; if safe → run in prod off-peak
  3. Measure: error rate, latency during experiment
  4. Fix gaps: if hypothesis fails → fix the weakness
  5. Automate: run chaos experiments in CI pipeline (Chaos Monkey, LitmusChaos)
```

---

## 8. HA Checklist

```
Infrastructure:
✅ No SPOF in any layer (LB, app, cache, DB, queue)
✅ Multi-AZ deployment (minimum 3 AZs)
✅ Auto-scaling configured and tested
✅ Health checks on all services (liveness + readiness)
✅ Automated DB failover tested (failover drill quarterly)

Application:
✅ Circuit breakers on all external dependencies
✅ Timeouts on all network calls
✅ Retries with exponential backoff + jitter
✅ Graceful degradation paths defined for each critical service
✅ Idempotent APIs and consumers

Operations:
✅ SLOs defined and dashboards live
✅ Alerts cover all SLO-impacting conditions
✅ Runbooks for every alert
✅ Canary deployments; rollback < 5 minutes
✅ Chaos engineering schedule (monthly minimum)
✅ Incident retrospectives documented and action items tracked
```

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
