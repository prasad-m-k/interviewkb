# Scenario: Zero-Downtime Deployment

**Category:** Operations / Reliability
**Difficulty:** Hard
**Seen at:** Google, Meta, Amazon, Apple

## The Scenario
You need to deploy a new version of a model-serving API without any dropped requests. The service handles 1000 RPS with a p99 of 200ms.

## Why Rolling Update Alone Is Not Enough

A `RollingUpdate` strategy replaces old Pods with new ones. But there are several windows where requests can be dropped:

1. **New Pod not ready yet**: load balancer still sends traffic to starting Pods
2. **Old Pod gets SIGTERM while handling a request**: in-flight requests are dropped
3. **Service endpoints not yet updated**: kube-proxy still routes to terminating Pods for a brief window

## The Complete Zero-Downtime Recipe

### 1. Readiness Probe (gate traffic)
New Pods only receive traffic after passing the readiness probe. Without this, the load balancer sends traffic to a Pod that hasn't loaded its model yet.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 1        # remove from rotation on first failure
  successThreshold: 1
```

### 2. Rolling Update with maxUnavailable: 0
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1              # one extra Pod during rollout
    maxUnavailable: 0        # never reduce below desired replica count
```

This ensures the old Pods are only removed after new Pods are Ready.

### 3. preStop Hook + terminationGracePeriodSeconds (drain in-flight requests)
When K8s sends SIGTERM, the process might kill active connections immediately. A `preStop` sleep gives the load balancer time to stop routing new requests before the process is signaled.

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
terminationGracePeriodSeconds: 30   # must be > preStop sleep + drain time
```

**Sequence with preStop:**
1. Pod is marked for deletion
2. Pod removed from Service endpoints (load balancer stops sending new requests)
3. `preStop` hook runs (sleep 10s — lets in-flight requests complete)
4. SIGTERM sent to main process
5. Process gracefully shuts down (handle SIGTERM in code!)
6. After `terminationGracePeriodSeconds`, SIGKILL is sent as a fallback

### 4. PodDisruptionBudget (protect during voluntary disruptions)
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: model-server-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: model-server
```

Prevents node drains and Cluster Autoscaler from removing too many Pods at once.

### 5. Handle SIGTERM in your application
```python
import signal, sys

def graceful_shutdown(signum, frame):
    # Stop accepting new requests
    server.stop(grace=5)    # 5s drain
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
```

## The Endpoint Propagation Race

Even after a Pod starts terminating, kube-proxy may take a few seconds to remove it from iptables rules. During this window, new connections are still routed to the terminating Pod.

`preStop` sleep of 5–10s bridges this window. The duration should exceed the time from "Pod marked Terminating" to "kube-proxy removes the endpoint."

## Deployment with Database Schema Migration

When a new version requires a DB schema change:

**Safe pattern:**
1. Deploy migration as a K8s Job *before* deploying the new application version
2. Write migration to be backwards compatible (old app + new schema both work)
3. Deploy new app version
4. Optional: clean up deprecated schema in a later migration

**Never**: run migrations inside an init container of the app Deployment — multiple Pods racing to run the same migration causes conflicts.

## Sources
- [[K8S overview]]
- [[k8s/topics/workloads]]
