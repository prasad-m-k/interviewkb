# Scenario: HPA Not Scaling

**Category:** Debugging / Autoscaling
**Difficulty:** Medium
**Seen at:** Google, Amazon

## The Scenario
Your model-serving Deployment has an HPA configured targeting 70% CPU utilization. Traffic is high, but the HPA isn't adding replicas. Users are experiencing latency spikes.

## Immediate Diagnosis

```bash
# Step 1: Describe the HPA
kubectl describe hpa <hpa-name> -n <namespace>
# Look at:
# - Current Metrics vs Target
# - Conditions section (shows why it's NOT scaling)
# - Events section

# Step 2: Check HPA status
kubectl get hpa <hpa-name>
# TARGETS column: shows current/target metrics
# If TARGETS shows <unknown>/70% → metrics not being collected
```

## The 7 Common Causes

### 1. Metrics Server not installed or unavailable
```bash
kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | head -5
# If 404 or connection refused → Metrics Server not running

kubectl get pods -n kube-system | grep metrics-server
kubectl logs -n kube-system <metrics-server-pod>
```
**Fix**: install/restart metrics-server.

### 2. No resource `requests` set on containers
```bash
kubectl describe pod <pod> | grep -A 5 "Requests"
# If CPU: 0 → HPA can't compute utilization (needs denominator)
```

HPA calculates: `utilization% = current_CPU / requests_CPU`. If `requests` is 0, this is undefined.

**Fix**: always set `resources.requests.cpu` on every container.

### 3. Already at `maxReplicas`
```bash
kubectl get hpa <name>
# REPLICAS column = maxReplicas → can't scale higher
```
**Fix**: increase `maxReplicas` in the HPA spec.

### 4. Scale-down stabilization window
```bash
kubectl describe hpa <name>
# Condition: AbleToScale False  ScaleDownStabilized
# Normal — HPA waits 5 min (default) before scaling down to prevent thrashing
```
**This is expected behavior**. The scale-up direction has no stabilization window by default.

### 5. New Pods not becoming Ready
If HPA scales up but new Pods fail their readiness probe, the replicas increase but effective capacity doesn't. HPA will keep trying to scale.

```bash
kubectl get pods | grep <deployment>
# Many pods in 0/1 Not Ready → fix readiness probe / app issue
```

### 6. CPU throttling, not just utilization
CPU limit throttling is NOT reflected in CPU utilization metrics. A Pod can show 70% CPU utilization but be severely throttled (100% throttle ratio).

```bash
# Check throttling
kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat | grep throttled
# Or via Prometheus: container_cpu_cfs_throttled_seconds_total
```
**Fix**: increase CPU `limits` or remove them (and rely on node-level fairness).

### 7. KEDA/custom metrics not configured
If scaling on custom metrics (queue depth, RPS), the Prometheus Adapter or KEDA ScaledObject must be deployed and the metric must be exposed.

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1
kubectl get scaledobject -n <namespace>
```

## HPA Describe Output — Reading the Conditions

```
Conditions:
  Type            Status  Reason              Message
  ----            ------  ------              -------
  AbleToScale     True    ReadyForNewScale    recommended size matches current size
  ScalingActive   True    ValidMetricFound    the HPA was able to successfully calculate
  ScalingLimited  False   DesiredWithinRange  the desired count is within the acceptable range
```

If `ScalingActive` is `False` with reason `FailedGetResourceMetric` → metrics collection issue.

## Sources
- [[K8S overview]]
- [[k8s/concepts/hpa]]
