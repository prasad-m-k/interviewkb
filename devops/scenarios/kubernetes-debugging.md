# Scenario: Kubernetes Pod Debugging

**Type:** Operational / Debugging
**Difficulty:** Medium
**Frequency:** Very high in senior DevOps / Platform Engineering interviews

## Common Pod States and Their Fixes

### Cheat Sheet

```
kubectl get pods -n <ns>          → see all pod states
kubectl describe pod <pod>        → events, resource issues, probe failures
kubectl logs <pod> --previous     → logs from the crashed container
kubectl exec -it <pod> -- sh      → interactive debug shell
kubectl top pod <pod>             → live CPU/memory usage
kubectl get events -n <ns> --sort-by='.lastTimestamp'  → cluster events
```

---

## State 1: CrashLoopBackOff

**What it means:** the container starts, crashes, Kubernetes restarts it, it crashes again. The backoff period between restarts increases exponentially (10s → 20s → 40s → 5min max).

**Debug steps:**
```bash
# 1. Look at the crash logs (previous attempt, not current)
kubectl logs <pod> --previous

# 2. Look at events
kubectl describe pod <pod>
# Look for: Exit Code, Last State, Reason

# Common exit codes:
# Exit 0: normal exit (process shouldn't have been exiting at all)
# Exit 1: application error (check app logs)
# Exit 137: OOMKilled (out of memory) — see OOMKill below
# Exit 139: segfault (language runtime issue, native library)
```

**Common causes:**
| Cause | Signal | Fix |
|---|---|---|
| App crashes on startup | Exit 1, stack trace in logs | Fix the code/config bug |
| Missing env var / secret | `KeyError`, `undefined` | Add the missing config |
| Can't connect to DB | `Connection refused` at startup | Fix DB connectivity or startup ordering |
| Liveness probe too aggressive | Probe failures in describe | Tune `initialDelaySeconds`, `failureThreshold` |
| OOMKilled | Exit 137 | Increase memory limit or fix memory leak |

---

## State 2: OOMKilled

**What it means:** container used more memory than its `resources.limits.memory`. Linux OOM killer terminates it.

```bash
kubectl describe pod <pod>
# Look for:
# Last State: Terminated
# Reason: OOMKilled
# Exit Code: 137

kubectl top pod <pod>    # live memory usage
```

**Fixes:**
```yaml
# 1. Increase the limit (short-term fix)
resources:
  requests:
    memory: "256Mi"
  limits:
    memory: "512Mi"    # increase this

# 2. Fix the memory leak (long-term)
# - Profile the app: add heap dump on high-mem signal
# - Add memory metric + alert before it hits the limit
# - Check for unbounded caches, connection pool leaks, large in-memory datasets
```

**Request vs Limit:**
- `requests`: guaranteed; used for scheduling decisions
- `limits`: enforced ceiling; OOMKill or CPU throttle at this value
- **Best practice:** set requests = typical usage; limit = 2-3× requests (buffer for spikes)

---

## State 3: Pending (pod not starting)

**What it means:** the pod is created but not scheduled to any node yet.

```bash
kubectl describe pod <pod>
# Look for Events section:
# "0/3 nodes are available: 3 Insufficient cpu"  → resource constraint
# "0/3 nodes are available: 3 node(s) had taint ... pod didn't tolerate"  → taint issue
# "0/3 nodes are available: 3 node(s) didn't match Pod's node affinity"  → affinity
```

**Fixes by cause:**

| Event message | Cause | Fix |
|---|---|---|
| `Insufficient cpu/memory` | No node has enough resources | Scale node group; reduce pod requests |
| `didn't tolerate taint` | Node tainted; pod has no toleration | Add toleration to pod spec |
| `didn't match node affinity` | nodeSelector / affinity too restrictive | Relax affinity or label nodes correctly |
| `pod has unbound PersistentVolumeClaims` | PVC not bound (no matching PV, wrong StorageClass) | Check StorageClass; check PV availability |
| `Unschedulable` (no reason) | Check if nodes exist and are Ready | `kubectl get nodes` — any NotReady? |

---

## State 4: ImagePullBackOff / ErrImagePull

**What it means:** Kubernetes can't pull the container image.

```bash
kubectl describe pod <pod>
# Events: "Failed to pull image ... 401 Unauthorized"
#         "Failed to pull image ... not found"
#         "rpc error: code = Unknown"
```

**Fixes:**
```bash
# Wrong image name/tag
kubectl set image deployment/my-app my-app=my-registry/my-app:correct-tag

# Private registry: check imagePullSecret
kubectl get secret registry-credentials -n prod
# If missing:
kubectl create secret docker-registry registry-credentials \
  --docker-server=my.registry.io \
  --docker-username=user \
  --docker-password=pass
# Then reference in deployment spec:
#   imagePullSecrets:
#     - name: registry-credentials
```

---

## State 5: Running but not Ready (Readiness probe failing)

**What it means:** pod is running but readiness probe fails → removed from Service endpoints → no traffic routed.

```bash
kubectl describe pod <pod>
# Events: "Readiness probe failed: HTTP probe failed with statuscode: 503"
#         "Readiness probe failed: dial tcp ... connection refused"

kubectl logs <pod>   # is the app actually listening on the probe port?
```

**Common causes:**
- App not finished initializing (fix: `initialDelaySeconds` or `startupProbe`)
- App listening on wrong port
- App health endpoint returns non-2xx during degraded state
- App crashed but container didn't exit (deadlock) → liveness probe would catch this

---

## State 6: Terminating (stuck)

**What it means:** pod is stuck in `Terminating` state and won't die.

```bash
# Force delete (last resort — check for data safety first)
kubectl delete pod <pod> --force --grace-period=0
```

**Cause:** usually a finalizer that's blocked. Check:
```bash
kubectl get pod <pod> -o yaml | grep finalizers
# Remove the finalizer if the process is stuck:
kubectl patch pod <pod> -p '{"metadata":{"finalizers":null}}'
```

---

## Systematic Debug Checklist

```
[ ] kubectl get pods → what state?
[ ] kubectl describe pod → events section (resources, probes, scheduling)
[ ] kubectl logs --previous → crash logs
[ ] kubectl top pod → resource usage
[ ] kubectl get events → cluster-wide context
[ ] Check node health: kubectl get nodes
[ ] Check ConfigMap / Secret mounted correctly
[ ] Check NetworkPolicy blocking traffic
[ ] Check RBAC if pod needs to call API server
```

## Sources
- [[devops/concepts/kubernetes-architecture]]
- [[devops/topics/containers-orchestration]]
