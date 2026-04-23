# The Site Reliability Workbook

**Type:** Book
**Authors:** Beyer, Murphy, Rensin, Kawahara, Thorne (Google)
**Publisher:** O'Reilly, 2018
**Ingested:** 2026-04-22
**Topics covered:** [[sre/concepts/slo-sli-sla]], [[sre/topics/system-design]]

## Summary

The SRE Workbook is the practical companion to [[sre/sources/google-sre-book]]. Where the SRE Book defines the principles, the Workbook provides concrete, step-by-step playbooks for implementing those principles in real organizations. It includes case studies from Google and other companies, making it more applicable to non-Google environments.

The Workbook is particularly valuable for two audiences:
1. Engineers preparing for SRE interviews who need to speak concretely about *how* to implement SRE practices, not just what they are.
2. Engineers who have read the SRE Book but struggle to apply it because their org is resistant to the practices.

## Key Takeaways

- **Introducing SLOs to a resistant team:** Start with a single service, define an SLO for it, and demonstrate that SLO-based decisions reduce friction between ops and product. The first SLO convinces the rest of the org.
- **Alerting on burn rate (not thresholds):** Chapter 5 introduces multi-window burn rate alerting — alert only when the error budget is being consumed too fast, not whenever the error rate exceeds any threshold. This dramatically reduces false-positive alerts.
- **Error budget policy is organizational, not technical:** The technical part (computing error budget) is trivial. The hard part is getting product teams and SRE teams to agree in advance on what happens when the budget is exhausted. Write it down before the first incident.
- **Game Days / DiRT (Disaster Recovery Testing):** Intentionally inject failures in production (or a staging environment with production-like traffic) to validate runbooks and train on-call engineers. Chaos engineering before chaos engineering was a term.
- **Oncall rotation design:** Sustainable on-call requires: <2 actionable pages per shift, runbooks for every alert, shift handoff documents, and post-on-call retrospectives.
- **Tiered service model:** Not all services need the same SRE investment. Tier 1 (critical revenue path) gets full SRE partnership. Tier 2 (internal tooling) gets self-service SRE guidance. Tier 3 (dev tools) gets no SRE investment.

## What it Updated

- Informed [[sre/concepts/slo-sli-sla]] — multi-window burn rate alerting section
- Informed [[sre/companies/google]] — on-call practices; error budget policy; behavioral Q&A ("how do you push back when product wants to disable an SLO?")

## Most Interview-Relevant Sections

### Multi-Window Burn Rate (Chapter 5)
The key insight: a single threshold alert on error rate has a terrible signal-to-noise ratio. A 1% error rate alarm goes off at 2am for a single bad request and also goes off for a 10-minute outage — different severity, same alert.

Burn rate = how fast the error budget is being consumed relative to the budget window.

| Alert type | Condition | Severity | Response |
|---|---|---|---|
| Page now | 2% budget in 1h OR 5% in 6h | Critical | Wake someone up |
| Ticket | 10% budget in 3 days | Warning | Address in business hours |
| None | Budget on track | — | No alert |

Formula: `burn_rate = error_rate / (1 - SLO_target)`

If SLO = 99.9% (0.1% error budget) and current error rate is 1%:
`burn_rate = 0.01 / 0.001 = 10×`

At 10× burn rate, the monthly budget is consumed in 72 hours.

### Sustainable On-Call (Chapter 8)
Sustainable on-call is one of the most asked behavioral topics in SRE interviews. From the Workbook:

- **Cognitive load limit:** Each on-call shift should have no more than 2 actionable incidents. Beyond that, the engineer is in firefighting mode and cannot do root-cause analysis.
- **Runbook requirement:** Every alert must have a runbook. If there's no runbook, the alert should be removed or the runbook written before the next on-call rotation.
- **Post-on-call review:** After each on-call rotation, review all pages. Categorize: noisy (needs threshold tuning), actionable (needs runbook improvement), systemic (needs engineering fix).

## Sources
- [[sre/sources/google-sre-book]]
