# Pattern: Zero-Downtime Deployment

**Topic:** [[devops/topics/ci-cd]]
**Related:** [[devops/concepts/deployment-strategies]], [[devops/concepts/kubernetes-architecture]]

## Requirements for Zero Downtime

All four must be true simultaneously:
1. **No gap in serving capacity** — new pods ready before old pods are removed
2. **Health checks work** — readiness probe passes before traffic is routed
3. **Graceful shutdown** — old pods finish in-flight requests before terminating
4. **Backward-compatible changes** — DB, API, config all handle both versions during rollout

---

## Kubernetes Rolling Update Config

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2         # allow 2 extra pods (total 8 during rollout)
      maxUnavailable: 0   # never go below 6 healthy pods
  template:
    spec:
      containers:
        - name: app
          image: my-app:v2
          readinessProbe:         # MUST be defined for zero-downtime
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:              # delay SIGTERM to finish in-flight requests
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 30
```

### Why each field matters

| Field | Without it | With it |
|---|---|---|
| `maxUnavailable: 0` | Old pods killed before new ones ready → gap in capacity | Always maintain at least `replicas` healthy pods |
| `readinessProbe` | Traffic sent to pods that aren't ready yet → errors | Traffic only routes to ready pods |
| `preStop: sleep` | Pod removed from LB but iptables rules still route traffic for a few seconds | Delays SIGTERM so in-flight requests can complete |
| `terminationGracePeriodSeconds` | SIGKILL fires too early → in-flight requests killed | Gives app time to drain |

---

## The Connection Draining Problem

When K8s removes a pod from Service endpoints, iptables rules take a few seconds to propagate across all nodes. During this gap, new requests might still arrive at the pod being terminated.

```
Timeline of pod deletion:
t=0s: Pod deleted → K8s sends SIGTERM + removes from Endpoints
t=0s-2s: iptables rules still route some traffic to this pod
t=0s: preStop hook fires → sleep 10 (keeps pod alive for draining)
t=10s: preStop completes → app receives SIGTERM
t=10s-30s: app's SIGTERM handler drains in-flight requests gracefully
t=30s: terminationGracePeriodSeconds → SIGKILL if still running
```

**Application code must handle SIGTERM gracefully:**
```python
import signal, sys

def graceful_shutdown(signum, frame):
    print("SIGTERM received, finishing in-flight requests...")
    server.shutdown()  # stop accepting new requests
    # existing requests continue until done
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

---

## Database Migration: Expand-Contract

The biggest threat to zero-downtime is a DB schema change that's incompatible with the old code.

```
Phase 1 — Expand (backward-compatible):
  ALTER TABLE orders ADD COLUMN new_status VARCHAR(50) NULL;
  Deploy v2 code: reads new_status, falls back to old_status if NULL
  ← old code still works (ignores new column)

Phase 2 — Migrate:
  UPDATE orders SET new_status = map(old_status) WHERE new_status IS NULL;
  Deploy v3 code: reads only new_status
  ← old code still works

Phase 3 — Contract (cleanup):
  ALTER TABLE orders DROP COLUMN old_status;
  ← safe: no code reads old_status anymore
```

**Rules:**
- Never add `NOT NULL` column without a default in the same deploy as the code change
- Never drop a column in the same deploy as removing its code usage
- Never rename a column in one step — add new, migrate data, remove old

---

## Rollback Triggers

Define automated rollback conditions before you deploy:

```yaml
# Argo Rollouts analysis template
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
spec:
  metrics:
    - name: error-rate
      interval: 1m
      successCondition: result[0] < 0.05    # < 5% error rate
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus:9090
          query: |
            sum(rate(http_requests_total{status=~"5..",app="my-app"}[1m]))
            /
            sum(rate(http_requests_total{app="my-app"}[1m]))
```

Manual rollback:
```bash
# Kubernetes
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3

# Helm
helm rollback my-app 2   # roll back to revision 2

# Check rollout history
kubectl rollout history deployment/my-app
```

## Sources
- [[devops/concepts/deployment-strategies]]
- [[devops/topics/ci-cd]]
