# Scenario: Pod OOMKilled

**Category:** Debugging / Resource Management
**Difficulty:** Medium
**Seen at:** Google, Amazon, Meta

## The Scenario
A model-serving Pod (or training job) is repeatedly killed with reason `OOMKilled`. Exit code is `137`. The service is unstable.

## What OOMKilled Means
The Linux kernel OOM killer terminated the container's main process because the container exceeded its memory `limit`. This is a hard kill — no graceful shutdown, no chance for the process to clean up.

## Immediate Diagnosis

```bash
# Confirm it's OOM
kubectl describe pod <pod>
# Look for:
# Last State: Terminated  Reason: OOMKilled  Exit Code: 137

# Check current memory usage vs. limit
kubectl top pod <pod> --containers
kubectl describe pod | grep -A 3 "Limits\|Requests"

# Check node memory pressure
kubectl describe node <node> | grep -A 5 "MemoryPressure\|Conditions"
```

## Root Cause Analysis

### Is the limit too low for normal operation?
```bash
kubectl top pod <pod> -n <namespace>
# If working_set_memory approaches the limit → raise the limit
```

**Fix**: increase `resources.limits.memory` to give legitimate headroom.

### Is there a memory leak?
```bash
# Watch memory growth over time
kubectl top pod <pod> --watch
# Continuously growing → leak

# Check application metrics if exposed on /metrics
kubectl port-forward pod/<pod> 8080:8080
curl localhost:8080/metrics | grep process_resident_memory
```

**Fix**: profile the application; fix the leak; add `-Xmx` flags (JVM), or implement a periodic worker restart (`terminationMessagePath` + CronJob).

### Is the process loading too much data at startup?
**Common in ML**: loading a large model into memory spikes usage before serving. The container gets OOMKilled before it's even ready.

```bash
# Check if it's killed during startup vs. under load
kubectl logs <pod> --previous | tail -30
# "Loading model..." with no further output → killed during load
```

**Fix**: raise the limit to accommodate the model size; add a `startupProbe` with a long `failureThreshold` to give it time.

### Is the limit set but requests are much lower?
```yaml
# Bad: request=256Mi, limit=4Gi
# K8s schedules to a node with 256Mi free → OOMKill under load
resources:
  requests:
    memory: "2Gi"   # should be close to expected steady-state
  limits:
    memory: "4Gi"   # room for spikes
```

## Permanent Fixes

1. **Raise the limit** to match real workload requirements
2. **Profile memory usage** with `memory-profiler` (Python), `pprof` (Go), JVM heap dumps
3. **Set requests close to steady-state usage** — prevents overcommit + OOM
4. **Use VPA** to right-size requests/limits automatically based on historical usage
5. **For ML model loading**: pre-warm the model in an init container and share via `emptyDir` volume

## Sources
- [[K8S overview]]
- [[k8s/topics/scheduling]]
