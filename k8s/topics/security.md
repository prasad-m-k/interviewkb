# Kubernetes Security

**Topic:** [[k8s/topics/security]]
**Related:** [[k8s/concepts/rbac]], [[k8s/concepts/configmap-secret]]

## RBAC (Role-Based Access Control)

Controls who can do what to which Kubernetes resources.

Four key objects:
- **ServiceAccount**: an identity for a Pod (or process) to use when calling the K8s API
- **Role** / **ClusterRole**: defines a set of permissions (verbs on resources)
- **RoleBinding** / **ClusterRoleBinding**: assigns a Role to a Subject (user, group, or ServiceAccount)

`Role` + `RoleBinding` = namespace-scoped. `ClusterRole` + `ClusterRoleBinding` = cluster-wide.

**See:** [[k8s/concepts/rbac]]

---

## Secrets

Stores sensitive data (passwords, tokens, TLS certs). Base64-encoded (not encrypted) by default in etcd — enable etcd encryption at rest for real security.

Types: `Opaque`, `kubernetes.io/tls`, `kubernetes.io/dockerconfigjson`, `kubernetes.io/service-account-token`

**Best practices:**
- Mount as files, not env vars (env vars can leak via `/proc/<pid>/environ`)
- Use external secret managers (Vault, AWS Secrets Manager) with ESO (External Secrets Operator)
- Enable etcd encryption at rest
- Rotate regularly; use projected volumes with `expirationSeconds` for automatic token rotation

**See:** [[k8s/concepts/configmap-secret]]

---

## Pod Security

### Pod Security Admission (PSA) — K8s 1.25+
Replaced PodSecurityPolicy. Three built-in policy levels applied at namespace level via labels:

| Level | What it prevents |
|---|---|
| `privileged` | No restrictions |
| `baseline` | Blocks known privilege escalations (hostPID, hostNetwork, privileged containers, hostPath volumes) |
| `restricted` | Baseline + requires non-root, drops all capabilities, requires seccomp |

```yaml
# Label on namespace:
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/warn: restricted
```

### securityContext
Applied at Pod or container level:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
```

---

## ServiceAccount Token Projection

Pods automatically get a ServiceAccount token mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. Since K8s 1.20, these are **bound tokens** with:
- Expiry (`expirationSeconds`, default 1h)
- Audience binding
- Automatic rotation by kubelet

For workloads calling cloud APIs (AWS, GCP): use **Workload Identity** (GKE) or **IRSA** (EKS) to bind a Kubernetes ServiceAccount to an IAM role — no long-lived cloud credentials in Pods.

---

## Admission Controllers

Webhooks that intercept API server requests before persistence:
- **MutatingAdmissionWebhook**: can modify the request (e.g., inject a sidecar, add default resource limits)
- **ValidatingAdmissionWebhook**: can accept or reject (e.g., reject Pods without resource limits, enforce naming conventions)

Common tools: OPA/Gatekeeper, Kyverno.

---

## Network Security

NetworkPolicy defines which Pods can communicate. Combined with mTLS (Istio/Linkerd service mesh), you get both network-level and application-level encryption and authentication.

**See:** [[k8s/topics/networking]]

## Sources
- [[K8S overview]]
