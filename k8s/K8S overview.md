# Kubernetes Knowledge Base — Overview

**Audience:** Senior MLOps / Platform engineers preparing for FAANG / cloud-native company interviews
**Last updated:** 2026-04-21

---

## Why K8s Interviews Are Different

Kubernetes interviews at senior level are almost entirely **scenario-based**. Interviewers don't ask "what is a Pod?" — they describe a broken production cluster and ask what you do next. The gap between junior and senior K8s candidates is:

- Junior: knows the objects and can write YAML
- Senior: can reason about *why* something is broken, trace through the control plane, and explain trade-offs between design choices

**The mental model you need:** Kubernetes is a control loop system. Every controller watches desired state vs. actual state and reconciles. Every debugging problem is: which component is responsible for this reconciliation, and what is it seeing?

---

## Control Plane Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Control Plane                      │
│                                                     │
│  kubectl ──▶ API Server ──▶ etcd                    │
│                  │                                  │
│           Scheduler  Controller Manager              │
│           (watch + assign)  (reconcile loops)       │
└─────────────────────────────────────────────────────┘
                   │  (kubelet pulls desired state)
┌─────────────────────────────────────────────────────┐
│                  Worker Nodes                        │
│                                                     │
│  kubelet ──▶ Container Runtime ──▶ Pods             │
│  kube-proxy ──▶ iptables / ipvs rules               │
└─────────────────────────────────────────────────────┘
```

| Component | Role | What breaks if it's down |
|---|---|---|
| **API Server** | Central hub; validates and persists all state to etcd | Nothing can be created/updated/read; existing pods keep running |
| **etcd** | Distributed KV store; source of truth | API server can't read/write state; cluster is read-only at best |
| **Scheduler** | Watches unscheduled Pods; assigns to nodes | New Pods stay Pending forever; existing pods unaffected |
| **Controller Manager** | Runs all built-in controllers (ReplicaSet, Deployment, etc.) | Desired-state reconciliation stops; manual changes still apply |
| **kubelet** | On each node; ensures Pods run as specified | Pods on that node stop being managed; node goes NotReady |
| **kube-proxy** | Manages iptables/ipvs rules for Service routing | New Services don't get routing rules; existing connections may persist |

---

## Object Hierarchy

```
Namespace
└── Deployment (desired state: N replicas of this Pod template)
    └── ReplicaSet (current state: X Pods matching this template)
        └── Pod (one instance: one or more containers)
            ├── Container (runs image, has resource limits)
            ├── Volume (storage mount)
            └── ServiceAccount (identity for API calls)
```

---

## Core Interview Topic Map

| Interview Area | Key Concepts | Scenario Types |
|---|---|---|
| **Debugging** | Pod lifecycle, events, logs | CrashLoopBackOff, Pending, OOMKilled |
| **Networking** | Service types, CoreDNS, Ingress, NetworkPolicy | Service unreachable, DNS failure, Ingress misconfiguration |
| **Storage** | PV/PVC/StorageClass, access modes | PVC stuck Pending, StatefulSet mount failures |
| **Security** | RBAC, ServiceAccount, Secrets, PSA | Privilege escalation, secret rotation, least-privilege |
| **Scheduling** | Affinity, taints/tolerations, resources | Pod can't schedule, GPU nodes unused, hot nodes |
| **Scaling** | HPA, VPA, KEDA, Cluster Autoscaler | HPA not scaling, CA not adding nodes |
| **Reliability** | PDB, rolling updates, probes | Zero-downtime deploy, readiness misconfiguration |
| **MLOps-specific** | GPU scheduling, Jobs, Operators | Training job stuck, model serving OOM, multi-tenant isolation |

---

## The 5 Questions That Unlock Any K8s Debug

1. **What does `kubectl describe <resource>` say?** — Events section is always first.
2. **What do the container logs say?** — `kubectl logs <pod> [--previous]`
3. **What does `kubectl get events` show?** — Cluster-wide timeline of what happened.
4. **Is the desired state reachable?** — Resources, node selectors, taints, image availability.
5. **Which controller is responsible?** — Find the controller watching this object, check its health.

---

## Troubleshooting Reference

- [[troubleshooting-mindmap]] — Visual mindmap of all major failure modes with quick symptom → scenario routing
