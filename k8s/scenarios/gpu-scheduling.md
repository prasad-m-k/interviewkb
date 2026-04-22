# Scenario: Running ML Workloads on GPU Nodes

**Category:** MLOps / Scheduling
**Difficulty:** Hard
**Seen at:** Google, Amazon, Meta (ML-focused roles)

## The Scenario
You have a K8s cluster with a mix of CPU nodes and GPU nodes. You need to:
1. Ensure ML training jobs use GPU nodes
2. Ensure CPU-only workloads don't accidentally consume GPU nodes
3. Efficiently utilize expensive GPU capacity

## GPU Node Setup

### Node Labels and Taints
```bash
# GPU nodes are typically labeled automatically by the GPU device plugin or cloud provider
# Labels: kubernetes.io/accelerator=nvidia-tesla-v100
#         nvidia.com/gpu.product=Tesla-V100-SXM2-16GB

# Taint GPU nodes to prevent CPU workloads from landing there
kubectl taint nodes <gpu-node> dedicated=gpu:NoSchedule
```

### NVIDIA Device Plugin
GPU nodes need the NVIDIA device plugin DaemonSet — it registers GPUs as extended resources (`nvidia.com/gpu`) so the scheduler can see and allocate them.

```bash
kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml
```

## Requesting GPUs in a Pod

```yaml
apiVersion: v1
kind: Pod
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/accelerator
            operator: In
            values: [nvidia-tesla-v100]
  containers:
  - name: trainer
    image: pytorch/pytorch:2.0-cuda11.7
    resources:
      limits:
        nvidia.com/gpu: 2      # request 2 GPUs
        memory: "32Gi"
        cpu: "8"
      requests:
        nvidia.com/gpu: 2      # must equal limits for GPU resources
        memory: "32Gi"
        cpu: "8"
```

**Key rules for GPU resources:**
- GPU requests must equal limits (no overcommit)
- Integer values only (no fractional GPUs in standard K8s — need MIG or time-slicing for that)
- GPU resources are not visible to CPU pods — they need the toleration + nodeAffinity

## Distributed Training (Multi-GPU, Multi-Node)

Use the Kubeflow Training Operator (PyTorchJob):

```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: resnet-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          tolerations:
          - {key: dedicated, value: gpu, effect: NoSchedule}
          containers:
          - name: trainer
            image: my-trainer:v1
            resources:
              limits:
                nvidia.com/gpu: 4
    Worker:
      replicas: 3
      template:
        spec:
          tolerations:
          - {key: dedicated, value: gpu, effect: NoSchedule}
          containers:
          - name: trainer
            image: my-trainer:v1
            resources:
              limits:
                nvidia.com/gpu: 4
```

The operator creates 1 master + 3 worker Pods, configures `MASTER_ADDR`/`MASTER_PORT` env vars, and handles job completion/failure.

## GPU Utilization and Efficiency

```bash
# Check GPU utilization per node (requires DCGM exporter + Prometheus)
# Or SSH to node: nvidia-smi

# Check which pods have GPU allocations
kubectl get pods -o json | jq '.items[] | select(.spec.containers[].resources.limits["nvidia.com/gpu"] != null) | .metadata.name'
```

**Common GPU efficiency problems:**
- **GPU idle during data loading**: use multiple DataLoader workers; pin memory; prefetch to GPU
- **GPU fragmentation**: multiple small jobs each using 1 GPU on 4-GPU nodes wastes capacity — use Gang Scheduling (Volcano) to schedule all or none
- **GPU time-slicing**: enable `nvidia.com/gpu.sharing-strategy: time-slicing` for dev/test workloads that don't need a full GPU

## Interview Key Points

1. **Taint + Toleration + NodeAffinity** — the triple combination: taint prevents accidental scheduling; toleration allows it; nodeAffinity *forces* it to GPU nodes
2. **GPU resources must equal requests = limits** — no overcommit, unlike CPU/memory
3. **Device plugin** — GPUs are extended resources registered by the NVIDIA device plugin; without it, the scheduler doesn't know GPUs exist
4. **Kubeflow Training Operator** — the right way to run distributed training; not raw Jobs

## Sources
- [[K8S overview]]
- [[k8s/topics/scheduling]]
