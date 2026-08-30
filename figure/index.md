---
tags:
  - index
  - figure
  - robotics
  - sre
  - interview-prep
---

# Figure AI — Interview Prep Index

**Role:** Staff Site Reliability Engineer
**Company:** Figure AI (AI robotics, humanoid robots)
**Location:** San Jose, CA
**Compensation:** $175K–$250K base

---

## Start Here

- [[figure/company-context]] — What Figure builds, why infrastructure is uniquely hard in physical AI, and what "Staff SRE at a robotics startup" actually means
- [[figure/staff-sre-role]] — The job posting decoded: what they're really asking for, the subtext behind each requirement, questions to ask them

---

## Scenarios (Design Problems)

These are the types of open-ended design problems you should expect:

- [[figure/scenarios/robot-fleet-cicd]] — Design a CI/CD pipeline for deploying software to a fleet of humanoid robots in production
- [[figure/scenarios/robot-telemetry]] — Design the observability stack for robot fleet telemetry at scale
- [[figure/scenarios/saas-to-self-hosted]] — Migrate GitHub / Jira / Confluence to self-hosted alternatives without disrupting 300+ engineers

---

## Flashcards

- [[figure/flashcards/figure-sre-top15]] — 15 Q&A: robotics infra, hybrid cloud/on-prem, manufacturing SLOs, startup SRE mindset

---

## Cross-References (Existing KB)

| Topic | Link | Why Relevant |
|---|---|---|
| Linux fundamentals | [[sre/topics/linux-cli]] | On-prem systems; Figure runs Linux on robots |
| SLO / SLI / SLA | [[sre/concepts/slo-sli-sla]] | Must define SLOs for manufacturing systems |
| Networking | [[sre/concepts/networking-fundamentals]] | Hybrid cloud + on-prem networking |
| CI/CD pipeline | [[devops/concepts/ci-cd-pipeline]] | Robot deployment extends standard CI/CD |
| IaC | [[devops/topics/infrastructure-as-code]] | Terraform mastery required per JD |
| Observability | [[devops/concepts/observability-pillars]] | Prometheus/Grafana/Datadog per JD |
| Secrets management | [[devops/concepts/secrets-management]] | SaaS-to-self-hosted = own secrets |
| Incident response | [[devops/patterns/incident-response]] | Manufacturing incidents have physical consequences |
| Troubleshooting framework | [[sre/patterns/troubleshooting-framework]] | Core SRE skill; apply to robot incidents |
| Deployment strategies | [[devops/concepts/deployment-strategies]] | Staged rollout to robot fleets |
