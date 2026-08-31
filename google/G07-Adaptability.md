# Googliness — Adaptability

> What Google is testing: comfort with ambiguity and change, and calculated speed — moving fast on a reversible or time-boxed decision without waiting for perfect conditions, while still getting the precision-critical details right.

**Traits emphasized:** Proactive · Goal oriented
**Adapted from:** [[amazon/LP09-Bias-for-Action]]

---

## STAR Format — Stabilizing a Production Service Without a Confirmed Root Cause

**Situation:** At PNC, during a Monday morning peak processing window, the alert/announcement Spring Boot service started returning intermittent 500 errors under load. Business teams making real-time risk decisions began seeing gaps in the alert feed. Triage pointed to Oracle connection pool exhaustion, but the exact trigger — a new batch reconciliation job another team had just introduced, a query regression, or a leak in our own service — wasn't yet confirmed.

**Task:** I was on the incident call. A full root-cause diagnosis — query plans, session traces, AWR reports — would take hours the active business impact couldn't absorb. I had to decide whether to wait for certainty or act on a partial diagnosis.

**Action:** I applied mitigations chosen specifically because they were reversible and low-risk: raised the connection pool ceiling as an immediate pressure valve, added an aggressive query timeout so requests failed fast instead of queuing, and asked the owning team to temporarily throttle the new batch job's fetch size — all while the real investigation continued in parallel instead of waiting on it.

**Result:** The service stabilized within the incident window and the alert feed gap closed before it hit the next business decision cycle. The parallel investigation later confirmed an unindexed JPQL query in the new batch job as the actual cause; that fix was applied permanently, and the incident became the basis for a query-review checklist now required before any new JPQL is merged.

**Decision · Reasoning · Outcome**
- **Decision:** Apply reversible mitigations immediately instead of waiting for a confirmed root cause.
- **Reasoning:** The cost of continued alert-feed gaps was accruing in real time, and every mitigation chosen could be undone in minutes if it turned out to be wrong — the downside of acting was bounded, the downside of waiting was not.
- **Outcome:** Service stabilized quickly, and the actual root cause was found and permanently fixed once the time pressure was off.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Estimated MTTR cut from ~2–3 hours (a root-cause-first response) to under 30 minutes by mitigating first and diagnosing in parallel.
- The resulting query-review checklist is estimated to have caught 2–3 similar unindexed-query issues before merge in the following quarter.

**Why this reads L6/L7:** The judgment call was distinguishing "reversible, bounded-risk action" from "waiting for certainty" under live production pressure — and then converting the incident into a permanent process change (the checklist) instead of just closing the ticket. That combination of judgment-under-ambiguity and systemic follow-through is what separates a Staff-level incident response from a competent one.

## Sample interview questions this answers
- "Tell me about a time you had to make a decision with incomplete information." (Problem Solving)
- "Describe a time you had to act quickly without all the information you wanted."
- "Tell me about a production incident you handled and what you'd do differently."

---

*Tags: #Googliness #adaptability #bias-for-action #ambiguity #production-incident*
