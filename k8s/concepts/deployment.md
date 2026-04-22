# Deployment

**Topic:** [[k8s/topics/workloads]]
**Related:** [[k8s/concepts/pod]], [[k8s/concepts/hpa]]

## What it is
A Deployment manages a ReplicaSet, which manages a set of identical Pods. Provides declarative rolling updates, rollbacks, and horizontal scaling for stateless workloads.

## How it works
The Deployment controller watches the Deployment spec and creates/updates a ReplicaSet to match. During a rolling update, it creates a *new* ReplicaSet and gradually scales it up while scaling down the old one.

```
Deployment v1 (desired: 5 replicas)
└── ReplicaSet-abc (old spec: 0 replicas ← being scaled down)
└── ReplicaSet-xyz (new spec: 5 replicas ← being scaled up)
```

## Key Fields

```yaml
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # extra pods during rollout (can be %)
      maxUnavailable: 0    # pods that can be down during rollout
  template:
    metadata:
      labels:
        app: model-server
    spec:
      containers:
      - name: server
        image: model-server:v2
      terminationGracePeriodSeconds: 60
```

## Rolling Update Mechanics

With `maxSurge: 1, maxUnavailable: 0` and 3 replicas:
1. Create 1 new Pod (total: 4)
2. Wait for new Pod to be Ready
3. Terminate 1 old Pod (total: 3)
4. Repeat until all 3 old Pods replaced

`maxUnavailable: 0` ensures 3 Pods are always serving traffic. Critical for zero-downtime deployments.

## Rollback

```bash
kubectl rollout status deployment/model-server
kubectl rollout history deployment/model-server
kubectl rollout undo deployment/model-server                    # roll back one step
kubectl rollout undo deployment/model-server --to-revision=2   # specific revision
```

## Zero-Downtime Deployment Requirements

A rolling update alone is not enough. Also required:
1. **Readiness probe**: new Pods must pass readiness before old ones are removed
2. **`terminationGracePeriodSeconds`**: give old Pods time to finish in-flight requests
3. **`preStop` hook**: add a sleep to allow the load balancer to drain connections before the process is signaled

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

4. **PodDisruptionBudget**: prevents over-aggressive rolling under pressure

## Deployment Strategies

| Strategy | How | Use when |
|---|---|---|
| `RollingUpdate` | Incremental replacement | Default; zero-downtime |
| `Recreate` | Kill all, then start all | OK for dev/batch; causes downtime |
| Blue/Green (external) | Route traffic between two Deployments | Instant rollback; requires extra infrastructure |
| Canary (external) | Route small % to new version | Progressive rollout; needs Ingress/service mesh |

## Common Interview Angles
- "Your rolling update is stuck" → one Pod not passing readiness; check logs + describe
- "How do you do a canary deploy in K8s?" → two Deployments, Ingress or service mesh for traffic split
- "New pods are created but old ones aren't terminating" → PDB preventing disruption; `kubectl describe pdb`

## Sources
- [[K8S overview]]
