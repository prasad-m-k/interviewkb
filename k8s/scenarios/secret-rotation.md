# Scenario: Rotating Secrets Without Downtime

**Category:** Security / Operations
**Difficulty:** Hard
**Seen at:** Google, Amazon, Meta

## The Scenario
Your model serving Pods use a database password stored in a K8s Secret. The security team requires password rotation every 90 days. You need to rotate without restarting Pods or dropping requests.

## The Challenge

K8s Secrets injected as **environment variables** require a Pod restart to update. Secrets mounted as **files** (volumes) update automatically within ~1 minute (kubelet sync period).

The rotation strategy depends on how the secret is consumed.

## Approach 1: Volume-Mounted Secret (Preferred)

```yaml
volumes:
- name: db-credentials
  secret:
    secretName: db-password
    defaultMode: 0400
volumeMounts:
- name: db-credentials
  mountPath: /etc/secrets
  readOnly: true
```

**Rotation procedure:**
1. Update the Secret value:
   ```bash
   kubectl create secret generic db-password \
     --from-literal=password=NEW_PASSWORD \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
2. Wait ~1 minute for kubelet to propagate the new file to all Pods
3. Application must watch the file and reload credentials dynamically — implement a file watcher or re-read on each connection establishment

**Application code pattern:**
```python
import os, time
from threading import Thread

def watch_secret(path, reload_callback):
    last_mtime = 0
    while True:
        mtime = os.path.getmtime(path)
        if mtime > last_mtime:
            reload_callback()
            last_mtime = mtime
        time.sleep(10)
```

## Approach 2: External Secrets Operator (Best Practice)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h     # ESO syncs from vault every hour
  secretStoreRef:
    name: aws-secrets-manager-store
    kind: ClusterSecretStore
  target:
    name: db-password      # creates/updates this K8s Secret
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/ml-team/db-password
```

**Rotation procedure:**
1. Rotate in AWS Secrets Manager (the source of truth)
2. ESO automatically syncs to the K8s Secret within `refreshInterval`
3. kubelet propagates to mounted volumes within ~1 minute

No manual K8s operations needed.

## Approach 3: Dual-Secret Rolling Rotation (for env vars)

If you must use env vars (legacy apps), do a rolling rotation:

1. Set both old and new passwords as valid in the database (dual-credential period)
2. Update K8s Secret to new password
3. Trigger rolling restart: `kubectl rollout restart deployment/<name>`
4. Wait for rollout to complete (all Pods now have new password)
5. Revoke old password from database

```bash
kubectl rollout restart deployment/model-server
kubectl rollout status deployment/model-server
```

## Automated Rotation with RBAC

```yaml
# Service account that can update the specific secret
kind: Role
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-password"]   # restrict to ONE secret
  verbs: ["get", "update", "patch"]
```

## Interview Key Points

1. **Volume-mounted secrets auto-update; env vars don't** — design new apps to use volume mounts
2. **Application must reload credentials** — just updating the K8s Secret does nothing if the app doesn't re-read it
3. **External Secrets Operator** is the production-grade answer — vault is the source of truth
4. **Dual-credential period** — always rotate with overlap; never revoke before all Pods have the new credential

## Sources
- [[K8S overview]]
- [[k8s/concepts/configmap-secret]]
