# Secrets Management

**Topic:** [[devops/topics/security-devsecops]]
**Related:** [[devops/concepts/gitops]], [[devops/concepts/kubernetes-architecture]]

## The Rule

**Never put secrets in:**
- Source code or config files committed to Git
- Docker image layers (`ENV MY_SECRET=...` in Dockerfile)
- Plain Kubernetes Secrets without encryption at rest
- CI/CD environment variable logs (mask them)
- Container stdout/logs

---

## Why K8s Secrets Aren't Actually Secret by Default

Kubernetes Secrets are **base64-encoded, not encrypted**. Anyone with `get secret` RBAC permission can decode them instantly. etcd stores them unencrypted unless you enable **EncryptionConfiguration**.

```bash
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
# Instantly reveals the plaintext value
```

**Fixes:**
1. Enable etcd encryption at rest (`EncryptionConfiguration` with AES-256 or KMS provider)
2. RBAC: restrict `get/list secrets` to only the service accounts that need it
3. Use an external secrets solution (secrets never live in etcd at all)

---

## Solution 1: HashiCorp Vault

Vault is a secrets-as-a-service system. Apps authenticate and fetch secrets dynamically.

```
App ──▶ Vault (auth with K8s service account token)
         │── returns: short-lived DB credentials (lease: 1 hour)
         │── auto-renews or re-issues
         │── on app death: credentials auto-expire
```

**Dynamic secrets:** Vault generates a unique DB username/password per app instance. When the lease expires, Vault revokes those credentials. Compromise is scoped to one instance for < 1 hour.

```bash
# App fetches secret at runtime
vault kv get -format=json secret/my-app/db-password

# Kubernetes auth
vault write auth/kubernetes/login \
  role="my-app" \
  jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"
```

---

## Solution 2: External Secrets Operator (K8s)

Sync secrets from an external store (Vault, AWS Secrets Manager, GCP Secret Manager) into K8s Secrets. GitOps-compatible: you commit the `ExternalSecret` CRD (which just says "fetch from here"), not the actual secret value.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-credentials    # creates this K8s Secret
  data:
    - secretKey: password
      remoteRef:
        key: secret/my-app/db    # Vault path
        property: password
```

---

## Solution 3: Sealed Secrets

Encrypt K8s Secret manifests with a cluster-specific public key. The ciphertext is safe to commit to Git. Only the in-cluster Sealed Secrets controller (holding the private key) can decrypt.

```bash
# Encrypt
kubectl create secret generic my-secret \
  --from-literal=password=s3cr3t \
  --dry-run=client -o yaml | kubeseal > my-sealed-secret.yaml

# Commit my-sealed-secret.yaml to Git (safe!)
# Controller decrypts and creates the K8s Secret automatically
```

---

## Solution 4: SOPS + Age/KMS

Encrypt individual secret files in Git. Decrypt at deploy time using an Age key or KMS.

```bash
# Encrypt with Age
sops --encrypt --age age1abc...xyz secrets.yaml > secrets.enc.yaml

# Commit secrets.enc.yaml to Git
# CI decrypts using the Age private key from a secure location
sops --decrypt secrets.enc.yaml | kubectl apply -f -
```

---

## Rotation Strategy

| Secret type | Rotation frequency | Rotation mechanism |
|---|---|---|
| DB passwords | Every 30–90 days (or on breach) | Vault dynamic secrets (auto) |
| API keys | Every 90 days or on engineer offboarding | Manual + automated where API supports |
| TLS certificates | Before expiry (Let's Encrypt: 90 days) | cert-manager auto-renews |
| Service account tokens | K8s auto-rotates | Built-in |
| SSH keys | On employee offboarding | Forced rotation |

**cert-manager** automates TLS certificate issuance and renewal from Let's Encrypt or internal CAs.

---

## Common Interview Q&A

**Q: How do you handle secret rotation without downtime?**  
A: Use two-phase rotation:
1. Add new credential (write access with new secret)
2. Deploy new version reading new secret (old version still uses old secret)
3. Verify new version works
4. Revoke old credential

With Vault dynamic secrets, this is automatic — each instance gets its own credential with a short TTL.

**Q: Where should CI/CD pipeline secrets (deploy keys, registry credentials) live?**  
A: In the CI system's secret store (GitHub Actions Secrets, GitLab CI Variables, Vault for Jenkins). Never hardcoded in workflow YAML. Prefer OIDC-based authentication (CI gets a short-lived token from cloud IAM, no static credential needed).

## Sources
- [[devops/topics/security-devsecops]]
- [[devops/overview]]
