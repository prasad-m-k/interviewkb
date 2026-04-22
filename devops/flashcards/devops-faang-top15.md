# DevOps FAANG — Top 15 Interview Q&A

**Type:** Flashcards
**Difficulty:** Mixed
**Coverage:** CI/CD, K8s, Observability, Security, Reliability

---

> [!question]- 1. What are the DORA metrics and what does "elite" look like?
> **The four metrics:**
> | Metric | Measures | Elite |
> |---|---|---|
> | Deployment Frequency | How often you ship | Multiple/day |
> | Lead Time for Changes | Commit → prod | < 1 hour |
> | Change Failure Rate | % deploys that cause incidents | < 5% |
> | MTTR | Mean time to restore | < 1 hour |
> 
> **Why they matter:** DORA research (Accelerate book) shows these four metrics predict organizational performance. High-performing teams score elite on all four simultaneously — they are not tradeoffs.
> 
> **Interview tip:** Know the four names cold, and know that elite = high frequency + low lead time + low failure rate + fast recovery.

---

> [!question]- 2. Blue-green vs canary vs rolling — when do you use each?
> **Blue-Green:**
> - Two identical environments; switch router from blue → green
> - Zero downtime; instant rollback (flip back)
> - Cost: 2× infrastructure running simultaneously
> - Use when: you need instant rollback, DB migration is risky (run on green only), or compliance requires clean cutover
> 
> **Canary:**
> - Gradually shift traffic (5% → 25% → 100%) with metric gates
> - Catches regressions before full rollout; slower than blue-green
> - Use when: you want data-driven confidence; ML models; high-risk changes
> 
> **Rolling:**
> - Replace instances one batch at a time; no extra infra
> - During rollout, two versions run simultaneously — must be backward-compatible
> - Use when: cost matters; changes are backward-compatible; rollback can be slow
> 
> **Feature flags:**
> - Code ships dark; flag enables it for % of users
> - Decouples deploy from release; instant kill switch
> - Use when: A/B testing; progressive exposure; emergency disable needed

---

> [!question]- 3. How does Kubernetes scheduling work?
> **Two phases:**
> 
> **Filter:** eliminate nodes that can't run the pod
> - Not enough CPU/memory (requests)
> - Node has taint the pod doesn't tolerate
> - Pod has nodeSelector/affinity that excludes the node
> - PVC can't be mounted on this node (zone mismatch)
> 
> **Score:** rank remaining nodes 0–100
> - Spread pods across zones (topology spread)
> - Least-requested node first (bin-packing or spreading)
> - Node affinity preference (soft rules)
> 
> **Scheduler selects highest-scoring node → binds pod → kubelet starts it.**
> 
> **Common interview follow-up:** "What's a taint/toleration?"
> Taint marks a node as special (e.g., `node=gpu:NoSchedule`). Only pods with the matching toleration can be scheduled there. Used to dedicate nodes to specific workloads.

---

> [!question]- 4. Explain the Kubernetes control loop / reconciliation
> **Core concept:** Kubernetes is a **declarative control system**. You describe desired state; controllers continuously reconcile actual state toward desired state.
> 
> **The loop:**
> ```
> Watch API server for resource changes
>   ↓
> Compare current state to desired state
>   ↓
> If different: take action (create/update/delete resources)
>   ↓
> Update status subresource
>   ↓
> Repeat (event-driven, not polling)
> ```
> 
> **Example:** Deployment controller watches Deployments. If `spec.replicas=3` but only 2 pods exist, it creates a ReplicaSet which creates the missing pod. If a pod dies, kubelet reports it, the controller creates a replacement.
> 
> **Why this matters:** idempotent, self-healing, eventual consistency. Running `kubectl apply` twice is safe — second apply is a no-op if state matches.

---

> [!question]- 5. What's the difference between liveness, readiness, and startup probes?
> **Liveness probe:** "Is the container alive?"
> - Failure → K8s kills and restarts the container (CrashLoopBackOff if repeated)
> - Use for: deadlocks, infinite loops — the process is running but stuck
> - Risk: if misconfigured (too aggressive), causes unnecessary restarts
> 
> **Readiness probe:** "Is the container ready to serve traffic?"
> - Failure → K8s removes the pod from Service endpoints (no traffic routed)
> - Container is NOT killed; stays running but gets no requests
> - Use for: warmup time, dependency checks, circuit breaker pattern
> 
> **Startup probe:** "Has the container finished starting?"
> - Disables liveness/readiness checks until startup probe passes
> - Use for: slow-starting apps (JVM, ML model loading) — prevents liveness from killing them during boot
> 
> **Rule of thumb:** Always define readiness. Add liveness only if you have actual deadlock risk. Add startup only if startup time > `initialDelaySeconds`.

---

> [!question]- 6. What are the four golden signals?
> From Google SRE Book — the minimum viable monitoring for any service:
> 
> | Signal | Definition | Example metric |
> |---|---|---|
> | **Latency** | Time to serve a request | p50/p99 response time |
> | **Traffic** | Demand on the system | Requests per second |
> | **Errors** | Rate of failed requests | 5xx rate, timeout rate |
> | **Saturation** | How "full" the service is | CPU %, connection pool % |
> 
> **Why four?** Together they tell you: are users waiting? (latency), how many? (traffic), are requests failing? (errors), are we about to fall over? (saturation).
> 
> **Companion frameworks:**
> - **RED** (Rate, Errors, Duration) — for request-driven microservices
> - **USE** (Utilization, Saturation, Errors) — for infrastructure resources (CPU, disk, network)

---

> [!question]- 7. Explain SLI, SLO, SLA, and error budget
> **SLI (Service Level Indicator):** A metric that measures the thing you care about.
> Example: `success_rate = successful_requests / total_requests`
> 
> **SLO (Service Level Objective):** Your internal target for that metric.
> Example: `success_rate >= 99.9%` over a rolling 30-day window
> 
> **Error budget:** `100% - SLO = budget for failures`
> 99.9% SLO → 0.1% budget → ~43 minutes/month of allowed downtime
> 
> **SLA (Service Level Agreement):** External contract with customers, with financial penalties.
> SLA is typically looser than SLO (SLO = 99.9%, SLA = 99.5%)
> 
> **How to use error budget:**
> - Budget healthy → ship features aggressively
> - Budget burning fast → freeze non-critical deploys, fix reliability
> - Budget exhausted → all hands on reliability until budget recovers
> 
> **Interview insight:** Error budgets align product and SRE incentives — product wants features, SRE wants reliability, the budget is the negotiated middle ground.

---

> [!question]- 8. How do you manage secrets in a Kubernetes cluster?
> **Four approaches (know the tradeoffs):**
> 
> **1. Kubernetes Secrets (baseline)**
> - Base64-encoded, stored in etcd
> - Problem: readable by anyone with RBAC access; not encrypted at rest by default
> - Enable: `EncryptionConfiguration` with KMS provider
> 
> **2. HashiCorp Vault**
> - Secrets stored in Vault, not K8s
> - Dynamic secrets: Vault generates time-limited DB credentials per request
> - App authenticates via Vault Agent sidecar or Vault injector
> - Best for: large orgs, compliance requirements, secret rotation
> 
> **3. External Secrets Operator**
> - K8s operator syncs secrets from AWS Secrets Manager / GCP Secret Manager / Azure Key Vault
> - Creates a native K8s Secret automatically; app sees a normal Secret
> - Best for: cloud-native stacks; integrates with cloud IAM
> 
> **4. Sealed Secrets**
> - Encrypt secrets with a cluster public key; store encrypted in Git
> - `kubeseal` CLI seals; controller unseals in-cluster
> - Best for: GitOps (sealed secret committed alongside deployment YAML)
> 
> **Never:** store plaintext secrets in Git, environment variables in Dockerfiles, or CI/CD pipeline logs.

---

> [!question]- 9. What is GitOps and how is it different from traditional CI/CD?
> **GitOps** = Git is the single source of truth for both application code AND infrastructure state.
> 
> **Four principles:**
> 1. Desired state declared in Git (Kubernetes manifests, Helm values)
> 2. Desired state is stored in Git (with full history)
> 3. Desired state is continuously applied by an agent (Argo CD, Flux)
> 4. Divergence triggers automated reconciliation (drift is auto-corrected)
> 
> **Traditional CI/CD (push model):**
> ```
> CI pipeline → kubectl apply → K8s   (pipeline pushes state)
> ```
> 
> **GitOps (pull model):**
> ```
> CI pipeline → updates config repo → Argo CD detects change → Argo CD applies to K8s
> ```
> 
> **Benefits of pull model:**
> - CI pipeline never needs K8s credentials
> - Rollback = revert git commit
> - Drift is auto-corrected (Argo CD resync)
> - Audit trail lives in Git

---

> [!question]- 10. A developer committed a secret to a public GitHub repo. First action?
> **REVOKE IMMEDIATELY — before anything else.**
> 
> The secret must be treated as compromised from the moment it was committed — it may have been scraped by automated bots within seconds of the push.
> 
> ```bash
> # AWS key
> aws iam delete-access-key --access-key-id AKIAIOSFODNN7EXAMPLE --user-name ci-user
> ```
> 
> **Then (in order):**
> 1. Check CloudTrail for abuse during the exposure window
> 2. Issue replacement credentials
> 3. Scope exposure (when was it committed, public/private, any forks)
> 4. Clean Git history with `git filter-repo` (hygiene — doesn't undo scraping)
> 5. Notify security team; file incident report
> 6. Add pre-commit hooks (detect-secrets) to prevent recurrence
> 
> **Wrong answer:** "First I'd remove the secret from Git history." — this is too late and only hygiene.

---

> [!question]- 11. What's the difference between horizontal and vertical scaling?
> **Vertical scaling (scale up):**
> - Add more CPU/RAM to an existing instance
> - Simple: no code changes, no load balancer changes
> - Limits: finite (can't add infinite CPUs); single point of failure; downtime to resize
> - Use for: stateful systems (DB primary), when horizontal is impossible
> 
> **Horizontal scaling (scale out):**
> - Add more instances of the same size
> - Resilient: no SPOF; can scale to near-arbitrary capacity
> - Requires: stateless application; load balancer; distributed session/cache
> - Use for: stateless app servers, API services, queue consumers
> 
> **K8s HPA (Horizontal Pod Autoscaler):** scales pods based on CPU/memory/custom metrics
> **K8s Cluster Autoscaler:** scales nodes when pods are Pending due to resource constraints
> **KEDA:** scales pods based on event sources (SQS queue depth, Kafka lag, cron)
> 
> **At 10× traffic:** scale out first (faster, cheaper); scale up DB if connection limits hit.

---

> [!question]- 12. Explain the expand-contract pattern for zero-downtime DB migrations
> **Problem:** Dropping or renaming a DB column breaks the old code that's still running during a rolling deploy.
> 
> **Solution: three-phase expand-contract**
> 
> **Phase 1 — Expand (backward-compatible):**
> ```sql
> ALTER TABLE orders ADD COLUMN new_status VARCHAR(50) NULL;
> ```
> Deploy new code that writes both `old_status` and `new_status`; reads `new_status` with fallback to `old_status`. Old code still works (ignores new column).
> 
> **Phase 2 — Migrate:**
> ```sql
> UPDATE orders SET new_status = map(old_status) WHERE new_status IS NULL;
> ```
> Deploy code that reads only `new_status`. Old code still works.
> 
> **Phase 3 — Contract (cleanup):**
> ```sql
> ALTER TABLE orders DROP COLUMN old_status;
> ```
> Safe: no code reads `old_status` anymore.
> 
> **Rules:** Never add `NOT NULL` column without default in the same deploy as code. Never drop a column in the same deploy as removing its usage. Never rename in one step.

---

> [!question]- 13. What is a service mesh and when would you use one?
> **Service mesh** = infrastructure layer that handles service-to-service communication transparently, without application code changes.
> 
> **How it works:** sidecar proxy (Envoy) injected alongside every pod. All traffic goes through the sidecar, not directly between pods.
> 
> **What it provides:**
> | Feature | Without mesh | With mesh |
> |---|---|---|
> | mTLS between services | Manual cert management per service | Automatic, rotated |
> | Distributed tracing | Instrument every service | Automatic span propagation |
> | Traffic splitting | Redeploy service | Config change in mesh |
> | Circuit breaking | Each service implements its own | Centrally configured |
> | Retries/timeouts | Per-service code | Mesh policy |
> 
> **When to use:** 15+ services, compliance requires encryption in transit, you need sophisticated traffic management (canary at L7, A/B testing), or you want observability without instrumenting every service.
> 
> **When not to use:** < 5 services; operational complexity isn't worth it; team unfamiliar with Istio/Linkerd debugging.
> 
> **Tools:** Istio (most features, most complex), Linkerd (simpler, lighter), Cilium (eBPF-based, no sidecar).

---

> [!question]- 14. What happens when you run `kubectl apply -f deployment.yaml`?
> **Full flow:**
> 
> 1. `kubectl` reads the YAML → sends `PATCH`/`PUT` to `kube-apiserver`
> 2. **API server:** authenticates (cert/token), authorizes (RBAC), validates (admission webhooks), persists to **etcd**
> 3. **Deployment controller** (in controller-manager) watches etcd via informer → sees the Deployment changed → creates/updates a **ReplicaSet**
> 4. **ReplicaSet controller** sees desired pods ≠ current pods → creates Pod objects in etcd
> 5. **Scheduler** watches for Pods with no `spec.nodeName` → runs filter+score → writes `nodeName` to the Pod
> 6. **kubelet** on the assigned node watches for Pods assigned to it → calls container runtime (containerd) → pulls image → starts container
> 7. **kubelet** reports pod status back to API server
> 8. **Endpoint controller** adds the new pod's IP to the Service's Endpoints if readiness probe passes
> 9. **kube-proxy** on each node syncs iptables/IPVS rules with the new Endpoints
> 
> **Key insight:** nothing talks to anything else directly — every component watches the API server. This is the reconciliation loop.

---

> [!question]- 15. How do you do a production outage postmortem?
> **Blameless postmortem — the Google SRE approach:**
> 
> **When:** within 24–48 hours of incident resolution, while memory is fresh.
> 
> **Structure:**
> ```
> Title: [Date] [Severity] [Brief description]
> Duration: 14:28 – 15:03 (35 minutes)
> Impact: 40% error rate on checkout; ~12,000 users affected
> 
> ## Timeline (from logs, not memory)
> 14:28 - Deploy of v2.3.1 to prod (rolling update starts)
> 14:32 - First alerts: error rate 40%
> 14:35 - Incident declared P1; rollback initiated
> 14:41 - Rollback complete; error rate back to baseline
> 15:03 - Incident closed; all-clear
> 
> ## Root Cause
> New code path in PaymentService didn't handle null card_type;
> threw NullPointerException for ~30% of transactions.
> 
> ## Contributing Factors
> - Integration tests didn't cover null card_type case
> - Canary analysis only checked error rate (not null handling)
> 
> ## Action Items
> [ ] Add null card_type test cases — @engineer — due 2026-04-28
> [ ] Add error rate threshold to canary template — @sre — due 2026-04-25
> ```
> 
> **Blameless rule:** Action items target processes and tooling, never "developer should be more careful." The question is always: what would have caught this automatically?

## Sources
- [[devops/overview]]
- [[devops/topics/ci-cd]]
- [[devops/concepts/kubernetes-architecture]]
- [[devops/topics/observability]]
- [[devops/concepts/secrets-management]]
- [[devops/concepts/gitops]]
- [[devops/patterns/incident-response]]
- [[devops/patterns/zero-downtime-deployment]]
- [[devops/scenarios/production-outage]]
- [[devops/scenarios/secret-exposed]]
