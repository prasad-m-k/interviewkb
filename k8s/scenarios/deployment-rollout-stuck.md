# Scenario: Deployment Rollout Stuck

**Category:** Debugging / Deployments
**Difficulty:** Medium
**Seen at:** Amazon, Google, Meta

## The Scenario
You run `kubectl apply` on a new Deployment image version. `kubectl rollout status deployment/model-server` is stuck at "Waiting for deployment rollout to finish: 1 out of 3 new replicas have been updated..." for 10 minutes.

## Systematic Debug Steps

### Step 1: Check the rollout status
```bash
kubectl rollout status deployment/model-server
# Gives: "Waiting for deployment..." → rollout not complete

kubectl get deployment model-server
# READY column: 2/3 → one replica not ready
```

### Step 2: Find the new (stuck) Pod
```bash
kubectl get pods -l app=model-server --sort-by=.metadata.creationTimestamp
# The newest Pod is the one that's stuck
# Look at its STATUS and READY columns
```

### Step 3: Describe the stuck Pod
```bash
kubectl describe pod <newest-pod>
# Check:
# - State (Waiting, Running, Terminated)
# - Reason (CrashLoopBackOff, ImagePullBackOff, OOMKilled)
# - Events section at the bottom
```

### Step 4: Diagnose by Pod state

**CrashLoopBackOff**: new version has a bug
```bash
kubectl logs <new-pod> --previous
# Fix the bug, push a new image, re-apply
```

**ImagePullBackOff**: wrong image tag or missing pull secret
```bash
kubectl describe pod <new-pod> | grep "Image\|Pull"
# "Failed to pull image" → check image name/tag
# "unauthorized" → missing imagePullSecret
kubectl get secret regcred -n <namespace>
```

**Pending**: not enough resources or scheduling constraint
```bash
kubectl describe pod <new-pod> | grep "Events" -A 10
# "Insufficient memory" → new image has higher resource requests
# Fix: add more nodes or reduce requests
```

**Readiness probe failing**: new version can't serve health checks
```bash
kubectl logs <new-pod>
kubectl exec -it <new-pod> -- curl localhost:8080/ready
# App not listening on expected port, or /ready returns non-200
# Fix: fix the app, or fix the readiness probe endpoint
```

### Step 5: Check if PDB is blocking
```bash
kubectl describe pdb -n <namespace>
# "disruption allowed: 0" → PDB preventing old Pod removal
# Happens when current healthy Pods < minAvailable
# Fix: check if other Pods are already down; investigate why
```

## Undo a Stuck Rollout

```bash
# Option 1: Roll back to previous version
kubectl rollout undo deployment/model-server

# Option 2: Roll back to specific revision
kubectl rollout history deployment/model-server
kubectl rollout undo deployment/model-server --to-revision=<n>

# Option 3: Pause the rollout while you investigate
kubectl rollout pause deployment/model-server
# ... investigate ...
kubectl rollout resume deployment/model-server
```

## Deployment Rollout Timeout

Set a deadline after which K8s marks the deployment as failed:
```yaml
spec:
  progressDeadlineSeconds: 600   # fail after 10 min of no progress
```

```bash
kubectl get deployment model-server
# If AVAILABLE < DESIRED and progressDeadlineSeconds exceeded:
# Deployment shows "DeadlineExceeded" condition
```

## Sources
- [[K8S overview]]
- [[k8s/concepts/deployment]]
