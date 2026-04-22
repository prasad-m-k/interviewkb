# StatefulSet

**Topic:** [[k8s/topics/workloads]]
**Related:** [[k8s/concepts/persistent-volume]], [[k8s/concepts/service]]

## What it is
A StatefulSet manages a set of Pods with **stable identity** — each Pod has a persistent name, stable network hostname, and optionally its own PersistentVolumeClaim that survives Pod restarts and rescheduling.

## How it differs from a Deployment

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random suffix (`app-7d4b9-xyz`) | Ordered index (`app-0`, `app-1`) |
| Network identity | No stable hostname | Each Pod has a DNS name via Headless Service |
| Storage | Shared PVC or none | Per-Pod PVC via `volumeClaimTemplates` |
| Startup order | All Pods start in parallel | Sequential: `app-0` ready → `app-1` starts |
| Scale-down | Simultaneous | Sequential: highest index first |
| Use case | Stateless services | Databases, distributed training, Kafka, ZooKeeper |

## Key YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: training-worker
spec:
  serviceName: training-worker-headless   # must reference a Headless Service
  replicas: 3
  selector:
    matchLabels:
      app: training-worker
  template:
    metadata:
      labels:
        app: training-worker
    spec:
      containers:
      - name: worker
        image: training:v1
        volumeMounts:
        - name: checkpoint
          mountPath: /checkpoints
  volumeClaimTemplates:
  - metadata:
      name: checkpoint
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
---
# Required: Headless Service for stable DNS
apiVersion: v1
kind: Service
metadata:
  name: training-worker-headless
spec:
  clusterIP: None
  selector:
    app: training-worker
  ports:
  - port: 7777
```

This gives each Pod a DNS name: `training-worker-0.training-worker-headless.<ns>.svc.cluster.local`

## Pod Management Policy

- `OrderedReady` (default): sequential start/stop
- `Parallel`: all Pods start/stop simultaneously — use when ordering isn't required but stable identity is

## Common Interview Angles
- "Scaling down a StatefulSet doesn't delete PVCs" → intentional; data is preserved; must delete PVCs manually
- "StatefulSet Pod is stuck because previous Pod isn't Ready" → `OrderedReady` policy; fix the blocking Pod first
- "How does distributed ML training use StatefulSets?" → each worker has a stable hostname for peer discovery (e.g., `worker-0.svc` as the master); each has its own checkpoint storage

## Sources
- [[K8S overview]]
