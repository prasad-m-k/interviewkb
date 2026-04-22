# Scenario-Based Questions — SLO, SLI, SLA

**Topic:** [[sre/concepts/slo-sli-sla]]
**Created:** 2026-04-21

Read the scenario, formulate your answer, then expand the callout to check.

---

## Scenario 1
**Your team owns a payment API. The product team proposes an SLO of 99.99% availability. Your current measured availability over the last 90 days is 99.94%. What do you say?**

> [!note]- Answer
> Push back — 99.99% is aspirational, not grounded in data, and it sets the team up to fail.
>
> **The math problem:**
> - Current: 99.94% → ~26 min downtime/month
> - Proposed: 99.99% → ~4.3 min downtime/month
> - That's a 6× reduction in error budget with no corresponding reliability investment
>
> **What to say:**
> 1. "Our current baseline is 99.94%. Committing to 99.99% with no infrastructure changes means we'll be in permanent error budget exhaustion and unable to ship features."
> 2. "Let's set the SLO at 99.9% — tighter than our current baseline but achievable — and build a roadmap to 99.99% over two quarters as we invest in redundancy."
> 3. "The SLA to customers should be ≤ SLO. If we need to offer 99.99% in the SLA, we need 99.99%+ in practice first."
>
> **Key principle:** SLOs should be set based on measured baselines + realistic improvement trajectory. "Aspirational SLOs" just mean your error budget is always negative and the team is always firefighting instead of shipping.

---

## Scenario 2
**Your API has an SLO: 99.9% of requests return 2xx in < 300ms, measured over a 28-day rolling window. An incident on Monday took the service down for 55 minutes. How much error budget remains for the month? Should you halt feature releases?**

> [!note]- Answer
> **Step 1 — Calculate the monthly error budget:**
> ```
> 28 days = 28 × 24 × 60 = 40,320 minutes
> SLO = 99.9%  →  Error budget = 0.1% of 40,320 = 40.32 minutes
> ```
>
> **Step 2 — Calculate budget consumed:**
> ```
> Incident duration: 55 minutes
> Budget consumed: 55 / 40.32 = 136%
> ```
>
> The error budget is **fully exhausted** — you've consumed 136% of the monthly allowance in a single incident.
>
> **Should you halt releases?**
> Yes, by the SRE model, when the error budget is exhausted:
> - Freeze all non-critical feature releases
> - Redirect engineering effort to reliability improvements
> - Do not resume normal release velocity until the budget recovers (which takes time on a 28-day rolling window — as the incident falls out of the window)
>
> **What to communicate to stakeholders:**
> "We burned our entire monthly error budget in one incident. We're pausing feature releases and focusing on root cause remediation and prevention. We expect to resume normal velocity in approximately [X] days as the incident rolls off the 28-day window, assuming no further incidents."
>
> *See also:* [[sre/concepts/slo-sli-sla]]

---

## Scenario 3
**You're designing a new recommendation service. A PM asks you to write an SLO. What questions do you ask, and what does a well-formed SLO look like for this service?**

> [!note]- Answer
> **Questions to ask first:**
> 1. "Who calls this service and what do they do with the result?" (Synchronous user-facing path vs async batch enrichment changes everything)
> 2. "What's the latency tolerance? Does a slow recommendation block page load, or is it optional?"
> 3. "What does 'error' mean here — is a fallback to popular items an error or acceptable degradation?"
> 4. "Do we have historical data on usage patterns, traffic volume, and error rates?"
> 5. "What are the downstream SLAs we're bound by?"
>
> **SLI choices for a recommendation service:**
>
> | SLI | Measurement |
> |---|---|
> | Availability | % requests returning a valid (non-empty) recommendation list |
> | Latency | p99 response time (< 150ms is typical for sync user-facing) |
> | Freshness | % requests served from a model trained within last 24h |
> | Relevance | Click-through rate — harder to SLO-ize; use business metrics |
>
> **Well-formed SLO example:**
> ```
> Availability SLO: 99.5% of requests return a non-empty recommendation list
>                   over a 28-day rolling window.
>
> Latency SLO: 95% of requests respond in < 100ms;
>              99% respond in < 300ms,
>              measured over a 28-day rolling window.
> ```
>
> **What makes it well-formed:**
> - Specific SLI (what is measured)
> - A threshold (the number)
> - A time window (28-day rolling)
> - Based on user-facing behavior, not server uptime
>
> **Error budget from these SLOs:**
> - Availability: 0.5% of requests may fail → at 1000 RPS, ~432,000 failures/month allowed
> - Latency: 1% of requests may exceed 300ms

---

## Scenario 4
**Your SLO is 99.9%. You have a customer SLA of 99.5%. The service has been at 99.97% for 6 months. A new CTO asks: "Can we just set the SLA at 99.9% to match our actual performance and charge more?" How do you respond?**

> [!note]- Answer
> This is a business + reliability tradeoff question. The answer is nuanced.
>
> **The SRE argument for keeping the SLA gap:**
>
> The gap between SLO (99.9%) and SLA (99.5%) is intentional — it's a safety buffer.
>
> ```
> Actual performance: 99.97%
> SLO:                99.9%   ← internal target; triggers reliability work if breached
> SLA:                99.5%   ← customer contract; breach triggers penalties/credits
> ```
>
> - If you set the SLA at 99.9%, any incident that pushes you below 99.9% immediately triggers customer penalties — before your team even has time to respond
> - The current 0.4% gap gives you ~175 minutes/month of incident buffer before customer impact
> - Moving the SLA up by 0.4% reduces that buffer to ~4 minutes/month — essentially zero tolerance
>
> **Legitimate reasons to tighten the SLA:**
> - You've invested in redundancy that genuinely supports 99.9%
> - The product justifies the premium pricing and customers require it
> - You accept the engineering obligation to maintain 99.99%+ actual performance
>
> **What to recommend:**
> "We can explore this if we first invest in reliability improvements that give us a measured baseline of 99.99%+ — which gives us a buffer even at a 99.9% SLA. Jumping the SLA without improving the baseline is just converting reliability risk into financial and reputational risk."

---

## Scenario 5
**You have three services in a critical path: auth (99.9%), API gateway (99.95%), and your app (99.9%). A VP asks: "What's the availability of our end-to-end product?" What's your answer, and what do you recommend?**

> [!note]- Answer
> **The math:**
> ```
> Combined availability = 0.999 × 0.9995 × 0.999
>                       = 0.9975  →  99.75%
> ```
>
> The product is only as reliable as the chain — dependencies *multiply*, they don't average.
>
> **Translating to downtime:**
> - Auth alone (99.9%): ~43 min/month downtime
> - Combined (99.75%): ~108 min/month downtime — 2.5× worse
>
> **What to say to the VP:**
> "Our end-to-end availability is ~99.75%, even though each individual service is at 99.9%+. This is a structural property of dependent services — the math compounds. If we want to offer customers 99.9% end-to-end, we need each dependency at ~99.967% or better, or we need to add resilience patterns to break the dependency chain."
>
> **Recommendations:**
> 1. **Redundancy on the weakest link** — auth at 99.9% is the bottleneck; move it to 99.99%
> 2. **Circuit breakers + graceful degradation** — if auth is down, serve cached sessions for N minutes instead of failing 100% of requests
> 3. **Independent error budgets** — each service tracks its own budget; central dashboard shows combined
> 4. **Tiered SLOs** — not all features need auth; identify auth-optional paths and keep them serving during auth outages

---

## Scenario 6
**At 2am you get paged: "SLO breach in progress — error rate is 8%, SLO target is 0.1%." Walk through your incident response.**

> [!note]- Answer
> **Immediate (0–5 min): Triage and communicate**
> ```bash
> # Is this real or a monitoring bug?
> curl https://api.example.com/health
> # Check error rate in dashboards from multiple sources (avoid alert fatigue false alarms)
>
> # Who else knows?
> # Post to #incidents: "Investigating elevated error rate on payment-api. 8% error rate vs 0.1% SLO. ETA for update: 10 min."
> ```
>
> **Scoping (5–15 min): What, where, since when?**
> ```bash
> # When did it start?
> grep "ERROR" /var/log/app.log | tail -100 | awk '{print $1, $2}' | uniq -c
>
> # Which endpoints are affected?
> awk '$9 ~ /^5/ {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
>
> # All users or a subset?
> awk '$9 ~ /^5/ {print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
>
> # Any recent deploys?
> git log --oneline -10   # or check deployment pipeline
> ```
>
> **Mitigation before root cause (if deploy is suspected):**
> - If a deploy happened in the last 2 hours: **roll back first, investigate second**
> - A 5-minute rollback saves 2 hours of debugging at 2am
>
> **Diagnosis (15–30 min): Root cause hunt**
> ```bash
> journalctl -u payment-api -n 200 --no-pager | grep -E "ERROR|FATAL|Exception"
> top    # CPU, memory normal?
> df -h  # disk full?
> ss -s  # connection exhaustion?
> ```
>
> **Error budget impact: calculate in real-time**
> ```
> Incident duration so far: 20 min
> Monthly budget: 43 min (99.9% SLO)
> Budget consumed: 20/43 = 46%
> At current rate, budget exhausted in: 23 more minutes
> ```
>
> **Communication cadence:** update the incident channel every 15 minutes even if you have nothing new — "still investigating, no ETA yet" beats silence.
>
> **Post-incident:** file a postmortem with timeline, root cause, action items. Update the error budget dashboard. Review SLO if this is a systemic pattern.

---

## Scenario 7
**A product manager says: "We don't need SLOs — just make sure nothing breaks." How do you respond?**

> [!note]- Answer
> This is a common cultural pushback. The answer is about *why SLOs exist*, not just what they are.
>
> **The problem with "just don't break":**
> - "Nothing breaks" is unmeasurable — you can't tell if you're getting better or worse
> - It implies 100% availability, which is impossible and leads to over-engineering
> - It gives SREs no framework to say "this reliability investment is enough" — work expands infinitely
> - It creates conflict: engineers want to ship features; ops wants stability; no objective arbiter
>
> **What SLOs actually give you:**
> 1. **A shared contract**: PM, engineering, and ops agree on what "reliable enough" means
> 2. **An error budget**: transforms the reliability vs. velocity tension into math. When budget is healthy, move fast. When it's low, slow down. No more arguments.
> 3. **Prioritization**: "Should we fix this flaky test or add a new feature?" → check the error budget. If budget is healthy, ship the feature.
> 4. **Measurable progress**: you can demonstrate reliability improvements over time with data
>
> **Analogy that lands well in interviews:**
> "Saying 'just don't break' is like a finance team saying 'just don't spend money.' You need a budget — a number — to make tradeoff decisions. The error budget is that number for reliability."
>
> **How to close:**
> "Let's start with a simple SLO based on what users actually experience — something like 99.9% of checkout requests succeed within 2 seconds. That gives us a number to work toward and a framework for deciding when reliability work is done."

---

## Scenario 8
**You're asked to set an SLI for a batch ML training pipeline. The pipeline runs once per day, takes 4 hours, and produces a model artifact. What SLIs would you choose?**

> [!note]- Answer
> Batch pipelines are fundamentally different from request-serving systems — there are no requests per second to measure. The right SLIs measure *outcomes and timeliness*, not throughput.
>
> **Candidate SLIs:**
>
> | SLI | Definition | Why it matters |
> |---|---|---|
> | **Freshness** | Hours since last successful run completed | Model staleness directly impacts prediction quality |
> | **Success rate** | % of scheduled runs that complete without error over 30 days | Reliability of the pipeline itself |
> | **Completion time** | % of runs that complete within 6 hours (4h + 50% buffer) | Downstream consumers depend on the artifact being ready |
> | **Data completeness** | % of expected input records that were processed | A run can "succeed" but silently skip records |
>
> **Well-formed SLOs:**
> ```
> Freshness SLO:     The model artifact is never more than 30 hours old
>                    (allows for 1 missed run + recovery window).
>
> Success rate SLO:  ≥ 95% of scheduled daily runs complete successfully
>                    over a 28-day rolling window.
>
> Completeness SLO:  ≥ 99.5% of input records are processed in each run.
> ```
>
> **What NOT to use:**
> - Server uptime — the server can be up and the pipeline still silently fail
> - "Pipeline ran" without checking output quality — a run that produces a corrupt artifact is worse than no run
>
> **Interview key insight:**
> "For batch systems, freshness and completeness are the user-facing SLIs. Latency and availability only matter insofar as they determine whether those outcomes are met."
>
> *See also:* [[sre/concepts/slo-sli-sla]]

---

## Scenario 9
**Your on-call rotation is getting burned out — every week someone is paged 20+ times. Your SLO dashboards look green. What's going on, and how do you fix it?**

> [!note]- Answer
> This is the **alert fatigue / toil** problem — a classic SRE scenario.
>
> **What's happening:**
> The SLO dashboards are green because the SLOs are loose — the alerts are firing on symptoms that don't actually represent an SLO breach. Two common causes:
>
> 1. **Alerts are not SLO-based**: alerts are on low-level signals (CPU > 80%, disk > 70%) that aren't tied to user impact. They wake people up but don't represent real reliability problems.
> 2. **SLOs are too coarse**: the SLO tracks a 28-day window. An incident that burns 10 minutes of budget at 2am looks flat on a 28-day chart — but still woke someone up.
>
> **The fix: symptom-based alerting + burn rate alerts**
>
> *Symptom-based*: page only when users are being impacted. CPU at 80% is not user impact. Error rate at 2% is.
>
> *Burn rate alerts*: instead of alerting on the raw SLO window, alert on how fast the error budget is being consumed. Two-window burn rate:
>
> ```
> Fast burn (page immediately):
>   1-hour burn rate > 14.4×  AND  5-minute burn rate > 14.4×
>   → At this rate, 100% of monthly budget gone in 2 days
>
> Slow burn (ticket, not page):
>   6-hour burn rate > 6×
>   → At this rate, budget exhausted in ~5 days
> ```
>
> **Burn rate math:**
> ```
> 14.4× burn rate = consuming budget 14.4× faster than normal
> At 14.4×: 1-hour consumes 14.4/720 = 2% of monthly budget
> After 5 days at 14.4×: 100% consumed
> ```
>
> **Other fixes:**
> - **Reduce alert noise**: for every alert, ask "if I ignore this for 30 min at 3am, does it breach an SLO?" If no → remove or downgrade to a ticket
> - **Track toil**: if the same alert fires 3x in 30 days with the same manual fix → automate the fix or fix the root cause
> - **Error budget policy**: formally document "when budget < X%, on-call is allowed to silence non-critical alerts"

---

## Scenario 10
**You join a team where there are no SLOs, only "we try to have 99.9% uptime." How do you introduce SLOs without creating organizational resistance?**

> [!note]- Answer
> Introducing SLOs is as much a people problem as a technical one. Resistance comes from: fear of accountability, not understanding the purpose, and concern that the bar will be used punitively.
>
> **Step 1 — Start descriptive, not prescriptive**
> Don't start with targets. Start by measuring what you already have.
> ```bash
> # Pull 90 days of metrics, compute what your actual availability has been
> # Show it to the team: "This is what we're already delivering."
> ```
> Targets based on historical baselines feel fair, not arbitrary.
>
> **Step 2 — Frame it as protection, not surveillance**
> The error budget protects engineers: "When the budget is healthy, no one can ask you to cancel a vacation for an alert. When it's exhausted, we pause releases — this protects you from being blamed for a production incident that happened because someone pushed through a risky deploy anyway."
>
> **Step 3 — Set a deliberately achievable first SLO**
> Pick a target that's 10% tighter than your measured baseline. Set the window to 28 days. Let the team feel what "healthy error budget" feels like before tightening.
>
> **Step 4 — Wire the error budget to something the team controls**
> "When budget > 50%, we ship features. When budget < 25%, we dedicate one sprint to reliability. When exhausted, we freeze releases." Make the policy concrete and written down.
>
> **Step 5 — Never use SLO breach data for performance reviews**
> The fastest way to kill an SLO program: use it to blame people. SLOs measure system reliability, not engineer performance. Say this explicitly.
>
> **What to say in the interview:**
> "I'd start by measuring before targeting, frame the error budget as a feature-shipping enabler (not a punitive measure), set an achievable first SLO, and establish a written error budget policy so the team knows exactly what happens when it's spent."
