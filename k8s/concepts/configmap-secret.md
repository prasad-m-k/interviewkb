# ConfigMap and Secret

**Topic:** [[k8s/topics/security]]
**Related:** [[k8s/concepts/rbac]], [[k8s/concepts/pod]]

## ConfigMap

Stores non-sensitive configuration as key-value pairs or files. Decouples config from container images.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: model-config
data:
  MODEL_VERSION: "v1.2"
  MAX_BATCH_SIZE: "32"
  config.yaml: |
    model:
      version: v1.2
      batch_size: 32
      timeout_ms: 500
```

**Consuming a ConfigMap:**
```yaml
# As environment variables
envFrom:
- configMapRef:
    name: model-config

# As a mounted file
volumes:
- name: config
  configMap:
    name: model-config
volumeMounts:
- name: config
  mountPath: /etc/config
  readOnly: true
```

**Dynamic updates**: when mounted as a volume, ConfigMap changes propagate to the Pod *eventually* (kubelet sync period, ~1 minute). Env var injections do NOT update — require Pod restart.

---

## Secret

Same structure as ConfigMap but values are **base64-encoded** (not encrypted). Designed for sensitive data.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: dXNlcm5hbWU=   # base64("username")
  password: cGFzc3dvcmQ=   # base64("password")
stringData:                  # auto-encodes to base64
  api_key: "my-secret-key"
```

## Secret Security Realities

| Risk | Mitigation |
|---|---|
| Secrets are base64 in etcd (not encrypted by default) | Enable etcd encryption at rest (`EncryptionConfiguration`) |
| Any Pod in the namespace can read any Secret | RBAC: restrict `get`/`list` on secrets to specific ServiceAccounts |
| Secrets in env vars leak via `/proc/<pid>/environ` | Mount as files instead |
| Secret value visible in `kubectl describe` (truncated) and `get -o yaml` | Restrict `kubectl get secret` with RBAC |

## External Secrets Operator (ESO)

Best practice: store secrets in an external vault (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager) and sync into K8s Secrets via ESO. This way:
- Rotation happens in the vault; ESO syncs automatically
- K8s Secrets are ephemeral and auto-rotated
- Audit trail exists in the external vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: prod/db/password
```

## Common Interview Angles
- "How do you rotate a secret without downtime?" → update the external vault; ESO syncs to K8s Secret; if mounted as volume, propagates automatically; if env var, need rolling restart
- "What's the difference between ConfigMap and Secret?" → intent + slight security treatment; both are key-value stores; Secret has base64 encoding and stricter RBAC by convention
- "How do you prevent secrets from being checked into Git?" → never apply raw Secret YAML; use sealed-secrets, ESO, or Helm secrets plugin

## Sources
- [[K8S overview]]
