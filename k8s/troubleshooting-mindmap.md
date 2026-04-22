# K8s Troubleshooting

## Pod Problems

### CrashLoopBackOff
- `kubectl logs --previous`
- Exit code 1 → app error
- Exit code 137 → OOMKilled
- Exit code 143 → SIGTERM
- [[scenarios/crashloopbackoff]]

### OOMKilled
- Exit code 137
- `kubectl top pod --containers`
- Raise memory limit or fix leak
- [[scenarios/oomkilled]]

### Pending
- 0 nodes available
- Insufficient CPU/memory
- Untolerated taint
- nodeAffinity mismatch
- PVC missing or wrong AZ
- [[scenarios/pod-pending]]

### ImagePullBackOff
- Wrong image tag
- Missing imagePullSecret
- Registry unreachable

### Terminating Stuck
- Finalizer not released
- `kubectl patch` remove finalizer
- PVC still mounted

## Deployment & Rollout

### Rollout Stuck
- New pod CrashLoop / Pending
- Readiness probe failing
- PDB blocking old pod removal
- `kubectl rollout undo`
- [[scenarios/deployment-rollout-stuck]]

### ReplicaSet Not Scaling
- Selector mismatch
- Resource quota exceeded
- Node capacity exhausted

### Zero-Downtime Deploy Failed
- maxUnavailable=0 misconfigured
- Readiness gate not clearing
- [[scenarios/zero-downtime-deployment]]

## Autoscaling

### HPA Not Scaling
- Metrics Server missing
- No resource requests set
- Already at maxReplicas
- CPU throttled not utilized
- [[scenarios/hpa-not-scaling]]

### VPA Conflict with HPA
- Never use both on CPU/memory
- VPA for requests, HPA for replicas

### KEDA Not Triggering
- ScaledObject misconfigured
- Prometheus Adapter not deployed
- Metric not exposed

## Networking

### Service Not Reachable
- Empty endpoints → selector mismatch
- Pod not Ready → readiness probe
- NetworkPolicy blocking
- DNS resolution failing
- [[scenarios/service-unreachable]]

### Ingress Not Routing
- IngressClass mismatch
- Backend service wrong port
- TLS cert missing or expired
- `kubectl describe ingress` → Events

### DNS Failure
- CoreDNS pod not running
- ndots misconfigured
- `nslookup <svc>.<ns>.svc.cluster.local`

### Network Policy Blocked
- Default-deny active
- Missing egress rule
- `kubectl describe netpol`

## Node Problems

### Node NotReady
- kubelet stopped → SSH check
- Disk pressure → `df -h`
- Memory pressure → `free -m`
- Network partition
- `kubectl describe node` → Conditions

### Node Cordoned
- `kubectl cordon node` → no new pods
- `kubectl uncordon` to restore
- Drain before maintenance — `kubectl drain --ignore-daemonsets`

### DiskPressure
- Log rotation not running
- Image layers accumulating
- `kubectl describe node` → eviction threshold
- `crictl image prune`

### Node OOM
- System-level OOM killer
- All pods on node killed
- Review allocatable vs requested

## Storage

### PVC Stuck Pending
- No matching StorageClass
- Volume topology mismatch
- Manual PV not available
- `kubectl describe pvc` → Events

### PVC Stuck Terminating
- Pod still using volume
- Finalizer not released
- Force delete pod first

### Volume Mount Failing
- Secret/ConfigMap missing
- Wrong mountPath
- ReadWriteOnce used by 2 nodes
- [[concepts/persistent-volume]]

## Security & Config

### Secret Rotation Failing
- Env var requires pod restart
- Volume mount auto-updates ~1 min
- App must reload credential
- [[scenarios/secret-rotation]]

### RBAC Denied
- `kubectl auth can-i <verb> <resource>`
- Missing RoleBinding
- Wrong namespace scope
- [[concepts/rbac]]

### Pod Security Admission Blocked
- privileged not allowed
- runAsRoot forbidden
- securityContext required

## Observability Gaps

### Metrics Missing
- Metrics Server not installed
- ServiceMonitor selector wrong
- Pod missing prometheus annotations

### Logs Not Appearing
- Fluentd/Fluent Bit DaemonSet down
- Log rotation overwriting fast
- Structured log format broken

### Alerts Firing Silently
- AlertManager not configured
- Route not matching labels
- Receiver endpoint down
