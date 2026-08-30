# SaaS to Self-Hosted Migration

**Difficulty:** Medium–Hard
**Topic:** Infrastructure design, migration
**Companies:** [[figure/index]]
**Related:** [[devops/concepts/deployment-strategies]], [[devops/concepts/secrets-management]], [[devops/topics/infrastructure-as-code]]

---

## Problem

Figure AI uses SaaS tools (GitHub.com, Jira Cloud, Confluence Cloud, possibly Datadog, Slack) and wants to migrate critical systems to self-hosted alternatives to keep sensitive IP within their own security perimeter. You are the Staff SRE leading this migration for 300+ engineers without disrupting active robot software development.

---

## Why This Migration Is Explicitly in the JD

This is not a cost-cutting exercise. Figure's motivations:

1. **IP protection** — Robot control software, ML model weights, and manufacturing processes are worth hundreds of millions of dollars. Having them on GitHub.com (a Microsoft-owned SaaS) means a GitHub breach = Figure's IP breach.
2. **Supply chain security** — CI/CD pipelines running on SaaS have attack surface that was weaponized in the SolarWinds and CodeCov attacks. Self-hosting shrinks that surface.
3. **Regulatory / contractual** — Automotive partners (BMW) may have data handling requirements that restrict where production-related data can reside.
4. **Long-term cost** — At scale, self-hosted can be cheaper than per-seat SaaS, though this is secondary at Figure's current headcount.

---

## Migration Scope (What Gets Migrated)

| SaaS Tool | Category | Self-Hosted Alternative | Priority |
|---|---|---|---|
| GitHub.com | SCM / Code Review | GitLab CE/EE or Gitea | Critical — robot software lives here |
| GitHub Actions | CI/CD | GitLab CI or self-hosted Buildkite / Jenkins | Critical — deployed with SCM |
| Jira Cloud | Issue tracking | Jira Data Center, Plane, or Linear (partial self-hosted) | High |
| Confluence Cloud | Documentation | Confluence Data Center or wiki.js | Medium |
| Datadog | Observability | Grafana + Prometheus + Loki stack | High |
| Docker Hub | Container registry | Harbor (self-hosted OCI registry) | High — robot images here |
| 1Password / LastPass | Secrets management | HashiCorp Vault | Critical — credential management |
| Slack | Comms | Stay SaaS (comms data is less sensitive) or Mattermost | Low |

Not everything needs to move. Communication tools (Slack, Zoom) carry low-sensitivity data. Focus migration energy on: code, CI/CD artifacts, container images, credentials, and issue tracking.

---

## Migration Strategy: SCM (GitHub → GitLab)

This is the highest-risk migration. 300+ engineers depend on SCM daily; any disruption halts development.

### Phase 1: Parallel Run (4 weeks)

Deploy GitLab instance. Do not cut over any active repositories yet.

```
Infrastructure setup:
  • GitLab on Kubernetes (gitlab/gitlab Helm chart) or dedicated VMs
  • Postgres for metadata, Gitaly for git storage, Redis for caching, MinIO for artifacts
  • Sizing: 300 users, active repos; recommend 32-core / 128GB RAM / 2TB NVMe for Gitaly
  • TLS termination, SSO integration (Azure AD / SAML)
  • Disaster recovery: daily backup to offsite object store, tested restore
  • Run a fake mirror of a non-critical repo to validate the setup under load
```

### Phase 2: Non-Critical Repos First (2 weeks)

Migrate internal tooling repos, documentation repos, and archived projects first.

```
Migration script per repo:
  git clone --mirror https://github.com/figure-ai/repo-name.git
  git push --mirror https://gitlab.figure.ai/figure-ai/repo-name.git
```

Preserve: branch history, tags, PRs (converted to GitLab MRs via GitHub API export), wiki pages, issue history (if co-migrating from Jira), CI/CD pipeline definitions (rewrite from GitHub Actions YAML to GitLab CI YAML).

### Phase 3: CI/CD Migration (2 weeks)

Do not migrate SCM and CI/CD simultaneously. Migrate CI/CD while GitHub is still the SCM, using GitLab CI runners pointed at GitHub repos. Validate that pipelines produce identical artifacts.

```
GitLab CI runner setup:
  • Self-hosted runners in Kubernetes (gitlab-runner Helm chart)
  • Dedicated runner pool for hardware-in-the-loop (physical machines)
  • Separate runner tag for simulation jobs (GPU-enabled nodes)
  • Runner autoscaling: scale to 0 when idle; scale to 50 during peak
```

### Phase 4: Critical Repos Cutover (1 weekend per batch)

For each batch of critical repos:
1. **Freeze period announcement:** 2-hour freeze on Friday evening (no merges to main)
2. **Final mirror:** sync GitHub → GitLab with `--mirror` to capture last commits
3. **DNS / redirect:** update `git remote` URLs; set GitHub repo to archived (read-only redirect)
4. **Validation:** engineering leads confirm they can clone, push, and trigger CI from GitLab
5. **Go/No-Go:** if validation passes, GitLab is live; GitHub is read-only fallback

Rollback plan: GitHub repos remain readable for 30 days post-cutover. If GitLab has a critical failure, engineers can revert remotes to GitHub temporarily.

### Phase 5: Decommission (30 days post-cutover)

Confirm no active traffic to GitHub. Delete CI/CD integrations. Cancel GitHub org subscription.

---

## Container Registry Migration (Docker Hub → Harbor)

Docker Hub's free tier has pull rate limits that break CI. Harbor is the CNCF-recommended self-hosted alternative.

```
Harbor setup:
  • Trivy scanner integration for automatic vulnerability scanning on every push
  • Replication: sync critical public base images FROM Docker Hub TO Harbor (cache)
  • Robots pull from harbor.figure.ai, not docker.io — eliminates external dependency
  • RBAC: per-project push permissions (robotics team, ML team, infrastructure)
  • Garbage collection: scheduled deletion of untagged manifests older than 90 days
```

Migration:
1. Stand up Harbor, test authentication, scanner, and replication
2. Push all current production images to Harbor (automated script with `skopeo copy`)
3. Update all CI/CD pipeline image references from `docker.io/figure/robot:v1.2` to `harbor.figure.ai/robot/robot:v1.2`
4. Update Kubernetes manifests and robot OTA configs to reference Harbor
5. Disable Docker Hub pull access (force all traffic through Harbor)

---

## Secrets Migration (SaaS → Vault)

Migrating from a SaaS secrets manager (1Password Teams, AWS Secrets Manager) to self-hosted HashiCorp Vault is the most operationally risky migration because it touches credentials used by every automated system.

```
Vault deployment:
  • HA mode: 3-node Vault cluster + etcd or Raft for storage backend
  • Unsealing: auto-unseal with Azure Key Vault (cloud KMS for unsealing key only)
  • Auth methods: Kubernetes service account auth (for CI/CD runners), OIDC (for humans)
  • Secret engines: KV v2 (static secrets), PKI (internal TLS certs), database dynamic credentials

Migration process:
  1. Inventory all secrets in current system (audit log export)
  2. Write secrets to Vault (automated import with change tracking)
  3. Update applications to read from Vault agent sidecar / Vault SDK (not from env files)
  4. Validate each application in staging before cutover
  5. Rotate all migrated credentials (old values in SaaS should be expired post-migration)
  6. Decommission SaaS secrets manager
```

**Critical point:** never have the same secret in both systems simultaneously for longer than the migration window. A breach of the old SaaS system after migration should not expose secrets already in Vault.

---

## Observability Stack Migration (Datadog → Self-Hosted)

Datadog is expensive at robot telemetry volume (thousands of custom metrics per robot × 1000 robots = millions of custom metrics/month = very large Datadog bill). Self-hosting the observability stack pays for itself quickly at this scale.

Target stack:
- **Metrics:** VictoriaMetrics (Prometheus-compatible, better write throughput than vanilla Prometheus)
- **Logs:** Grafana Loki (cost-effective; index-free; label-based)
- **Traces:** Grafana Tempo or Jaeger
- **Dashboards/Alerting:** Grafana + AlertManager + PagerDuty integration
- **Uptime:** Grafana synthetic monitoring or Uptime Kuma (simple self-hosted)

See [[figure/scenarios/robot-telemetry]] for the full stack design.

---

## Risk Management

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| SCM outage during cutover | Low | Critical | 2-hour freeze window; GitHub fallback for 30 days |
| Engineers push to wrong remote after cutover | Medium | Medium | Git hooks on GitHub to redirect; clear email announcement; update .gitconfig org-wide |
| CI artifacts reference old registry after Harbor migration | High | High | Lint pipeline YAMLs before cutover; automated scan for docker.io references |
| Vault unsealing fails after power outage | Low | Critical | Auto-unseal with cloud KMS; test quarterly |
| Data loss during SCM mirror | Very Low | Critical | Validate mirror with `git log --all` count comparison before decommission |
| Key rotation missed post-migration | Medium | High | Rotation checklist per service; track in Jira/Linear |

---

## Communication Plan

Engineers do not like surprise infrastructure changes. Mitigate resistance:
- **Announce 4 weeks before cutover:** reason (security), timeline, what changes for them (primarily git remote URLs)
- **Tooling:** provide a one-line script to update git remotes: `git remote set-url origin gitlab.figure.ai/...`
- **Office hours:** two open sessions for questions before cutover
- **War room:** #infra-migration Slack channel, on-call during cutover weekend
- **Rollback commitment:** publicly commit to a rollback plan and the conditions that trigger it

---

## SLOs for the Migrated Systems

After migration, set formal SLOs for systems that did not have them:

| System | SLI | Target |
|---|---|---|
| GitLab | Git push success rate | 99.9% |
| GitLab CI | Pipeline job start latency (P95) | < 2 minutes |
| Harbor | Image push/pull success rate | 99.9% |
| Vault | Secret read latency (P99) | < 100ms |
| Grafana | Dashboard load time (P95) | < 5 seconds |

---

## Interview Follow-Ups to Anticipate

- "How do you handle engineers who are resistant to leaving GitHub? It's the tool they know."
- "What happens if GitLab goes down the week after migration? Your rollback plan expired."
- "How would you prioritize which SaaS tools to migrate first if you have limited capacity?"
- "How do you ensure no credentials leaked between old SaaS secrets manager and new Vault?"
- "What is your testing strategy for Vault before you start migrating production secrets?"

---

## Sources
- [[devops/concepts/deployment-strategies]]
- [[devops/concepts/secrets-management]]
- [[devops/topics/infrastructure-as-code]]
- [[figure/company-context]]
- [[figure/staff-sre-role]]
