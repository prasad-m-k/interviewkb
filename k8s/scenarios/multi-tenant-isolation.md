# Scenario: Multi-Tenant K8s Cluster for ML Teams

**Category:** Architecture / Security
**Difficulty:** Hard
**Seen at:** Google, Meta, Amazon (platform/MLOps roles)

## The Scenario
You manage a shared K8s cluster used by 5 ML teams. You need to ensure each team:
1. Can only see and modify their own resources
2. Cannot accidentally consume another team's node capacity
3. Cannot send network traffic to another team's services
4. Gets a fair share of cluster GPU resources

## Layer 1: Namespaces (Isolation Unit)

```bash
kubectl create namespace team-alpha
kubectl create namespace team-beta
kubectl create namespace team-gamma
```

Namespaces provide a scope for names and the boundary for most K8s isolation mechanisms. They do **not** provide security isolation by themselves — that requires the additional layers below.

## Layer 2: RBAC (Access Control)

```yaml
# Each team gets a ServiceAccount and Role
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: team-alpha-sa
  namespace: team-alpha
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: team-alpha-role
  namespace: team-alpha
rules:
- apiGroups: ["", "apps", "batch"]
  resources: ["pods", "deployments", "jobs", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
kind: RoleBinding
subjects:
- kind: ServiceAccount
  name: team-alpha-sa
  namespace: team-alpha
roleRef:
  kind: Role
  name: team-alpha-role
```

**Key restriction**: no ClusterRole or ClusterRoleBinding — teams cannot see other namespaces.

## Layer 3: ResourceQuota (Capacity Fairness)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "40"
    requests.memory: 160Gi
    limits.cpu: "80"
    limits.memory: 320Gi
    requests.nvidia.com/gpu: "8"      # GPU quota
    limits.nvidia.com/gpu: "8"
    count/pods: "50"
    count/services: "20"
    persistentvolumeclaims: "10"
    requests.storage: 5Ti
```

A team exceeding its quota gets a `403 Forbidden` on new resource creation — it cannot starve other teams.

## Layer 4: LimitRange (Per-Pod Defaults and Ceilings)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  - type: Container
    default:           # applied if no limits set
      cpu: "1"
      memory: 2Gi
    defaultRequest:    # applied if no requests set
      cpu: "200m"
      memory: 256Mi
    max:               # ceiling for any single container
      cpu: "16"
      memory: 64Gi
    min:
      cpu: "100m"
      memory: 128Mi
```

Prevents a single Pod from requesting all of a team's quota.

## Layer 5: NetworkPolicy (Network Isolation)

```yaml
# Default deny all ingress in each namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: team-alpha
spec:
  podSelector: {}      # selects ALL pods in namespace
  policyTypes: [Ingress, Egress]
---
# Allow intra-team traffic only
kind: NetworkPolicy
metadata:
  name: allow-intra-team
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes: [Ingress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: team-alpha
```

Without this, a Pod in team-beta can freely call team-alpha's services — defeating namespace isolation.

## Layer 6: Node Isolation (Optional, for Strict Isolation)

For compliance or noisy-neighbor concerns, dedicate node pools per team:
```bash
kubectl taint nodes <node> team=alpha:NoSchedule
```

Then add the toleration to team-alpha Pods. More expensive but provides true compute isolation.

## The Multi-Tenancy Checklist

| Layer | K8s Mechanism | What it Protects |
|---|---|---|
| API isolation | RBAC + Namespaces | Team can't see/modify other namespaces |
| Compute fairness | ResourceQuota | Team can't consume all GPU/CPU |
| Workload safety | LimitRange | Single Pod can't take all the quota |
| Network isolation | NetworkPolicy | Teams can't call each other's services |
| Node isolation | Taints + Tolerations | Physical compute separation (optional) |

## Sources
- [[K8S overview]]
- [[k8s/topics/security]]
