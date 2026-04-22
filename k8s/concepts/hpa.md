# Horizontal Pod Autoscaler (HPA)

**Topic:** [[k8s/topics/scheduling]]
**Related:** [[k8s/concepts/deployment]], [[k8s/topics/observability]]

## What it is
The HPA automatically adjusts the number of Pod replicas in a Deployment (or StatefulSet, ReplicaSet) based on observed metrics, targeting a defined utilization.

## How it works

```
Metrics Server / Custom Metrics API
        │  (every 15s by default)
        ▼
HPA Controller
        │  computes: desiredReplicas = ceil(currentReplicas × currentMetric / targetMetric)
        ▼
Deployment replica count adjusted
```

The control loop runs every 15 seconds (configurable via `--horizontal-pod-autoscaler-sync-period`). It applies a stabilization window (default 5 min for scale-down) to prevent thrashing.

## Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: model-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: model-server
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70   # target 70% of CPU request
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 1Gi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60              # remove max 1 pod per minute
    scaleUp:
      stabilizationWindowSeconds: 0    # scale up immediately
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60              # can double replicas per minute
```

## Metric Sources

| Source | Examples | Requires |
|---|---|---|
| `Resource` | CPU, Memory | Metrics Server |
| `Pods` | custom per-pod metric | Prometheus Adapter |
| `Object` | Ingress RPS, queue depth | Prometheus Adapter |
| `External` | SQS queue length, Pub/Sub backlog | KEDA or custom adapter |

**KEDA (Kubernetes Event-Driven Autoscaling)**: extends HPA to scale on 50+ external event sources (SQS, Kafka, Redis, cron). Scales to zero. Critical for ML batch inference pipelines.

## Why HPA Doesn't Scale: Checklist

1. **Metrics Server not installed** → `kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods` returns 404
2. **No resource requests set on containers** → HPA can't compute utilization (needs `requests` as the denominator)
3. **Already at `maxReplicas`** → scaling up impossible; increase the max
4. **Stabilization window** → scale-down is being delayed; expected behavior
5. **Pods not becoming Ready** → new Pods count toward replicas but not ready capacity; HPA sees high utilization but scale-up isn't helping (fix the readiness issue first)
6. **Custom metric not exposed** → `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1`

## HPA vs VPA

| | HPA | VPA |
|---|---|---|
| Scales | Number of replicas (horizontal) | CPU/memory requests per Pod (vertical) |
| When to use | Stateless, load-based scaling | Single-instance workloads; right-sizing |
| Combines with HPA? | N/A | Only with VPA in `Off` or `Initial` mode for requests; not in `Auto` mode simultaneously |
| Limitation | Doesn't help if one replica is too small | Requires Pod restart to apply |

## Sources
- [[K8S overview]]
