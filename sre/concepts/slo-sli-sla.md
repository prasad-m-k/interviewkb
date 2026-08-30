# SLO, SLI, SLA — Reliability Targets

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[sre/patterns/troubleshooting-framework]], [[sre/concepts/escalation-management]], [[sre/concepts/rca-basics]], [[devops/patterns/incident-response]]

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

## SLO examples — concrete, internal targets

An SLO is the number your OWN team commits to hitting, measured over a rolling window, and used to decide whether to ship features or freeze and focus on reliability. A few worked examples across different service shapes:

```
1. Checkout API (request-based, latency + availability)
   SLO: 99.9% of checkout requests return 2xx within 500ms,
        measured over a rolling 28-day window.
   Why this shape: checkout is revenue-critical and user-facing —
   both availability AND latency matter, so the SLI is a compound
   condition (successful AND fast), not just "didn't error."

2. Nightly ETL pipeline (data freshness)
   SLO: 99.5% of daily batch runs complete and publish updated
        data before 06:00 UTC, measured over a rolling 90 days.
   Why 90 days, not 28: batch jobs run once a day, so a 28-day
   window only has ~28 data points — too few to be statistically
   meaningful. A longer window smooths out noise from any single
   bad day.

3. Internal search/autocomplete (latency-only, no strict
   availability requirement)
   SLO: p99 latency < 150ms for 99% of queries over a rolling 7
        days; availability tracked separately at 99.5%.
   Why split into two SLOs: autocomplete degrading to "slow" is a
   different user-experience failure than autocomplete being fully
   down — tracking them as one blended number would hide which
   failure mode is actually happening.

4. Message queue consumer (lag-based, not request-based)
   SLO: consumer lag stays under 60 seconds for 99.9% of 5-minute
        windows over a rolling 28 days.
   Why lag instead of throughput: a consumer can be "up" and still
   fall behind if producer volume spikes — lag is the SLI that
   actually reflects user-visible staleness of downstream data.
```

## SLA examples — external, contractual commitments

An SLA is what you promise a CUSTOMER, in a contract, with a consequence (usually a service credit) if you miss it. It should always be looser than your internal SLO — the SLO is your early-warning buffer before you'd actually breach the SLA.

```
1. Cloud API uptime SLA (the classic SaaS shape)
   SLA: 99.9% monthly uptime. Below 99.9% → 10% service credit for
        that month. Below 99.0% → 30% credit. Below 95.0% → 100%
        credit (full refund of that month's fees).
   Internal SLO backing it: 99.95% — gives the team a ~26-minute
   monthly buffer to detect and fix issues before the CUSTOMER-
   facing contract is actually at risk.

2. Support response-time SLA (not an availability SLA at all)
   SLA: Sev-1 tickets acknowledged within 15 minutes, 24/7;
        Sev-2 within 4 business hours; Sev-3 within 2 business days.
   Why this matters for interviews: SLAs aren't always about uptime
   — a response-time SLA on SUPPORT is just as real a contractual
   commitment, and violating it has the same credit/penalty shape.

3. Data processing SLA (freshness + completeness, contractual)
   SLA: 99.5% of daily files are processed and available to the
        customer by 08:00 UTC; any batch missing this triggers a
        service credit AND a mandatory customer notification within
        1 hour of the missed deadline.
   Note the extra clause: SLAs often bundle a COMMUNICATION
   obligation (notify within 1 hour) on top of the technical target
   — missing the notification can itself be a separate breach, even
   if the data eventually arrives late but complete.

4. Enterprise on-prem software SLA (patch/security response)
   SLA: Critical security vulnerabilities patched within 30 days
        of disclosure; high-severity within 90 days. Breach →
        contractual right for the customer to terminate without
        penalty.
   Why this is worth knowing: not every SLA is about a live
   service's uptime — some are about how fast you respond to a
   security disclosure, which is its own SLO/SLA pair engineering
   and security teams jointly own.
```

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
- "Give an example of an SLA that isn't about uptime" — a support response-time SLA, or a security-patch SLA

## Sources
- [[sre/concepts/escalation-management]] — what happens when an SLO/SLA is at risk of breaching during an active incident
- [[sre/concepts/rca-basics]] — the methodology for finding why an SLO was missed
- [[devops/patterns/incident-response]] — the incident lifecycle an SLO breach typically triggers
