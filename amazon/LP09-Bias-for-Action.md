# LP09 - Bias for Action

> Speed matters in business. Many decisions and actions are reversible and do not need extensive study. We value calculated risk taking.

---

## STAR Format

**Situation:** At PNC, during a Monday morning peak processing window, the alert/announcement Spring Boot service began returning intermittent 500 errors under load. Business teams making real-time risk decisions started seeing gaps in the alert feed. Initial triage pointed to Oracle connection pool exhaustion, but the exact trigger — a new batch reconciliation job another team had just introduced, a query regression, or a leak in our own service — was not yet confirmed.

**Task:** I was on the incident call. A full root-cause diagnosis — pulling query plans, session traces, and AWR reports — would take hours the business impact couldn't absorb. I had to decide whether to wait for certainty or act on a partial diagnosis.

**Action:** I chose mitigations that were deliberately reversible rather than waiting for the full picture: raised the connection pool ceiling as an immediate pressure valve, added an aggressive query timeout so requests would fail fast instead of piling up waiting threads, and asked the owning team to temporarily throttle the new batch job's fetch size. None of these were permanent fixes — they were low-risk, fast-to-undo moves chosen specifically because they could be rolled back in minutes if they made things worse, while the real investigation continued in parallel.

**Result:** The service stabilized within the incident window and the alert feed gap closed before it affected the next business decision cycle. The parallel investigation later confirmed the batch job was running a JPQL query with no supporting index, causing table-level lock contention. That fix was applied permanently, and the incident became the basis for a query-review checklist now required before any new JPQL is merged.

---

## SOAR Format

**Situation:** The alert/announcement service at PNC started failing intermittently under peak load, driven by Oracle connection pool exhaustion whose exact source was unclear — it could have come from several different places.

**Obstacle:** Diagnosing the precise root cause with certainty required hours of query-plan and session analysis, time the active business impact didn't allow. Acting on an unconfirmed diagnosis risked applying a change that didn't help, or masking the real signal entirely.

**Action:** I applied mitigations chosen for being reversible and low-risk — raising the connection pool ceiling, adding fail-fast query timeouts, and throttling the suspect batch job — all of which could be undone in minutes if wrong, while the actual root-cause investigation ran in parallel instead of being blocked on it.

**Result:** Service stabilized quickly and business impact was contained. The real cause — an unindexed JPQL query in the new batch job — was confirmed and permanently fixed once the time pressure was off, and the incident seeded a query-review checklist the team still uses before merging new JPQL.

---

*Tags: #amazon-lp #bias-for-action #pnc #production-incident #oracle #jpql #calculated-risk*
