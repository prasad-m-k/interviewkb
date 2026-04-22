# Scenario: Pod in CrashLoopBackOff

**Category:** Debugging / Pod Lifecycle
**Difficulty:** Medium
**Seen at:** Every K8s interview

## The Scenario
Your model-serving Pod keeps restarting. `kubectl get pods` shows `CrashLoopBackOff` with restart count climbing. Production is impacted.

## What CrashLoopBackOff Means
The container starts, exits with a non-zero code (or is killed), and K8s keeps trying to restart it. The "BackOff" part means K8s introduces exponential delays between restarts (10s, 20s, 40s... up to 5 minutes) to avoid hammering a broken system.

## Systematic Debug Steps

### Step 1: Get the exit reason
```bash
kubectl describe pod <pod-name>
# Look at: Last State, Exit Code, Reason
# Exit Code 1 = application error
# Exit Code 137 = OOMKilled (128 + 9 SIGKILL)
# Exit Code 139 = segfault
# Exit Code 143 = SIGTERM (graceful shutdown — something killed it)
```

### Step 2: Get the logs from the crashed container
```bash
kubectl logs <pod-name> --previous   # logs from the last failed container
kubectl logs <pod-name> -c <container> --previous
```

### Step 3: Identify root cause by exit code

| Exit Code | Likely Cause | Debug Action |
|---|---|---|
| `1` | Application error | Check `--previous` logs for exception/stack trace |
| `137` | OOMKilled | Check memory limit; see [[k8s/scenarios/oomkilled]] |
| `2` | Misuse of shell / bad command | Check `command`/`args` in Pod spec |
| `126` | Command not executable | Check image entry point and permissions |
| `127` | Command not found | Wrong binary name or missing in image |
| `1` (no logs) | Process crashes before logging | Add `command: ["sleep", "3600"]` to debug interactively |

### Step 4: Common non-crash causes

**Bad configuration:**
```bash
# Check if env vars are missing/wrong
kubectl exec <pod> -- env | grep MY_VAR

# Check if a mounted ConfigMap/Secret exists
kubectl get configmap <name> -n <namespace>
kubectl get secret <name> -n <namespace>
```

**Missing volume / PVC not bound:**
```bash
kubectl describe pod | grep -A 5 "Volumes"
kubectl get pvc -n <namespace>
```

**Wrong image / entrypoint:**
```bash
kubectl describe pod | grep "Image:"
# Try running the image locally: docker run <image>
```

**Liveness probe too aggressive:**
```bash
kubectl describe pod | grep -A 10 "Liveness"
# If initialDelaySeconds is too short, liveness kills before app is ready
# Fix: increase initialDelaySeconds or use startupProbe
```

## Quick Triage Cheatsheet

```bash
# Full picture in one command
kubectl describe pod <pod> | grep -E "State|Exit Code|Reason|Last State|OOM|Liveness"

# Logs from the crash
kubectl logs <pod> --previous --tail=50

# If no logs (crash before first log line)
kubectl get events --field-selector involvedObject.name=<pod>
```

## The Fix Approaches

1. **App error**: fix the application code or configuration
2. **OOMKilled**: raise memory limit or fix memory leak
3. **Missing config**: create/fix the ConfigMap or Secret
4. **Bad probe timing**: add `startupProbe` or increase `initialDelaySeconds`
5. **Entrypoint wrong**: fix `command`/`args` or Dockerfile CMD/ENTRYPOINT

## Sources
- [[K8S overview]]
- [[k8s/topics/observability]]
