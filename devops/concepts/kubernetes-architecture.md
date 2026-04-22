# Kubernetes Architecture

**Topic:** [[devops/topics/containers-orchestration]]
**Related:** [[devops/concepts/deployment-strategies]], [[devops/concepts/service-mesh]]

## The Big Picture

```
                    ┌──────────────────────────────────────┐
                    │           Control Plane              │
                    │                                      │
  kubectl/API ────▶ │  API Server                          │
                    │      │                               │
                    │  etcd (state) ◀── Scheduler          │
                    │                   Controller Manager │
                    └──────────────────────────────────────┘
                              │ (watches + reconciles)
                    ┌─────────┴────────────────────────────┐
                    │              Data Plane               │
                    │                                       │
                    │  Node 1          Node 2      Node 3  │
                    │  ┌──────┐        ┌──────┐    ┌─────┐ │
                    │  │kubelet│       │kubelet│   │kube │ │
                    │  │kube-  │       │kube-  │   │let  │ │
                    │  │proxy  │       │proxy  │   │...  │ │
                    │  │Pods   │       │Pods   │   │Pods │ │
                    │  └──────┘        └──────┘    └─────┘ │
                    └───────────────────────────────────────┘
```

---

## Control Plane Components

| Component | What it does | Interview importance |
|---|---|---|
| **kube-apiserver** | Single entry point for all K8s operations; validates + stores objects in etcd | Very high — all kubectl goes through here |
| **etcd** | Distributed KV store; holds the entire cluster state | High — if etcd dies, cluster is read-only. Back it up. |
| **kube-scheduler** | Assigns pods to nodes based on resource requests, node selectors, affinity/taints | Medium — understand scheduling constraints |
| **kube-controller-manager** | Runs reconciliation loops: ReplicaSet controller, Node controller, Job controller | Medium — "controllers watch state and reconcile" |
| **cloud-controller-manager** | Interfaces with cloud provider APIs (create LoadBalancer, attach volumes) | Low — know it exists |

---

## Node Components

| Component | What it does |
|---|---|
| **kubelet** | Agent on every node; receives pod specs from API server; starts/stops containers via CRI |
| **kube-proxy** | Manages network rules (iptables/IPVS) for Service routing on each node |
| **Container runtime** | Runs containers (containerd, CRI-O — Docker was removed in K8s 1.24) |

---

## The Reconciliation Loop (Most Important Concept)

```
Desired state (etcd) ──▶ Controller ──▶ Actual state (node)
       ▲                      │
       └──────────────────────┘
          (if drift → reconcile)
```

Every K8s controller does this:
1. Watch for changes to objects (via API server watch)
2. Compare desired state (spec) to actual state (status)
3. Take action to make actual = desired

This is why `kubectl apply` is idempotent — you just update desired state and controllers handle the rest.

---

## Scheduling: How Pods Get Placed on Nodes

```
Pod creation
     │
     ▼
Filtering (eliminates ineligible nodes):
  - Resource requests: node has enough CPU/memory?
  - NodeSelector / NodeAffinity: label matches?
  - Taints/Tolerations: pod tolerates node's taints?
  - PodAffinity/Anti-affinity: co-locate or spread pods?
     │
     ▼
Scoring (ranks eligible nodes):
  - Least requested (spread load)
  - Image locality (image already on node → faster start)
  - Pod topology spread
     │
     ▼
Bind pod to winning node
```

**Taint/Toleration pattern:**
```yaml
# Node taint: "only GPU workloads here"
kubectl taint nodes gpu-node gpu=true:NoSchedule

# Pod toleration: "I can run on GPU nodes"
tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

---

## Networking

### Service types

| Type | Scope | Use case |
|---|---|---|
| **ClusterIP** | Internal only | Pod-to-pod communication |
| **NodePort** | Exposes port on every node | Dev/testing; not for prod |
| **LoadBalancer** | Cloud-managed LB | Public-facing services in cloud |
| **ExternalName** | DNS alias | Route to external service |

### DNS
Every Service gets: `<name>.<namespace>.svc.cluster.local`  
Every Pod gets: `<pod-ip-dashes>.<namespace>.pod.cluster.local`

### Network flow (ClusterIP)
```
Client pod ──▶ kube-proxy (iptables) ──▶ Service ClusterIP ──▶ one of the pod endpoints
```

---

## RBAC

```yaml
# 1. Role (namespace-scoped) or ClusterRole (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

# 2. RoleBinding — attach role to a subject
kind: RoleBinding
subjects:
  - kind: ServiceAccount
    name: my-app-sa
    namespace: prod
roleRef:
  kind: Role
  name: pod-reader
```

**Principle of least privilege:** apps should only have the exact verbs/resources they need. Never `cluster-admin` for an app service account.

---

## Common Interview Q&A

**Q: What happens when you run `kubectl apply -f deployment.yaml`?**
1. kubectl authenticates to API server (kubeconfig cert/token)
2. API server validates the manifest, stores in etcd
3. Deployment controller sees the new Deployment, creates a ReplicaSet
4. ReplicaSet controller sees desired replicas > actual, creates Pods
5. Scheduler assigns each Pod to a Node
6. kubelet on that node pulls the image and starts the container
7. kube-proxy updates iptables rules when pod IP is assigned

**Q: How do you upgrade a Kubernetes cluster with zero downtime?**
A: Cordon + drain nodes one at a time (moves pods off the node), upgrade the node, uncordon. Use PodDisruptionBudgets to ensure enough replicas stay running during drain. Upgrade control plane first, then nodes.

**Q: What is a CRD and when would you use one?**  
A: Custom Resource Definition — extends the K8s API with your own object types. Paired with a custom controller (operator pattern), it lets you manage complex stateful applications (databases, message queues) using the K8s reconciliation model.

## Sources
- [[devops/topics/containers-orchestration]]
- [[devops/overview]]
