# K8s Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" k8s/log.md | tail -10`

---

## [2026-09-01] update | Multi-master control plane leader election
- User asked how a multi-master Kubernetes cluster elects its leader. Existing architecture.md mentioned etcd uses Raft in passing but had no coverage of HA/multi-master control-plane election specifically (scheduler/controller-manager replicas, or that the API server has no election at all).
- Added "High Availability — How a Multi-Master Control Plane Elects Its Leaders" section to topics/architecture.md: three separate mechanisms distinguished explicitly — (1) etcd's own Raft-based leader election, (2) kube-scheduler/kube-controller-manager's lease-based election via client-go's leaderelection package and a coordination.k8s.io/v1 Lease object (etcd resourceVersion CAS, not Raft directly), (3) kube-apiserver's active-active design with no election.
- Cross-linked to the new solution-arch/concepts/raft.md and solution-arch/concepts/leader-election.md (which owns the general Bully/Ring/lease-based-election theory) rather than re-deriving consensus theory here — this page stays the concrete Kubernetes instance of that theory.

---

## [2026-04-21] update | Kubernetes knowledge base initialized
- Created directory structure: k8s/, topics/, concepts/, scenarios/, flashcards/, companies/
- Created: k8s/index.md, k8s/log.md, k8s/overview.md
- Created topics: architecture, workloads, networking, storage, security, scheduling, observability
- Created concepts: pod, deployment, service, ingress, configmap-secret, rbac, persistent-volume, hpa, statefulset, operator-crd
- Created scenarios: crashloopbackoff, pod-pending, oomkilled, service-unreachable, deployment-rollout-stuck, zero-downtime-deployment, gpu-scheduling, multi-tenant-isolation, hpa-not-scaling, secret-rotation
- Created flashcards: k8s-scenarios (Obsidian), k8s-scenarios-anki.txt (Anki TSV, 35+ cards)
- Domain: Kubernetes for senior MLOps/platform engineers at FAANG
