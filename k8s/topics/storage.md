# Kubernetes Storage

**Topic:** [[k8s/topics/storage]]
**Related:** [[k8s/concepts/persistent-volume]], [[k8s/topics/workloads]]

## The Storage Object Hierarchy

```
StorageClass (provisioner config)
    │
    │  dynamic provisioning
    ▼
PersistentVolume (actual storage resource)
    ▲
    │  bound (1:1 relationship)
    │
PersistentVolumeClaim (request for storage)
    ▲
    │  mounted by
    │
Pod (uses the volume)
```

## PersistentVolume (PV)

A cluster-level resource representing a piece of storage. Can be:
- **Statically provisioned**: admin creates PV manually
- **Dynamically provisioned**: created automatically by a StorageClass when a PVC is created

Key fields: `capacity`, `accessModes`, `storageClassName`, `persistentVolumeReclaimPolicy`

## PersistentVolumeClaim (PVC)

A namespace-level resource that *requests* storage. The cluster finds a matching PV (by `accessModes`, `storageClassName`, and `capacity >= request`) and binds them.

**PVC binding is 1:1**: a PV bound to a PVC is unavailable to any other PVC.

## Access Modes

| Mode | Abbreviation | Meaning |
|---|---|---|
| `ReadWriteOnce` | RWO | One node can mount read-write |
| `ReadOnlyMany` | ROX | Many nodes can mount read-only |
| `ReadWriteMany` | RWX | Many nodes can mount read-write |
| `ReadWriteOncePod` | RWOP | Only one Pod (K8s 1.22+) |

**Most cloud block volumes (EBS, GCE PD) are RWO** — they can only attach to one node. For shared storage across Pods, you need NFS, CephFS, EFS, or a cloud file system (RWX).

## StorageClass

Defines a provisioner and its parameters. Enables dynamic provisioning.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
reclaimPolicy: Delete          # or Retain
volumeBindingMode: WaitForFirstConsumer   # or Immediate
```

`WaitForFirstConsumer`: PV is provisioned only when a Pod using the PVC is scheduled. Avoids AZ mismatch (PV provisioned in us-east-1a, Pod scheduled to us-east-1b).

## Reclaim Policy

What happens to the PV when its PVC is deleted:
- `Delete`: PV and underlying storage are deleted (cloud default)
- `Retain`: PV is kept, must be manually reclaimed (good for important data)

## Volume Types

| Volume | Use case |
|---|---|
| `emptyDir` | Temporary scratch space; survives container restarts, deleted when Pod ends |
| `hostPath` | Mount a node file/directory — dangerous, avoid in production |
| `configMap` / `secret` | Mount config/credentials as files |
| `persistentVolumeClaim` | Durable storage via PVC |
| `projected` | Combine multiple sources (SA token, ConfigMap, Secret) into one mount |

## StatefulSet and Storage

StatefulSets use `volumeClaimTemplates` to create a PVC per Pod. Each Pod gets its own stable PVC:
- `data-myapp-0`, `data-myapp-1`, ...
- If a StatefulSet Pod is deleted and recreated, it reattaches to the *same* PVC
- Scaling down a StatefulSet does **not** delete PVCs (data is preserved)

## Common Storage Issues

| Issue | Cause | Fix |
|---|---|---|
| PVC stuck `Pending` | No matching PV / StorageClass missing / quota exceeded | Check `kubectl describe pvc`; verify StorageClass exists; check ResourceQuota |
| Pod stuck `ContainerCreating` | PVC Pending or volume not attachable to node | Check PVC status; check CSI driver logs |
| AZ mismatch | PV in different zone than Pod's node | Use `WaitForFirstConsumer`; add node affinity to PVC |
| `ReadWriteOnce` used across nodes | Block volume can't attach to 2 nodes | Switch to RWX (EFS/NFS) or redesign for per-pod storage |

## Sources
- [[K8S overview]]
