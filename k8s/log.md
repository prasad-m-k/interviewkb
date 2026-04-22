# K8s Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" k8s/log.md | tail -10`

---

## [2026-04-21] update | Kubernetes knowledge base initialized
- Created directory structure: k8s/, topics/, concepts/, scenarios/, flashcards/, companies/
- Created: k8s/index.md, k8s/log.md, k8s/overview.md
- Created topics: architecture, workloads, networking, storage, security, scheduling, observability
- Created concepts: pod, deployment, service, ingress, configmap-secret, rbac, persistent-volume, hpa, statefulset, operator-crd
- Created scenarios: crashloopbackoff, pod-pending, oomkilled, service-unreachable, deployment-rollout-stuck, zero-downtime-deployment, gpu-scheduling, multi-tenant-isolation, hpa-not-scaling, secret-rotation
- Created flashcards: k8s-scenarios (Obsidian), k8s-scenarios-anki.txt (Anki TSV, 35+ cards)
- Domain: Kubernetes for senior MLOps/platform engineers at FAANG
