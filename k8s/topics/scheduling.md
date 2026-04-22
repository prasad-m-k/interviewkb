# Kubernetes Scheduling

**Topic:** [[k8s/topics/scheduling]]
**Related:** [[k8s/topics/workloads]], [[k8s/concepts/hpa]]

## How the Scheduler Works

The scheduler is a control loop that watches for Pods with no `nodeName`. For each, it runs:

1. **Filtering** (predicates): eliminates nodes that cannot run the Pod
   - Insufficient CPU/memory
   - Node has a taint the Pod doesn't tolerate
   - Pod has a required node affinity the node doesn't satisfy
   - Volume zone constraints (a PVC is in a specific AZ)

2. **Scoring** (priorities): ranks remaining nodes
   - `LeastAllocated`: prefer nodes with more free resources
   - `NodeAffinity` weight: prefer nodes matching soft affinity
   - `PodTopologySpread`: prefer spreading Pods across zones/nodes

The highest-scoring node is selected; the scheduler writes `nodeName` to the Pod spec.

---

## Resource Requests vs. Limits

```yaml
resources:
  requests:
    cpu: "500m"       # what the scheduler uses for placement
    memory: "256Mi"   # guaranteed — node must have this free
  limits:
    cpu: "2"          # throttled at this value (not killed)
    memory: "512Mi"   # OOMKilled if exceeded
```

- **Requests** = what the scheduler reserves on the node; what `kubectl top` compares against
- **CPU limit**: enforced by the kernel cgroup — the container is throttled, not killed
- **Memory limit**: if exceeded, the container is **OOMKilled** immediately
- **No requests set** = scheduler treats them as 0; Pod can end up on an overcommitted node
- **No limits set** = Pod can consume all node resources (noisy neighbor problem)

**QoS Classes** (set automatically):
- `Guaranteed`: requests == limits for all containers
- `Burstable`: requests set but < limits
- `BestEffort`: no requests or limits set (evicted first under node pressure)

---

## Node Selector and Node Affinity

**nodeSelector** (simple label match):
```yaml
spec:
  nodeSelector:
    accelerator: nvidia-tesla-v100
```

**nodeAffinity** (richer expressions):
```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:  # hard constraint
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b]
    preferredDuringSchedulingIgnoredDuringExecution: # soft constraint
    - weight: 1
      preference:
        matchExpressions:
        - key: node-type
          operator: In
          values: [high-memory]
```

`IgnoredDuringExecution` = once scheduled, the constraint is no longer enforced (Pod won't be evicted if the node label changes).

---

## Taints and Tolerations

**Taints** repel Pods from nodes. **Tolerations** allow a Pod to be scheduled on a tainted node.

```bash
# Taint a node (e.g., reserve GPU nodes)
kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule
```

```yaml
# Toleration in Pod spec
tolerations:
- key: dedicated
  operator: Equal
  value: gpu
  effect: NoSchedule
```

**Effects:**
- `NoSchedule`: new Pods without toleration won't be scheduled
- `PreferNoSchedule`: soft version — avoid if possible
- `NoExecute`: new Pods won't schedule AND existing Pods without toleration are evicted

**Pattern for dedicated node pools** (e.g., GPU nodes):
1. Taint GPU nodes with `NoSchedule`
2. Add toleration to GPU workload Pods
3. Add `nodeAffinity` to GPU Pods to require the GPU nodes (toleration alone doesn't *force* scheduling there)

---

## Pod Affinity and Anti-Affinity

**podAffinity**: schedule near Pods with certain labels (e.g., co-locate a model server and its Redis cache)
**podAntiAffinity**: spread Pods across nodes/zones (e.g., ensure no two replicas on the same node)

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: model-server
      topologyKey: kubernetes.io/hostname   # "no two on same node"
```

---

## TopologySpreadConstraints

More flexible than podAntiAffinity. Specifies how Pods should be distributed across topology domains (zones, nodes, racks).

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: model-server
```

`maxSkew: 1` means at most 1 Pod difference between the most- and least-loaded zone.

---

## PodDisruptionBudget (PDB)

Limits voluntary disruptions (node drain, rolling update) to protect availability.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2          # or maxUnavailable: 1
  selector:
    matchLabels:
      app: model-server
```

If a drain would violate the PDB, `kubectl drain` blocks until the constraint is satisfiable.

---

## PriorityClass

Defines scheduling priority. Higher-priority Pods can preempt (evict) lower-priority ones.

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority-training
value: 1000000
globalDefault: false
```

Useful for ensuring production model serving always has resources over development training jobs.

## Sources
- [[K8S overview]]
