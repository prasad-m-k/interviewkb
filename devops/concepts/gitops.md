# GitOps

**Topic:** [[devops/topics/ci-cd]]
**Related:** [[devops/concepts/ci-cd-pipeline]], [[devops/concepts/kubernetes-architecture]]

## What it is

GitOps = operating infrastructure and applications by using **Git as the single source of truth**. The desired state of the system lives in a Git repository. An automated agent continuously reconciles the actual state toward the desired state.

```
Developer             Git Repo               GitOps Agent          Cluster
   │                     │                      │                     │
   │── push manifest ──▶ │                      │                     │
   │                     │── trigger/poll ─────▶│                     │
   │                     │                      │── apply changes ──▶ │
   │                     │                      │◀── observe actual ──│
   │                     │                      │── reconcile loop ──▶│
```

**Key insight:** you never `kubectl apply` directly to production. You commit to Git; the agent applies.

---

## Four Principles (Weaveworks)

1. **Declarative** — desired system state expressed declaratively (K8s YAML, Helm charts, Kustomize)
2. **Versioned and immutable** — all desired state stored in Git; immutable versions; complete history
3. **Pulled automatically** — software agents (not humans) pull the desired state and apply it
4. **Continuously reconciled** — agents detect and correct drift between desired and actual state

---

## Push vs Pull Model

```
Push (traditional CI/CD):        Pull (GitOps):
CI pipeline ──kubectl apply──▶   Agent in cluster ──poll Git──▶ apply
  Cluster                        │
                                 Cluster
Problems:                        Benefits:
- CI needs cluster credentials   - Credentials stay in cluster
- Drift not detected             - Drift auto-corrected
- No continuous reconciliation   - Continuous reconciliation
```

---

## Tools: Argo CD vs Flux

| | Argo CD | Flux |
|---|---|---|
| **Model** | UI-first, rich dashboard | CLI/GitOps-first, lightweight |
| **Sync** | Push + pull | Pull only |
| **Multi-cluster** | Good (app-of-apps pattern) | Good (multi-tenant) |
| **Argo Rollouts** | Integrates natively | External |
| **CNCF status** | Graduated | Graduated |
| **Best for** | Teams that want UI observability | Teams that want GitOps purity |

### Argo CD basics

```yaml
# Application CRD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/infra
    targetRevision: HEAD
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true        # delete resources removed from Git
      selfHeal: true     # re-apply if someone manually changes the cluster
```

---

## Repo Structure Patterns

### Monorepo
```
infra-repo/
├── apps/
│   ├── my-app/
│   │   ├── base/         ← shared K8s manifests
│   │   └── overlays/
│   │       ├── staging/  ← staging-specific patches (Kustomize)
│   │       └── prod/     ← prod-specific patches
│   └── another-app/
└── platform/
    ├── monitoring/
    └── ingress/
```

### Separate app repo + infra repo
```
app-repo/        ← code + Dockerfile; CI builds + pushes image
infra-repo/      ← Helm values / K8s manifests; CI updates image tag; GitOps agent deploys
```

The **image updater** (Argo CD Image Updater / Flux Image Automation) watches the container registry and automatically opens PRs or commits to update the image tag in the infra repo.

---

## Common Interview Q&A

**Q: What is GitOps and how is it different from traditional CI/CD?**  
A: GitOps extends CI/CD by making Git the single source of truth for infrastructure state and using a pull-based agent for continuous reconciliation. Traditional CI/CD pushes changes to the cluster (requires cluster credentials in CI). GitOps uses a pull model where an in-cluster agent polls Git and applies — credentials stay in the cluster, drift is auto-corrected.

**Q: How do you handle secrets in a GitOps workflow?**  
A: Never commit plaintext secrets to the Git repo. Options:
1. **External Secrets Operator** — GitOps manages the ExternalSecret CRD; operator fetches actual secret from Vault/AWS
2. **Sealed Secrets** — encrypt K8s Secrets with a cluster-specific key; safe to commit ciphertext to Git
3. **SOPS** — encrypt secret files; decrypt in-cluster at apply time

**Q: What happens if someone manually changes a resource in the cluster?**  
A: With `selfHeal: true`, Argo CD detects the drift (by comparing live state to Git state) and reverts the manual change within seconds. This is a feature, not a bug — it enforces "Git is truth."

## Sources
- [[devops/topics/ci-cd]]
- [[devops/overview]]
