# PersistentVolume / PersistentVolumeClaim

**Topic:** [[k8s/topics/storage]]
**Related:** [[k8s/concepts/statefulset]]

## What it is
A **PersistentVolume (PV)** represents a piece of storage in the cluster — an NFS share, a cloud disk (EBS/GCE PD), or a local SSD. A **PersistentVolumeClaim (PVC)** is a Pod's request for storage. The cluster binds them 1:1.

## Lifecycle

```
StorageClass defines how to provision
        │
        │ PVC created → StorageClass provisions PV automatically
        ▼
PV (Available) ──bind──→ PV (Bound) ──release──→ PV (Released)
                              │
                              │ PVC is mounted by Pod
                              ▼
                           Pod uses storage
```

## YAML Examples

```yaml
# PVC — what a Pod requests
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
---
# Use in Pod
volumes:
- name: data
  persistentVolumeClaim:
    claimName: training-data
volumeMounts:
- name: data
  mountPath: /data
```

## Access Mode Gotchas

- `ReadWriteOnce` (RWO): **one node** can mount read-write — NOT one Pod. Two Pods on the same node can both mount a RWO volume.
- `ReadWriteMany` (RWX): requires a network file system (NFS, CephFS, EFS). Cloud block volumes (EBS, GCE PD) do not support RWX.
- `ReadWriteOncePod` (RWOP): only one Pod cluster-wide (K8s 1.22+).

## Common Issues and Diagnosis

**PVC stuck in `Pending`:**
```bash
kubectl describe pvc <name>
# Look for: "no persistent volumes available for this claim"
# → No PV matches the request, or StorageClass can't provision
# → Check: StorageClass exists? Quotas exceeded? Cloud permissions?
```

**Pod stuck in `ContainerCreating` with volume error:**
```bash
kubectl describe pod <name>
# Events section: "Unable to attach or mount volumes"
# → PVC is Pending, or PV is in wrong AZ, or CSI driver issue
kubectl logs -n kube-system -l app=ebs-csi-controller
```

**AZ mismatch:**
- PV provisioned in `us-east-1a`, Pod scheduled to node in `us-east-1b`
- Fix: set `volumeBindingMode: WaitForFirstConsumer` in StorageClass

**Expanding a PVC:**
```bash
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
# StorageClass must have allowVolumeExpansion: true
# May require Pod restart for the filesystem to pick up the resize
```

## Sources
- [[K8S overview]]
