# Scenario: Pod Stuck in Pending

**Category:** Debugging / Scheduling
**Difficulty:** Medium
**Seen at:** Every K8s interview

## The Scenario
You apply a new Deployment and the Pods stay in `Pending` state indefinitely. No containers have started.

## What Pending Means
The Pod has been accepted by the API server but the scheduler has not yet assigned it to a node — or has failed to do so. The container runtime has not been contacted yet.

## Systematic Debug Steps

### Step 1: Always start here
```bash
kubectl describe pod <pod-name>
# Scroll to the Events section at the bottom
# The scheduler writes exactly why it couldn't schedule there
```

Common event messages and their causes:

| Event Message | Root Cause |
|---|---|
| `0/5 nodes are available: 5 Insufficient cpu` | No node has enough CPU for the Pod's request |
| `0/5 nodes are available: 5 node(s) had untolerated taint` | Nodes have taints the Pod doesn't tolerate |
| `0/5 nodes are available: 5 node(s) didn't match Pod's node affinity` | nodeAffinity/nodeSelector mismatch |
| `0/1 nodes are available: persistentvolumeclaim "x" not found` | PVC doesn't exist or is in wrong namespace |
| `0/5 nodes are available: 5 node(s) had volume node affinity conflict` | PVC in different AZ than available nodes |
| `no nodes available to schedule pods` | All nodes are cordoned or NotReady |

### Step 2: Check resource availability
```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes
# Are all nodes at capacity?

# Check if namespace has a ResourceQuota that's been exceeded
kubectl describe resourcequota -n <namespace>
```

### Step 3: Check for scheduling constraints
```bash
# Are the node labels matching what the Pod requires?
kubectl get nodes --show-labels
kubectl describe pod | grep -A 10 "Node-Selectors\|Tolerations\|Affinity"
```

### Step 4: Check PVC state
```bash
kubectl get pvc -n <namespace>
# Pending PVC → StorageClass problem or no matching PV
kubectl describe pvc <pvc-name>
```

### Step 5: Check node status
```bash
kubectl get nodes
# Look for: NotReady, SchedulingDisabled (cordoned)
kubectl describe node <node> | grep -A 5 "Conditions\|Taints"
```

## The 6 Root Causes of Pending (Memorize These)

1. **Insufficient resources** — CPU/memory requests exceed what any node has free. Fix: reduce requests, add nodes, or clean up other workloads.
2. **Taint/toleration mismatch** — node has a taint the Pod doesn't tolerate. Fix: add toleration, or retaint node.
3. **Node selector / affinity mismatch** — Pod requires labels no node has. Fix: fix labels on node or relax affinity.
4. **PVC Pending** — storage not available. Fix: fix StorageClass or create the PV manually.
5. **No nodes in cluster** — all nodes NotReady or cordoned. Fix: check node health, uncordon if appropriate.
6. **ResourceQuota exceeded** — namespace quota is full. Fix: increase quota or clean up old resources.

## Sources
- [[K8S overview]]
- [[k8s/topics/scheduling]]
