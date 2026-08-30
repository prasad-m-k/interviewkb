---
tags:
  - index
  - devops
  - ci-cd
  - kubernetes
  - infrastructure
  - interview-prep
  - cloud
uid: 1b1eaa6e-119a-4554-92b9-670ffa7aa169

---

# DevOps — Index
Last updated: 2026-04-21

> DevOps is the practice of unifying software development and operations through automation, collaboration, and fast feedback loops. Interviews test both conceptual depth (architecture, trade-offs) and operational instincts (how you'd behave in a production incident).

## Overview
- [[devops/overview]] — Coverage map, interview strategy, how DevOps relates to SRE and MLOps

## Topics
- [[devops/topics/ci-cd]] — Pipelines, GitOps, branching strategies, artifact management
- [[devops/topics/containers-orchestration]] — Docker, Kubernetes, Helm, service mesh
- [[devops/topics/infrastructure-as-code]] — Terraform, Ansible, Packer, immutable infrastructure
- [[devops/topics/observability]] — Metrics, logs, traces, alerting, dashboards
- [[devops/topics/security-devsecops]] — Secrets, SAST/DAST, supply chain, zero-trust

## Concepts
- [[devops/concepts/ci-cd-pipeline]] — Stages, triggers, artifact promotion, branching strategies
- [[devops/concepts/kubernetes-architecture]] — Control plane, data plane, networking, scheduling, RBAC
- [[devops/concepts/deployment-strategies]] — Blue-green, canary, rolling, feature flags — trade-offs
- [[devops/concepts/gitops]] — Git as single source of truth; Argo CD / Flux reconciliation loop
- [[devops/concepts/observability-pillars]] — Metrics (RED/USE), logs (structured), traces (distributed), alerting
- [[devops/concepts/infrastructure-as-code]] — Declarative vs imperative, tool decision matrix (Terraform vs Ansible vs Pulumi), drift, IaC team patterns
- [[devops/concepts/terraform-state]] — State file internals, remote backends (S3/GCS/Azure), locking, `state mv`/`rm`/`import`, workspaces, split-state design
- [[devops/concepts/terraform-patterns]] — Variables, count vs for_each, dynamic blocks, lifecycle meta-arguments, data sources, providers, testing
- [[devops/concepts/ansible]] — Agentless config management; playbooks, roles, inventory, idempotency, Vault, Terraform+Ansible pairing
- [[devops/concepts/secrets-management]] — Vault, K8s secrets, SOPS, sealed secrets — never in Git
- [[devops/concepts/service-mesh]] — Sidecar proxies, mTLS, traffic shifting, Istio/Linkerd

## Patterns
- [[devops/patterns/zero-downtime-deployment]] — Combining deployment strategies; readiness probes; rollback triggers
- [[devops/patterns/incident-response]] — Detect → triage → mitigate → resolve → postmortem framework
- [[devops/patterns/terraform-module-design]] — Module directory structure, versioning, caller pattern, blast radius splitting

## Scenarios (Interview Questions)
- [[devops/scenarios/production-outage]] — "Your deploy just broke prod. Walk me through it."
- [[devops/scenarios/kubernetes-debugging]] — OOMKill, CrashLoopBackOff, pending pods — systematic debug
- [[devops/scenarios/sudden-traffic-spike]] — "Service is getting 10× traffic. What do you do?"
- [[devops/scenarios/secret-exposed]] — "A secret was committed to Git. Incident response."
- [[devops/scenarios/ci-cd-design-interview]] — "Design a CI/CD pipeline for a microservices app."
- [[devops/scenarios/high-latency-no-errors]] — "Latency spiked but error rate is zero. Debug."
- [[devops/scenarios/terraform-interview]] — Design TF for a new company; state corruption recovery; CFN migration; GitOps pipeline

## Flashcards
- [[devops/flashcards/devops-faang-top15]] — Top 15 DevOps interview Qs with full answers
- [[devops/flashcards/terraform-iac-top15]] — 15 Terraform & IaC Q&A: state, modules, for_each, lifecycle, Ansible, multi-env design

## Log
- [[devops/log]] — Append-only change log

## Sources
*(none yet)*
