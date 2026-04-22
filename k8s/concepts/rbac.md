# RBAC (Role-Based Access Control)

**Topic:** [[k8s/topics/security]]
**Related:** [[k8s/concepts/configmap-secret]]

## What it is
RBAC controls which users, groups, and service accounts can perform which actions (verbs) on which Kubernetes resources. Enabled by default since K8s 1.6.

## The Four Objects

```
Subject ──────────────────────────── (who)
  │
  │  bound by RoleBinding
  │
  ▼
Role ─────────────────────────────── (what they can do)
  │  (defines: verbs on resources)
  │
  ▼
Resources in a Namespace ──────────── (where)
```

### ServiceAccount
The identity used by Pods to authenticate to the K8s API. Every namespace has a `default` ServiceAccount — but it should not be used (it may have excessive permissions from ClusterRoleBindings).

### Role / ClusterRole
A Role is namespace-scoped. A ClusterRole is cluster-wide (or grants access to non-namespaced resources like `nodes`).

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: model-reader
  namespace: ml-team
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

Common verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`

### RoleBinding / ClusterRoleBinding
Binds a Role to a Subject.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: model-server-binding
  namespace: ml-team
subjects:
- kind: ServiceAccount
  name: model-server-sa
  namespace: ml-team
roleRef:
  kind: Role
  name: model-reader
  apiGroup: rbac.authorization.k8s.io
```

**ClusterRoleBinding with a ClusterRole**: grants access cluster-wide.
**RoleBinding with a ClusterRole**: grants the ClusterRole's permissions but *only in that namespace*. Useful for reusing a common ClusterRole without granting cluster-wide access.

## Giving a Pod Least-Privilege API Access

```yaml
# 1. Create a ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: model-server-sa
  namespace: ml-team

# 2. Create a Role with only what's needed
---
kind: Role
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]

# 3. Bind
---
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: model-server-sa

# 4. Reference in Pod spec
---
spec:
  serviceAccountName: model-server-sa
  automountServiceAccountToken: false   # if the pod doesn't need API access at all
```

## Debugging RBAC

```bash
# Check what a service account can do
kubectl auth can-i get pods --as=system:serviceaccount:ml-team:model-server-sa

# List all RoleBindings for a ServiceAccount
kubectl get rolebindings -n ml-team -o yaml | grep model-server-sa

# Check effective permissions
kubectl auth can-i --list --as=system:serviceaccount:ml-team:model-server-sa -n ml-team
```

## Common Interview Angles
- "A Pod is getting 403 from the API server" → check ServiceAccount, Role, and RoleBinding; use `kubectl auth can-i`
- "How do you prevent Pods from accessing the K8s API?" → `automountServiceAccountToken: false`
- "What's the difference between Role and ClusterRole?" → namespace-scoped vs. cluster-wide; you can reference a ClusterRole in a RoleBinding to scope it to one namespace

## Sources
- [[K8S overview]]
