# SLO, SLI, SLA — Reliability Targets

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/patterns/troubleshooting-framework]]

## Definitions

| Term | Stands for | What it is |
|---|---|---|
| **SLI** | Service Level Indicator | A *measurement* — the actual metric you track |
| **SLO** | Service Level Objective | Your *target* for an SLI — the number you commit to internally |
| **SLA** | Service Level Agreement | A *contract* with a customer — breach triggers penalties |
| **Error budget** | — | `1 - SLO` — how much unreliability you're allowed |

**The relationship**: SLA ≤ SLO ≤ actual performance. Your SLO should be tighter than your SLA so you have a warning buffer before breaching the customer contract.

---

## SLIs — what to measure

Good SLIs are:
- **User-facing**: measure what users experience, not internal system health
- **Proportional**: expressed as a ratio (not a raw count)

| Service type | Typical SLIs |
|---|---|
| Request-based (API, web) | Availability (% successful requests), latency (p99 < Xms), error rate |
| Data pipeline | Freshness (age of most recent record), completeness (% records processed) |
| Storage | Durability (% of writes that are readable), throughput |
| Streaming | Throughput (events/sec), consumer lag |

**Good SLI example**: "99.5% of requests in the past 28 days return 2xx within 300ms"
**Bad SLI**: "server uptime > 99.9%" — server can be up but serving errors

---

## Error budget

```
SLO = 99.9%
Error budget = 100% - 99.9% = 0.1% of requests may fail

Over 28 days = 28 × 24 × 60 × 60 = 2,419,200 seconds
Allowed downtime = 0.001 × 2,419,200 = 2,419 seconds ≈ 40 minutes
```

**Why error budgets matter for interviews**: the error budget is the negotiating currency between reliability and velocity. 
- Budget is healthy → ship features aggressively
- Budget is nearly exhausted → freeze risky releases, focus on reliability
- Budget is exhausted → no new features until it recovers

This is why SRE orgs have the power to say "no deploys this week."

---

## Availability math — must memorize

| Availability | Downtime / year | Downtime / month | Downtime / week |
|---|---|---|---|
| 99% ("two nines") | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% ("three nines") | 8.76 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5 min |
| 99.99% ("four nines") | 52.6 min | 4.38 min | 1.01 min |
| 99.999% ("five nines") | 5.26 min | 26.3 sec | 6.05 sec |

**Interview shortcut**: 99.9% ≈ 40 min/month, 99.99% ≈ 4 min/month.

---

## Composite SLO — dependent services

If your service depends on multiple components, the combined availability is the *product* of each:

```
Service A: 99.9%
Database B: 99.9%
Cache C: 99.9%

Combined: 0.999 × 0.999 × 0.999 = 99.7%
```

**Interview implication**: adding dependencies degrades your SLO. This is why circuit breakers, graceful degradation, and caching matter.

---

## Toil and reliability work

**Toil**: manual, repetitive, automatable operational work that scales with service growth. Not engineering — no lasting value. SRE teams target <50% time on toil.

Examples of toil: manual deployments, manual capacity changes, handling the same alert repeatedly without a code fix.

---

## Common interview angles

- "What's the difference between SLI, SLO, and SLA?" — definitions + the SLA ≤ SLO relationship
- "How do you calculate an error budget?" — `1 - SLO`, in time or in request count
- "When would you halt feature deployments?" — when error budget is exhausted
- "A service has 99.9% availability. How much downtime is allowed per month?" — ~43 minutes
- "Your service depends on three 99.9% services. What's your realistic SLO?" — ~99.7%
- "What's a good SLI for a data pipeline?" — freshness + completeness, not server uptime

## Sources
*(none yet)*
