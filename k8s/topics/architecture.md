# Kubernetes Architecture

**Topic:** [[k8s/topics/architecture]]
**Related:** [[k8s/topics/workloads]], [[k8s/topics/networking]]

## Control Plane Components

### API Server (`kube-apiserver`)
The front-end for the Kubernetes control plane. All communication goes through it — `kubectl`, controllers, kubelets, and external tools all talk to the API server.

- Validates and authenticates requests (via webhooks, certificates, tokens)
- Persists state to etcd
- Serves as the watch endpoint for all controllers
- **Stateless** — can be replicated for HA; clients automatically retry
- If it goes down: nothing can be created or updated; existing workloads continue running

### etcd
Distributed, consistent key-value store. The only component with persistent state.

- All cluster state lives here: Pod specs, Service definitions, Secrets, ConfigMaps
- Uses the Raft consensus algorithm; requires a quorum (majority) for writes
- Losing etcd quorum → cluster is effectively read-only
- Backups are critical; etcd is the only component that needs them

### Scheduler (`kube-scheduler`)
Watches for unscheduled Pods (Pods with no `nodeName`). For each, runs a two-phase decision:
1. **Filter**: eliminate nodes that don't satisfy constraints (resources, taints, affinity)
2. **Score**: rank remaining nodes by preference (spread, resource balance, affinity weights)

Assigns the best node by writing `nodeName` to the Pod spec. The kubelet on that node then takes over.

### Controller Manager (`kube-controller-manager`)
Runs all built-in control loops in a single process:
- **ReplicaSet controller**: ensures N copies of a Pod template are running
- **Deployment controller**: manages rolling updates via ReplicaSets
- **Node controller**: marks nodes NotReady after missed heartbeats
- **Job controller**: manages batch workload completion
- Each controller watches its resources via the API server and reconciles

### Cloud Controller Manager
Bridges Kubernetes to cloud provider APIs. Handles:
- Node lifecycle (provision/deprovision cloud VMs)
- LoadBalancer Service provisioning
- Volume provisioning and attachment

## High Availability — How a Multi-Master Control Plane Elects Its Leaders

A multi-master (multiple control-plane nodes) cluster has **three separate election/replication mechanisms** running at once — a common interview trap is treating them as one:

### 1. etcd's own leader — Raft
The etcd cluster backing the control plane (typically 3 or 5 members) runs its own Raft consensus internally to elect an etcd leader, which handles all writes and replicates them to followers. Standard Raft: randomized election timeouts, terms, majority vote. See [[solution-arch/concepts/raft]] for the full protocol (RequestVote/AppendEntries RPCs, the log matching property, why an out-of-date node can't win).

### 2. kube-scheduler and kube-controller-manager — lease-based election on etcd
These must have exactly one *active* instance even when multiple replicas run for HA (each master typically runs its own copy). They do **not** use Raft directly — they use the `client-go` `leaderelection` package:
```
Each replica started with --leader-elect=true
All replicas race to acquire a Lease object (coordination.k8s.io/v1)
  in kube-system, named e.g. "kube-scheduler" / "kube-controller-manager"

Lease fields: holderIdentity, leaseDurationSeconds (default 15s), renewTime

Winner renews before expiry (renewDeadline default 10s, retryPeriod 2s)
On crash/partition/GC pause → renewal missed → lease goes stale →
  standbys race again via a conditional write (etcd resourceVersion CAS
  guarantees only one candidate's update succeeds)
```
This is the general lease-based leader-election pattern — see [[solution-arch/concepts/leader-election]] for the ZooKeeper-equivalent (ephemeral sequential znodes) and for fencing.

### 3. kube-apiserver — no election at all
All API server replicas run **active-active**: stateless request handlers fronted by a load balancer, all reading/writing the same etcd cluster. There's no leader among API servers because there's nothing stateful to arbitrate.

**Interview-ready summary:** etcd elects its own leader via Raft; scheduler/controller-manager elect a leader via an etcd-backed Lease (compare-and-swap, not Raft directly); API servers don't elect anything.

## Worker Node Components

### kubelet
An agent on every node. Responsibilities:
- Watches the API server for Pods assigned to its node
- Calls the container runtime to create/start/stop containers
- Reports node and Pod status back to the API server
- Runs liveness/readiness/startup probes
- Manages volume mounts

### kube-proxy
Maintains network rules (iptables or ipvs) on each node to implement Service routing.
- When a Service's ClusterIP is hit, kube-proxy rules DNAT the packet to one of the backing Pods
- Uses EndpointSlices (backed by Endpoints) to know which Pods to route to

### Container Runtime
The actual process that runs containers. Must implement CRI (Container Runtime Interface).
- **containerd** (most common), **CRI-O**, ~~Docker~~ (deprecated in K8s 1.24+)

## The Reconciliation Loop Pattern

Every controller follows this pattern:
```
while True:
    desired = watch(API server for resource spec changes)
    actual  = observe(current state of the world)
    if desired != actual:
        reconcile(take actions to make actual = desired)
```

Understanding this loop is the key to reasoning about any K8s behavior. When something isn't working, ask: *which controller is responsible, what is its desired state, and what is blocking reconciliation?*

## Sources
- [[K8S overview]]
