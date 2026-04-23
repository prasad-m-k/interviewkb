# Site Reliability Engineering (The Google SRE Book)

**Type:** Book
**Authors:** Beyer, Jones, Petoff, Murphy (Google)
**Publisher:** O'Reilly, 2016
**Ingested:** 2026-04-22
**Topics covered:** [[sre/topics/linux-cli]], [[sre/topics/system-design]], [[sre/concepts/slo-sli-sla]]

## Summary

The Google SRE Book is the foundational text of the Site Reliability Engineering discipline. Written by Google engineers who built and ran Google's production systems for over a decade, it defines the principles, practices, and culture that distinguish SRE from traditional operations.

The book is organized in four parts: Introduction (what SRE is and why), Principles (SLOs, error budgets, toil, monitoring), Practices (on-call, incident response, postmortems, testing, capacity planning), and Management (organizational structure, team health).

The key intellectual contribution is framing **reliability as a software problem** — the SRE team's primary tool is software automation, not heroics. This has profound organizational implications: SREs are software engineers who spend up to 50% of their time on on-call/operational work, and any excess beyond 50% is treated as a bug to be engineered away.

## Key Takeaways

- **SLOs are the contract between SRE and product teams.** The error budget (1 - SLO) is shared capital: product teams spend it on risk, SRE teams spend it on changes. When it's exhausted, new features stop until the budget is restored.
- **Toil is the enemy.** Any manual, repetitive, automatable operational task is toil. Toil should be below 50% of an SRE's time. If it exceeds 50%, the team is effectively an operations team, not an engineering team.
- **Blameless postmortems are essential.** When blame is assigned, engineers hide information to protect themselves. A blameless culture surfaces information, which leads to systemic fixes rather than finger-pointing.
- **On-call is a production safety net, not a heroism showcase.** Being paged frequently is a system health metric, not a badge of honor. High page rates are incidents to be engineered away.
- **The four golden signals:** Latency, Traffic, Errors, Saturation. Monitor these for every service. Any other metric is supplementary.
- **Capacity planning is engineering work.** Demand forecasting, supply provisioning, and efficiency (utilization targets) should be planned annually with quarterly reviews — not handled reactively.
- **Configuration is code.** Configuration changes cause a significant fraction of production incidents. Treat config changes with the same rigor as code changes: version control, review, staged rollout.

## What it Updated

- Created [[sre/concepts/slo-sli-sla]] — core SLI/SLO/SLA framework, error budget math, burn rate alerting
- Informed [[sre/companies/google]] — SRE-SWE vs SRE-PE tracks; Googliness behavioral values; NALSD context
- Informed [[sre/topics/system-design]] — multi-window burn rate; toil definition; capacity planning section

## The Four Golden Signals (Quick Reference)

| Signal | What it measures | Example SLI |
|---|---|---|
| Latency | Response time of successful requests | p99 < 200ms |
| Traffic | Demand on your system | Requests per second |
| Errors | Rate of failed requests | HTTP 5xx / total × 100 |
| Saturation | How "full" your system is | CPU utilization < 80% |

## Chapters Most Relevant for Interviews

- **Chapter 3: Embracing Risk** — SLO framing, error budget concept, why 100% availability is wrong
- **Chapter 4: Service Level Objectives** — How to define SLIs, avoid common mistakes in SLO definition
- **Chapter 5: Eliminating Toil** — Toil definition, why it matters, how to quantify it
- **Chapter 6: Monitoring Distributed Systems** — Four golden signals, alerting philosophy
- **Chapter 11: Being On-Call** — Cognitive load, handoff practices, alert routing
- **Chapter 12: Effective Troubleshooting** — Scientific method for debugging, how to write a good hypothesis
- **Chapter 15: Postmortem Culture** — How to run a blameless postmortem, template
- **Chapter 22: Managing Critical State** — Data reliability, distributed consensus, split brain

## Companion: The SRE Workbook

[[sre/sources/sre-workbook]] — Practical implementation guide. Where the SRE Book defines principles, the Workbook gives concrete playbooks: how to introduce SLOs in a team that doesn't have them, how to structure an on-call rotation, how to run a Game Day.
