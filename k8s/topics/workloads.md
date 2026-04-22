# Kubernetes Workloads

**Topic:** [[k8s/topics/workloads]]
**Related:** [[k8s/topics/scheduling]], [[k8s/topics/observability]]

## Pod

The smallest deployable unit. One or more containers sharing a network namespace (same IP, localhost communication) and optional shared volumes. Pods are ephemeral — never restart in place; they are replaced.

**See:** [[k8s/concepts/pod]]

---

## Deployment

Manages a ReplicaSet, which manages a set of identical Pods. Handles rolling updates, rollbacks, and scaling.

- Use for: stateless services, model serving, API endpoints
- `maxSurge`: how many extra Pods can exist during a rolling update
- `maxUnavailable`: how many Pods can be unavailable during a rolling update
- Rollback: `kubectl rollout undo deployment/<name>`

**See:** [[k8s/concepts/deployment]]

---

## StatefulSet

Like a Deployment but for stateful workloads. Pods get:
- **Stable, predictable names**: `<name>-0`, `<name>-1`, ...
- **Stable network identity**: DNS names that persist across restarts
- **Ordered deployment and scaling**: Pod `n` starts only after Pod `n-1` is Running
- **Per-Pod PVCs**: each Pod gets its own PersistentVolumeClaim via `volumeClaimTemplates`

Use for: databases, distributed training with checkpointing, Kafka, ZooKeeper.

**See:** [[k8s/concepts/statefulset]]

---

## DaemonSet

Ensures exactly one Pod runs on every node (or on nodes matching a selector). New nodes automatically get a copy.

Use for: log collectors (Fluentd, Filebeat), monitoring agents (node-exporter), network plugins, GPU device plugins.

---

## Job

Runs Pods to completion. Guarantees at least one successful completion.

- `completions`: total number of Pods that must succeed
- `parallelism`: how many Pods run simultaneously
- `backoffLimit`: how many times to retry before marking the Job as failed
- **Indexed Jobs** (K8s 1.21+): each Pod gets a unique `JOB_COMPLETION_INDEX` env var — ideal for distributing work shards across training workers

Use for: ML training runs, batch inference, data preprocessing, database migrations.

---

## CronJob

Schedules Jobs on a cron expression. Creates a new Job object for each scheduled run.

- `concurrencyPolicy`: `Allow` / `Forbid` / `Replace` — controls what happens if a previous run is still active
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`: how many past Jobs to keep
- Common issue: `startingDeadlineSeconds` — if the CronJob controller misses a scheduled time by more than this, it skips that run

Use for: periodic model retraining triggers, scheduled batch scoring, data pipeline jobs.

---

## HorizontalPodAutoscaler (HPA)

Automatically scales a Deployment (or StatefulSet) based on observed metrics.

**See:** [[k8s/concepts/hpa]]

---

## Choosing the Right Workload Type

| Workload | Use When |
|---|---|
| Deployment | Stateless service, N identical replicas, rolling update needed |
| StatefulSet | Each Pod needs stable identity or its own storage |
| DaemonSet | Need exactly one Pod per node (monitoring, log shipping) |
| Job | One-time task that runs to completion (training, migration) |
| CronJob | Recurring Job on a schedule |

## Sources
- [[K8S overview]]
