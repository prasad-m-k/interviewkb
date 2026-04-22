# Containers & Orchestration

**Topic:** [[devops/overview]]
**Related:** [[devops/concepts/kubernetes-architecture]], [[devops/concepts/service-mesh]]

## Containers vs VMs

```
VMs:                           Containers:
┌─────────────────────────┐    ┌─────────────────────────┐
│  App A  │  App B         │    │  App A  │  App B         │
│  Libs   │  Libs          │    │  Libs   │  Libs          │
│  OS     │  OS            │    ├─────────────────────────┤
├─────────────────────────┤    │     Container Runtime    │
│      Hypervisor          │    │  (containerd / Docker)   │
├─────────────────────────┤    ├─────────────────────────┤
│      Host OS             │    │        Host OS           │
├─────────────────────────┤    ├─────────────────────────┤
│      Hardware            │    │        Hardware          │
└─────────────────────────┘    └─────────────────────────┘
Startup: minutes               Startup: milliseconds
Size: GBs                      Size: MBs
Isolation: strong (kernel)     Isolation: process-level (namespaces)
```

**Containers use:** Linux namespaces (PID, network, mount, UTS) + cgroups (CPU/memory limits) to isolate processes on the same kernel.

---

## Docker Essentials

### Key concepts
- **Image:** read-only layers; built from a Dockerfile; tagged with name:version
- **Container:** running instance of an image; adds a writable layer on top
- **Registry:** stores and distributes images (DockerHub, ECR, GCR, Artifactory)
- **Layer caching:** unchanged layers reuse the build cache — order Dockerfile steps from least to most frequently changed

### Dockerfile best practices

```dockerfile
# Bad: installs everything as root, large image
FROM ubuntu:latest
RUN apt-get install -y python3 pip && pip install -r requirements.txt
COPY . /app
CMD python3 /app/main.py

# Better: multi-stage build, non-root, pinned version, minimal base
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --from=builder /usr/local/bin /usr/local/bin
COPY src/ ./src/
USER 1000:1000           # non-root
CMD ["python3", "src/main.py"]
```

**Interview checklist for Dockerfile review:**
- [ ] Pinned base image version (not `latest`)
- [ ] Multi-stage build to minimize final image size
- [ ] Non-root user
- [ ] `.dockerignore` excludes `.git`, `node_modules`, secrets
- [ ] COPY before RUN to maximize layer cache reuse
- [ ] No secrets in ENV or RUN commands
- [ ] Health check defined

---

## Kubernetes Overview

See full architecture: [[devops/concepts/kubernetes-architecture]]

### The mental model

```
Kubernetes = desired state reconciler

You declare:  "I want 3 replicas of my-app:v2"
K8s does:     Compare desired ↔ actual → create/delete pods to match
```

### Core objects

| Object | What it is |
|---|---|
| **Pod** | Smallest deployable unit; 1+ containers sharing network + storage |
| **Deployment** | Manages ReplicaSets; handles rolling updates; most common way to run stateless apps |
| **Service** | Stable network endpoint for a set of pods (ClusterIP, NodePort, LoadBalancer) |
| **ConfigMap** | Non-secret config data injected as env vars or volume mounts |
| **Secret** | Base64-encoded (not encrypted!) sensitive config; use external secrets in prod |
| **Ingress** | HTTP/HTTPS routing from outside the cluster to Services |
| **PersistentVolume** | Cluster-level storage; PVC is a request for storage by a pod |
| **Namespace** | Virtual cluster isolation; RBAC and resource quotas applied per namespace |
| **HorizontalPodAutoscaler** | Scale Deployment replicas based on CPU/memory/custom metrics |

---

## Helm

Helm = package manager for Kubernetes.

- **Chart:** a package of K8s YAML templates with parameterized values
- **Release:** a deployed instance of a chart with specific values
- **Values:** override defaults per environment (dev/staging/prod `values.yaml`)

```bash
helm upgrade --install my-app ./chart \
  --namespace prod \
  --values values-prod.yaml \
  --atomic \              # roll back automatically if deploy fails
  --timeout 5m
```

---

## Common Interview Questions

**Q: What happens when a Pod is OOMKilled?**  
A: The container exceeds its `resources.limits.memory`. The Linux OOM killer terminates it. The pod restarts (if `restartPolicy` allows). Fix: increase limit, reduce memory usage, or set proper limits initially. See → [[devops/scenarios/kubernetes-debugging]]

**Q: Difference between liveness and readiness probes?**  
```
Readiness probe: "Is this pod ready to receive traffic?"
  → Failing: pod removed from Service endpoints (no traffic sent)
  → Use for: startup time, temporary unavailability

Liveness probe: "Is this pod alive and not deadlocked?"
  → Failing: pod restarted
  → Use for: deadlocks, hung processes that don't crash

Startup probe: "Has the container finished starting?"
  → Delays liveness probe; prevents premature restarts during slow startup
```

**Q: How does K8s networking work?**  
- Every pod gets its own IP
- Pods can reach each other directly (flat network via CNI plugin: Calico, Flannel, Cilium)
- Services get a stable ClusterIP via `kube-proxy` (iptables or IPVS)
- DNS: `<service>.<namespace>.svc.cluster.local`

## Sources
- [[devops/concepts/kubernetes-architecture]]
- [[devops/overview]]
