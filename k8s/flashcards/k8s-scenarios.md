# Flashcards — Kubernetes Scenario-Based Interview Questions

**Topic:** [[k8s/topics/architecture]], [[k8s/topics/workloads]], [[k8s/topics/networking]], [[k8s/topics/security]]
**Created:** 2026-04-21

Format: read the scenario, formulate your answer, then expand the callout to check.

---

## ARCHITECTURE & CONTROL PLANE

### Card A1
**Q: Walk me through everything that happens when you run `kubectl apply -f deployment.yaml`.**

> [!note]- Answer
> 1. **kubectl** reads the YAML, converts to JSON, sends a `PATCH` (or `POST` if new) to the **API server** over HTTPS
> 2. **API Server** authenticates (cert/token), authorizes (RBAC), validates the schema, runs admission webhooks (mutating → validating), then **writes to etcd**
> 3. The **Deployment controller** (in controller-manager) receives a watch event: "Deployment changed." It computes the desired ReplicaSet spec and creates/updates a **ReplicaSet**
> 4. The **ReplicaSet controller** receives a watch event: "ReplicaSet has 0 Pods, needs 3." It creates **Pod objects** in etcd (unscheduled — no `nodeName`)
> 5. The **Scheduler** watches for unscheduled Pods. For each, it filters nodes (resources, taints, affinity), scores them, and writes `nodeName` to the Pod spec in etcd
> 6. The **kubelet** on the target node watches for Pods assigned to it. It calls the **container runtime** (containerd) to pull the image and start the container
> 7. kubelet reports Pod status back to the API server. The **ReplicaSet controller** sees 3 Pods Running → desired state achieved
>
> *See also:* [[k8s/topics/architecture]]

---

### Card A2
**Q: The etcd cluster loses quorum. What still works in Kubernetes? What breaks?**

> [!note]- Answer
> **Still works:**
> - **Running Pods continue running** — kubelet doesn't need etcd to keep containers alive; it caches Pod specs locally
> - **Existing network rules** — kube-proxy iptables rules are already installed; Service routing continues
> - **Application traffic** — if your app is running, it keeps running; end users see no disruption
>
> **Breaks immediately:**
> - **API server** returns errors on most operations (read requests may work if cached, writes fail)
> - **kubectl** commands fail or hang
> - **No new Pods can be scheduled** — scheduler can't write to API server → etcd
> - **No self-healing** — if a Pod crashes, the ReplicaSet controller can't create a replacement
> - **No config updates** — new deployments, ConfigMap changes, Secret rotations all fail
>
> **Key insight:** The control plane is the brain; the data plane (running workloads) keeps running without it — until something needs to change.
>
> *See also:* [[k8s/topics/architecture]]

---

### Card A3
**Q: What is the difference between a Deployment, a ReplicaSet, and a Pod? When does the Deployment controller act vs. the ReplicaSet controller?**

> [!note]- Answer
> **Pod**: the actual running workload. One or more containers. Created by a ReplicaSet.
>
> **ReplicaSet**: ensures N Pods matching a selector are running. If a Pod dies, creates a replacement. Only manages *how many* — doesn't know about rolling updates.
>
> **Deployment**: manages ReplicaSets. Knows about *versions* and rolling updates. Creates a new ReplicaSet for each new Pod template, then scales it up while scaling the old one down.
>
> **Who acts when:**
> - You change replica count → **ReplicaSet controller** scales Pods
> - You change the Pod template (new image) → **Deployment controller** creates a new ReplicaSet, then replicaSet controller manages the Pod count in each
> - A Pod crashes → **ReplicaSet controller** creates a replacement immediately
> - You run `kubectl rollout undo` → **Deployment controller** scales old ReplicaSet back up, scales new one down
>
> **Why this separation?** Single responsibility. You can have a ReplicaSet without a Deployment (but you shouldn't — Deployments give you rollback history).

---

## POD LIFECYCLE & DEBUGGING

### Card B1
**Q: Your Pod is in CrashLoopBackOff. Walk through your complete debug methodology.**

> [!note]- Answer
> **Step 1:** `kubectl describe pod <name>` → Events section + Last State + Exit Code
>
> **Exit code triage:**
> - `1` → application error; check `kubectl logs <pod> --previous`
> - `137` → OOMKilled (128 + SIGKILL); raise memory limit
> - `127` → binary not found in image; check command/entrypoint
> - `2` → shell misuse; check command/args
>
> **Step 2:** `kubectl logs <pod> --previous --tail=100` — logs from the crashed container
>
> **Step 3 (no logs — crash before first line):**
> - Temporarily change `command` to `["sleep", "3600"]` to exec into the container and debug
>
> **Step 4: Common non-application causes:**
> - Liveness probe too aggressive (`initialDelaySeconds` too low) → use `startupProbe`
> - Missing ConfigMap/Secret → `kubectl get configmap <name>`, `kubectl get secret <name>`
> - PVC not mounted → `kubectl get pvc`
> - Wrong image entrypoint → check `kubectl describe pod | grep Image`
>
> **Key principle:** CrashLoopBackOff means "container exited, K8s restarted it, it exited again." The fix depends entirely on *why* it exited.
>
> *See also:* [[k8s/scenarios/crashloopbackoff]]

---

### Card B2
**Q: A Pod is stuck in Pending. Name the 6 possible causes and how you identify each.**

> [!note]- Answer
> Always start: `kubectl describe pod <name>` → Events section. The scheduler writes the exact reason.
>
> | Cause | Event Message | Fix |
> |---|---|---|
> | **Insufficient resources** | `0/N nodes available: N Insufficient cpu/memory` | Reduce requests or add nodes |
> | **Taint mismatch** | `N node(s) had untolerated taint` | Add toleration to Pod or remove taint |
> | **Affinity mismatch** | `N node(s) didn't match Pod's node affinity` | Fix nodeAffinity or node labels |
> | **PVC Pending** | `persistentvolumeclaim "x" not found / Pending` | Fix StorageClass or create PV |
> | **No nodes available** | `no nodes available / all nodes are NotReady` | Check node health, uncordon |
> | **ResourceQuota exceeded** | `exceeded quota` | Clean up resources or increase quota |
>
> *See also:* [[k8s/scenarios/pod-pending]]

---

### Card B3
**Q: How do liveness, readiness, and startup probes differ? What happens when each fails? Give an example of misusing liveness when readiness is correct.**

> [!note]- Answer
> | Probe | Failure Action | When to use |
> |---|---|---|
> | **liveness** | Kill + restart container | App can deadlock; only restart fixes it |
> | **readiness** | Remove from Service endpoints; NOT restarted | App is temporarily unable to serve (loading, upstream down) |
> | **startup** | Kill + restart; disables liveness/readiness until passes | Slow-starting apps (JVM, large ML model load) |
>
> **The misuse scenario:** A model-serving app becomes temporarily overloaded (high CPU, slow responses). A liveness probe fails → K8s restarts the Pod → the restarting Pod must reload the model → all pods restart simultaneously → thundering herd → cluster-wide outage.
>
> **Correct design:** Use `readinessProbe` to remove the overloaded Pod from rotation temporarily. It recovers on its own. Liveness is only for apps that are *permanently broken* without a restart (deadlock, file descriptor leak).
>
> **startupProbe trick:** `failureThreshold: 30, periodSeconds: 10` → allows up to 300s startup time without being killed by liveness.
>
> *See also:* [[k8s/topics/observability]]

---

### Card B4
**Q: Your Pod shows Exit Code 137. What happened, how do you confirm it, and what are the fix strategies?**

> [!note]- Answer
> **Exit code 137 = OOMKilled** (128 + 9 SIGKILL). The Linux kernel OOM killer terminated the container because it exceeded its memory `limit`.
>
> **Confirm:**
> ```
> kubectl describe pod <name>
> # Last State: Terminated  Reason: OOMKilled  Exit Code: 137
> kubectl top pod <pod> --containers  # compare usage vs limit
> ```
>
> **Root cause branches:**
> 1. **Limit set too low** → raise `resources.limits.memory`; set requests close to steady-state
> 2. **Memory leak** → watch `kubectl top pod --watch`; continuously growing usage = leak; profile with pprof/memory-profiler
> 3. **Large ML model load at startup** → model size > limit; raise limit to accommodate model + overhead
> 4. **Requests << limits** (overcommit) → node is overcommitted; multiple OOMKills racing; fix requests
>
> **Long-term fix:** use VPA (Vertical Pod Autoscaler) in recommendation mode to right-size requests/limits based on historical usage.
>
> *See also:* [[k8s/scenarios/oomkilled]]

---

### Card B5
**Q: How do you drain a node safely without dropping requests? What is a PodDisruptionBudget and how does it interact with drain?**

> [!note]- Answer
> ```bash
> kubectl cordon <node>    # marks unschedulable; no new Pods land here
> kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
> ```
>
> `drain` evicts all Pods from the node. It respects **PodDisruptionBudgets**.
>
> **PodDisruptionBudget (PDB):**
> ```yaml
> spec:
>   minAvailable: 2        # at least 2 Pods must stay running during disruption
>   selector:
>     matchLabels:
>       app: model-server
> ```
>
> If draining the node would bring running replicas below `minAvailable`, `kubectl drain` **blocks** and waits. This prevents draining from causing an outage.
>
> **Drain fails / hangs:**
> - PDB is too strict (e.g., `minAvailable: 3` with only 3 replicas → can never drain)
> - Pod with no controller (standalone Pod) → use `--force` flag (Pod is deleted permanently)
> - Job Pod → use `--delete-emptydir-data`
>
> **After maintenance:**
> ```bash
> kubectl uncordon <node>  # allow new Pods to schedule here again
> ```

---

## NETWORKING

### Card C1
**Q: A Pod can't reach a Service. Walk through the complete debug ladder from DNS to NetworkPolicy.**

> [!note]- Answer
> ```
> Level 1: kubectl get endpoints <svc>
>   → Empty? → selector doesn't match Pod labels (most common cause)
>
> Level 2: kubectl get pods -l <selector> → Are they Ready?
>   → Not Ready? → readiness probe failing → fix app/probe
>
> Level 3: curl <pod-ip>:<container-port> directly from source pod
>   → Works? → Service/kube-proxy issue
>   → Fails? → CNI or app not listening
>
> Level 4: curl <clusterIP>:<service-port>
>   → Works? → DNS issue
>   → Fails? → check targetPort; check kube-proxy rules
>
> Level 5: nslookup <svc-name> from source pod
>   → NXDOMAIN? → CoreDNS issue; check kube-dns pods
>
> Level 6: kubectl get networkpolicy -n <ns>
>   → Any policies? → Check if source pod's label is in allowed ingress sources
> ```
>
> **#1 root cause by far:** empty endpoints from label mismatch. Check this first.
>
> *See also:* [[k8s/scenarios/service-unreachable]]

---

### Card C2
**Q: Explain the 4 Service types — ClusterIP, NodePort, LoadBalancer, and Headless. When would you choose each?**

> [!note]- Answer
> | Type | Reachable From | Use When |
> |---|---|---|
> | **ClusterIP** | Inside cluster only | Default; internal microservice communication |
> | **NodePort** | `<NodeIP>:<30000-32767>` | Dev/testing; direct node access needed; not for production |
> | **LoadBalancer** | External via cloud LB | Production external traffic; one LB per Service (expensive) |
> | **Headless** (`clusterIP: None`) | Inside cluster; DNS returns Pod IPs | StatefulSets; client-side load balancing; direct Pod addressing |
>
> **Ingress vs LoadBalancer:** LoadBalancer creates one cloud LB per Service. Ingress shares one LB across many Services via L7 routing by host/path — much more cost-efficient.
>
> **Headless deep-dive:** CoreDNS returns A records for each Pod IP. StatefulSets use this for stable Pod DNS: `pod-0.svc-headless.ns.svc.cluster.local`. Distributed databases (Cassandra, Kafka) use this for peer discovery.
>
> *See also:* [[k8s/concepts/service]], [[k8s/topics/networking]]

---

### Card C3
**Q: How does CoreDNS resolve a service name? What DNS name formats are valid, and what is the `ndots` setting?**

> [!note]- Answer
> Every Pod's `/etc/resolv.conf` has:
> ```
> nameserver 10.96.0.10   # CoreDNS ClusterIP
> search default.svc.cluster.local svc.cluster.local cluster.local
> ndots: 5
> ```
>
> **FQDN**: `<svc>.<namespace>.svc.cluster.local` → always works
> **Short name** (`my-svc`): works within same namespace; expanded by search path
> **Cross-namespace**: `my-svc.other-namespace` → resolves via search path expansion
>
> **ndots: 5 behavior:** if the name has fewer than 5 dots, K8s tries appending search domains first before treating it as absolute. `my-svc` (0 dots) → tries `my-svc.default.svc.cluster.local` first.
>
> **Debugging DNS:**
> ```bash
> kubectl exec -it <pod> -- nslookup my-svc
> kubectl exec -it <pod> -- cat /etc/resolv.conf
> kubectl logs -n kube-system -l k8s-app=kube-dns
> ```
>
> *See also:* [[k8s/topics/networking]]

---

### Card C4
**Q: What is a NetworkPolicy? Write one that allows only your frontend Pods to reach the backend API Pods on port 8080, and denies all other ingress.**

> [!note]- Answer
> NetworkPolicy applies firewall rules using label selectors. Requires a CNI plugin that supports it (Calico, Cilium — not basic Flannel).
>
> ```yaml
> # Step 1: Default deny all ingress in backend namespace
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: default-deny-ingress
>   namespace: backend
> spec:
>   podSelector: {}      # selects ALL pods
>   policyTypes: [Ingress]
> ---
> # Step 2: Allow only frontend pods
> kind: NetworkPolicy
> metadata:
>   name: allow-frontend
>   namespace: backend
> spec:
>   podSelector:
>     matchLabels:
>       app: api           # target: backend API pods
>   policyTypes: [Ingress]
>   ingress:
>   - from:
>     - podSelector:
>         matchLabels:
>           app: frontend  # source: frontend pods
>       namespaceSelector:
>         matchLabels:
>           kubernetes.io/metadata.name: frontend-ns
>     ports:
>     - port: 8080
> ```
>
> **Key gotcha:** A pod with ANY NetworkPolicy becomes restricted. Multiple policies are additive (OR). An empty `podSelector: {}` selects ALL pods in the namespace.

---

## SECURITY & RBAC

### Card D1
**Q: A Pod needs to call the Kubernetes API to list Deployments in its namespace. Walk through the complete RBAC setup with least privilege.**

> [!note]- Answer
> ```yaml
> # 1. Create a dedicated ServiceAccount (never use 'default')
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: deploy-reader-sa
>   namespace: ml-team
> ---
> # 2. Create a Role with ONLY what's needed
> apiVersion: rbac.authorization.k8s.io/v1
> kind: Role
> metadata:
>   name: deployment-reader
>   namespace: ml-team
> rules:
> - apiGroups: ["apps"]
>   resources: ["deployments"]
>   verbs: ["get", "list", "watch"]   # read-only; no create/delete
> ---
> # 3. Bind the Role to the ServiceAccount
> kind: RoleBinding
> subjects:
> - kind: ServiceAccount
>   name: deploy-reader-sa
>   namespace: ml-team
> roleRef:
>   kind: Role
>   name: deployment-reader
> ---
> # 4. Reference in Pod spec
> spec:
>   serviceAccountName: deploy-reader-sa
> ```
>
> **Verify:**
> ```bash
> kubectl auth can-i list deployments \
>   --as=system:serviceaccount:ml-team:deploy-reader-sa -n ml-team
> # yes
> kubectl auth can-i delete deployments \
>   --as=system:serviceaccount:ml-team:deploy-reader-sa -n ml-team
> # no
> ```
>
> *See also:* [[k8s/concepts/rbac]]

---

### Card D2
**Q: How do you prevent a Pod from running as root? What security context fields do you set and at what levels?**

> [!note]- Answer
> ```yaml
> spec:
>   securityContext:           # Pod level (applies to all containers)
>     runAsNonRoot: true       # K8s validates UID != 0 before starting
>     runAsUser: 1000
>     runAsGroup: 3000
>     fsGroup: 2000            # volume ownership for mounted volumes
>     seccompProfile:
>       type: RuntimeDefault   # enable default seccomp filter
>   containers:
>   - name: app
>     securityContext:         # Container level (overrides pod level)
>       allowPrivilegeEscalation: false   # no sudo/setuid
>       readOnlyRootFilesystem: true      # prevents writing to container FS
>       capabilities:
>         drop: ["ALL"]        # drop all Linux capabilities
>         add: ["NET_BIND_SERVICE"]  # add back only if needed (bind port <1024)
> ```
>
> **Pod Security Admission** (PSA) enforces these at namespace level:
> ```bash
> kubectl label namespace ml-team \
>   pod-security.kubernetes.io/enforce=restricted \
>   pod-security.kubernetes.io/warn=restricted
> ```
> `restricted` level requires all of the above.
>
> *See also:* [[k8s/topics/security]]

---

## SCHEDULING & RESOURCES

### Card E1
**Q: How do you ensure ML training jobs always land on GPU nodes and CPU-only workloads never consume GPU capacity?**

> [!note]- Answer
> **Three-layer pattern:**
>
> ```bash
> # 1. Taint all GPU nodes (repel non-tolerating pods)
> kubectl taint nodes gpu-node-1 dedicated=gpu:NoSchedule
> ```
>
> ```yaml
> # 2. In the ML training Pod: add toleration + nodeAffinity
> tolerations:
> - key: dedicated
>   operator: Equal
>   value: gpu
>   effect: NoSchedule        # allows landing on tainted nodes
> affinity:
>   nodeAffinity:
>     requiredDuringSchedulingIgnoredDuringExecution:
>       nodeSelectorTerms:
>       - matchExpressions:
>         - key: kubernetes.io/accelerator
>           operator: In
>           values: [nvidia-tesla-v100]  # forces onto GPU nodes
> resources:
>   limits:
>     nvidia.com/gpu: 2       # extended resource; requires device plugin
> ```
>
> **Why all three layers?**
> - Taint alone: CPU pods stay off GPU nodes ✓, but ML pods might land on CPU nodes
> - Toleration alone: ML pods CAN land on GPU nodes, but aren't FORCED to
> - nodeAffinity alone: forces ML pods to GPU nodes, but CPU pods can still land there
> - All three: CPU pods excluded by taint; ML pods forced to GPU nodes
>
> *See also:* [[k8s/scenarios/gpu-scheduling]]

---

### Card E2
**Q: What is the difference between resource requests and limits? What are the three QoS classes and when is a Pod evicted?**

> [!note]- Answer
> **requests**: what the scheduler uses to find a node. Guaranteed amount — kubelet reserves this on the node.
> **limits**: enforced ceiling. CPU is throttled; memory triggers OOMKill.
>
> ```yaml
> resources:
>   requests:
>     cpu: "500m"     # scheduler uses this for placement
>     memory: "512Mi" # node must have this free
>   limits:
>     cpu: "2"        # throttled, not killed, when exceeded
>     memory: "1Gi"   # OOMKilled immediately when exceeded
> ```
>
> **QoS Classes (assigned automatically):**
> | Class | Condition | Eviction Priority |
> |---|---|---|
> | **Guaranteed** | requests == limits for all containers | Last evicted |
> | **Burstable** | requests set but < limits | Evicted if node under pressure |
> | **BestEffort** | no requests or limits | Evicted first |
>
> **Node pressure eviction order:** BestEffort → Burstable (furthest over request) → Guaranteed
>
> **Rule of thumb for production:** Set requests = expected steady-state; limits = 2× requests. Use `Guaranteed` for critical serving Pods.

---

### Card E3
**Q: Explain taints and tolerations. What are the three effects? Give a real scenario for each.**

> [!note]- Answer
> **Taint**: on a node — "I repel Pods that don't explicitly accept me"
> **Toleration**: on a Pod — "I accept this specific taint"
>
> | Effect | Behavior | Real Scenario |
> |---|---|---|
> | `NoSchedule` | New pods without toleration won't be scheduled (existing pods untouched) | Reserve GPU nodes for ML workloads only |
> | `PreferNoSchedule` | Soft version — avoid if possible but ok if necessary | Mark nodes as "prefer not to use" but allow overflow |
> | `NoExecute` | New pods won't schedule AND existing pods without toleration are **evicted** | Node is unhealthy/draining; evict all non-critical workloads |
>
> ```bash
> # Add taint
> kubectl taint nodes node1 gpu=reserved:NoSchedule
>
> # Remove taint
> kubectl taint nodes node1 gpu=reserved:NoSchedule-
> ```
>
> **K8s adds taints automatically:**
> - `node.kubernetes.io/not-ready:NoExecute` → node is NotReady
> - `node.kubernetes.io/unreachable:NoExecute` → node unreachable
> - `node.kubernetes.io/disk-pressure:NoSchedule` → node disk pressure
>
> *See also:* [[k8s/topics/scheduling]]

---

## SCALING & AUTOSCALING

### Card F1
**Q: Your HPA is configured but not scaling. Walk through the 7 possible causes and how you check each.**

> [!note]- Answer
> Start: `kubectl describe hpa <name>` → Conditions section tells you exactly why.
>
> | Cause | Check | Fix |
> |---|---|---|
> | **Metrics Server down** | `kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods` → 404 | Install/restart metrics-server |
> | **No resource requests** | `kubectl describe pod` → Requests: 0 | Set `resources.requests.cpu` |
> | **Already at maxReplicas** | `kubectl get hpa` → REPLICAS = maxReplicas | Increase `maxReplicas` |
> | **Scale-down stabilization** | HPA condition: `ScaleDownStabilized` | Normal; 5min default window |
> | **New pods not Ready** | `kubectl get pods` → 0/1 Not Ready | Fix readiness probe / app |
> | **CPU throttling (not utilization)** | `container_cpu_cfs_throttled_seconds_total` | Raise CPU limits |
> | **Custom metrics not configured** | `kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1` → 404 | Deploy Prometheus adapter / KEDA |
>
> *See also:* [[k8s/scenarios/hpa-not-scaling]], [[k8s/concepts/hpa]]

---

### Card F2
**Q: Compare HPA, VPA, and KEDA. When would you use each for an ML workload?**

> [!note]- Answer
> | | HPA | VPA | KEDA |
> |---|---|---|---|
> | **Scales** | Replica count | CPU/memory per Pod | Replica count (to 0) |
> | **Based on** | CPU%, memory%, custom metrics | Historical resource usage | External events (queue, Kafka, cron) |
> | **Restarts pods?** | No | Yes (to apply new requests) | No |
> | **Scale to zero?** | No (min 1) | No | Yes |
> | **Use with HPA?** | N/A | Only in `Off`/`Initial` mode | Yes — KEDA extends HPA |
>
> **ML workload recommendations:**
> - **Model serving (HTTP traffic)**: HPA on CPU% or custom RPS metric
> - **Right-sizing training jobs**: VPA in recommendation mode — tells you what to set, you apply it
> - **Batch inference (queue-based)**: KEDA on SQS/Kafka queue depth — scales to 0 when queue empty, scales up proportionally to backlog
> - **Scheduled training**: KEDA CronScaler — scale to N replicas at 2am, back to 0 at 6am
>
> *See also:* [[k8s/concepts/hpa]]

---

## MLOPS ON KUBERNETES

### Card G1
**Q: Design a multi-tenant Kubernetes cluster for 5 ML teams. What isolation layers do you implement and what does each protect?**

> [!note]- Answer
> | Layer | Mechanism | Protects Against |
> |---|---|---|
> | **API isolation** | Namespaces + RBAC (Role + RoleBinding per team) | Team sees/modifies only their namespace |
> | **Compute fairness** | ResourceQuota per namespace (CPU, memory, GPU) | One team can't starve others |
> | **Workload safety** | LimitRange per namespace (default + max per container) | Single Pod can't consume entire quota |
> | **Network isolation** | NetworkPolicy: default-deny + allow-intra-team | Teams can't call each other's services |
> | **Node isolation** | Taints + Tolerations (optional, for compliance) | Physical compute separation |
>
> **ResourceQuota example:**
> ```yaml
> hard:
>   requests.nvidia.com/gpu: "8"
>   requests.cpu: "40"
>   requests.memory: 160Gi
> ```
>
> **LimitRange ensures:** even if a team has a 40-CPU quota, a single pod can't grab all 40 at once — `max.cpu: 16` caps each container.
>
> **Key interview point:** "Namespace alone does NOT provide isolation — it's just a naming scope. You need RBAC + ResourceQuota + NetworkPolicy for real multi-tenancy."
>
> *See also:* [[k8s/scenarios/multi-tenant-isolation]]

---

### Card G2
**Q: How do you do a zero-downtime deployment of a model server? List every mechanism required and explain the race condition that preStop hooks solve.**

> [!note]- Answer
> **Required mechanisms:**
>
> 1. **`RollingUpdate` with `maxUnavailable: 0`**: old Pods removed only after new ones are Ready
> 2. **readinessProbe**: new Pods only receive traffic after passing; prevents routing to an unloaded model
> 3. **`preStop` hook (sleep 5-10s)**: bridges the endpoint propagation race
> 4. **`terminationGracePeriodSeconds`**: must exceed preStop sleep + max in-flight request duration
> 5. **SIGTERM handler in app**: process gracefully finishes in-flight requests on SIGTERM
> 6. **PodDisruptionBudget**: protects against too-aggressive rollout under pressure
>
> **The race condition:**
> When a Pod is marked for termination:
> - K8s removes it from Service endpoints → kube-proxy updates iptables
> - But kube-proxy propagation takes a few seconds
> - During that window, NEW connections still route to the terminating Pod
> - The Pod gets SIGTERM and terminates → connection refused
>
> **preStop sleep** holds the Pod alive for 5-10s after it's removed from endpoints but before SIGTERM — by the time SIGTERM arrives, kube-proxy has removed all routes to this Pod.
>
> *See also:* [[k8s/scenarios/zero-downtime-deployment]]

---

### Card G3
**Q: A Kubeflow PyTorchJob (distributed training) is stuck. The master Pod is Running but workers are Pending. What do you check?**

> [!note]- Answer
> **Step 1: Describe the worker Pods**
> ```bash
> kubectl describe pod <pytorch-job-worker-0>
> # Events: "0/N nodes available: N Insufficient nvidia.com/gpu"
> ```
>
> **Common causes for worker Pods pending:**
>
> 1. **Insufficient GPU resources**: cluster doesn't have enough free GPUs
>    ```bash
>    kubectl describe nodes | grep -A 3 "nvidia.com/gpu"
>    # allocatable vs allocated
>    ```
>
> 2. **Taint mismatch**: workers need GPU nodes but don't have the toleration
>    ```bash
>    kubectl get pytorchjob <name> -o yaml | grep -A 5 tolerations
>    ```
>
> 3. **ResourceQuota exceeded for the namespace**
>    ```bash
>    kubectl describe resourcequota -n <namespace>
>    # Check GPU quota remaining
>    ```
>
> 4. **Master is using wrong ServiceAccount** (can't talk back to API server to report status)
>    ```bash
>    kubectl logs <master-pod>
>    # "403 Forbidden" from API server
>    ```
>
> 5. **Training operator not running**
>    ```bash
>    kubectl get pods -n kubeflow | grep training-operator
>    ```
>
> **Key insight:** The master runs first (index 0); workers follow. A stuck worker = scheduling issue, not a training code issue. Fix scheduling first.

---

### Card G4
**Q: How do you rotate a database password used by model-serving Pods without restarting any Pod or dropping requests?**

> [!note]- Answer
> **Only possible if the Secret is mounted as a volume file (not an env var).**
>
> Env vars are snapshot at container start — they NEVER update without a restart.
>
> **Zero-downtime rotation procedure:**
>
> 1. **Ensure secret is mounted as a file:**
> ```yaml
> volumeMounts:
> - name: db-creds
>   mountPath: /etc/secrets
>   readOnly: true
> volumes:
> - name: db-creds
>   secret:
>     secretName: db-password
> ```
>
> 2. **Enable dual credentials in the database** (accept both old and new password)
>
> 3. **Update the K8s Secret:**
> ```bash
> kubectl create secret generic db-password \
>   --from-literal=password=NEW_PASS \
>   --dry-run=client -o yaml | kubectl apply -f -
> ```
>
> 4. **Wait ~60 seconds** for kubelet to propagate the new file to all Pods
>
> 5. **Application must re-read the file dynamically** — implement a file watcher or re-open the file on each connection establishment; do NOT cache at startup
>
> 6. **Revoke old password** from database once confirmed all Pods use new one
>
> **Best practice (ESO):** store in Vault/AWS Secrets Manager; External Secrets Operator syncs automatically — no manual kubectl needed.
>
> *See also:* [[k8s/scenarios/secret-rotation]]

---

### Card G5
**Q: Explain the Operator pattern. What is a CRD, what is a controller loop, and how does the Kubeflow Training Operator manage a PyTorchJob?**

> [!note]- Answer
> **CRD (Custom Resource Definition)**: extends the K8s API with a new resource type. After applying a CRD, you can `kubectl apply -f pytorchjob.yaml` and the API server accepts and stores it.
>
> **Controller loop** (reconciliation):
> ```
> while True:
>   desired = list(PyTorchJobs)
>   actual  = list(Jobs, Pods)
>   if desired != actual:
>     reconcile()   # create/delete/update K8s objects to match desired state
> ```
>
> **How Training Operator manages a PyTorchJob:**
> 1. User applies a `PyTorchJob` CR (master: 1 replica, workers: 3 replicas)
> 2. Controller creates 1 master Job + 3 worker Jobs → each creates a Pod
> 3. Controller injects env vars: `MASTER_ADDR=<master-pod-ip>`, `MASTER_PORT=23456`, `RANK`, `WORLD_SIZE`
> 4. PyTorch distributed training uses these for rendezvous (worker discovery)
> 5. Controller watches all Pods; when all succeed → marks PyTorchJob `Succeeded`
> 6. If a worker fails → retry per `backoffLimit`; if over limit → mark `Failed`
>
> **Why Operators instead of raw Jobs?** Operators encode domain logic — retry strategies, status reporting, checkpoint management, multi-Pod coordination — that plain K8s objects can't express.
>
> *See also:* [[k8s/concepts/operator-crd]]
