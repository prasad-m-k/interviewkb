# Kubernetes Observability

**Topic:** [[k8s/topics/observability]]
**Related:** [[k8s/topics/workloads]], [[k8s/concepts/pod]]

## Pod Probes

Three probe types, each configurable with `httpGet`, `tcpSocket`, or `exec`.

| Probe | Question asked | Action on failure | When to use |
|---|---|---|---|
| **liveness** | Is the app alive? | Kill and restart container | App can deadlock or enter a bad state that a restart fixes |
| **readiness** | Is the app ready for traffic? | Remove from Service endpoints | App needs warm-up time, or temporarily unable to serve |
| **startup** | Has the app finished starting? | Kill and restart | Slow-starting apps that would fail liveness during init |

**Critical rule:** `startupProbe` disables both `livenessProbe` and `readinessProbe` until it succeeds. This prevents premature restarts of slow-starting containers (e.g., JVM apps, ML model loading).

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 1      # fail fast for readiness

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30     # allow up to 30 * periodSeconds for startup
  periodSeconds: 10
```

**Common mistake**: using `livenessProbe` where `readinessProbe` is correct. A failing liveness probe restarts the Pod; a failing readiness probe just stops traffic. If the app is temporarily overloaded, a liveness restart makes it worse (thundering herd on startup).

---

## Logging

**kubectl commands:**
```bash
kubectl logs <pod>                     # current container logs
kubectl logs <pod> --previous          # logs from the previous (crashed) container
kubectl logs <pod> -c <container>      # specific container in multi-container pod
kubectl logs -l app=model-server       # all pods with label
kubectl logs <pod> --follow            # stream live
```

**Cluster-wide logging:** Pods write to stdout/stderr → container runtime writes to node log files → DaemonSet log collector (Fluentd/Filebeat) tails node files → ships to centralized store (Elasticsearch, Loki, CloudWatch).

---

## Metrics

**Metrics Server**: in-cluster component that collects CPU/memory from kubelet. Used by HPA and `kubectl top`.

```bash
kubectl top nodes
kubectl top pods -n <namespace>
kubectl top pods --sort-by=memory
```

**Prometheus stack**: production-grade. Prometheus scrapes `/metrics` endpoints; Grafana visualizes; AlertManager fires alerts.

**Key metrics for ML workloads:**
- `container_memory_working_set_bytes` vs. memory limit → OOM risk
- `container_cpu_cfs_throttled_seconds_total` → CPU throttling (often worse than OOM for latency)
- GPU utilization via DCGM exporter
- Custom model metrics (inference latency, queue depth) via Prometheus PushGateway or scrape endpoint

---

## Events

Kubernetes Events are the most underused debug tool. They record what happened in the cluster:

```bash
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl describe pod <pod>    # shows events in the bottom section
kubectl get events --field-selector reason=BackOff
```

Common event reasons: `Scheduled`, `Pulling`, `Pulled`, `Created`, `Started`, `BackOff`, `OOMKilling`, `FailedScheduling`, `FailedMount`.

---

## The Debug Toolkit

```bash
# Exec into a running pod
kubectl exec -it <pod> -- /bin/sh

# Run a debug container alongside a crashlooping pod (K8s 1.23+)
kubectl debug -it <pod> --image=busybox --target=<container>

# Check all resources in a namespace
kubectl get all -n <namespace>

# Describe any resource (events + full spec + status)
kubectl describe <kind> <name> -n <namespace>

# Watch events in real time
kubectl get events -n <namespace> -w

# Port-forward to test a service directly
kubectl port-forward svc/<service> 8080:80

# Check resource usage
kubectl top pods -n <namespace> --containers
```

## Sources
- [[K8S overview]]
