# Pod

**Topic:** [[k8s/topics/workloads]]
**Related:** [[k8s/concepts/deployment]], [[k8s/topics/observability]]

## What it is
The smallest deployable unit in Kubernetes. One or more containers sharing a network namespace (same IP address, communicate via localhost) and optionally sharing volumes. Pods are ephemeral — they are never updated in place; they are replaced.

## Key Fields

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: model-server
    image: my-registry/model-server:v1.2
    ports:
    - containerPort: 8080
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "2"
        memory: "2Gi"
    env:
    - name: MODEL_PATH
      value: /models/v1
    envFrom:
    - configMapRef:
        name: model-config
    - secretRef:
        name: model-credentials
    volumeMounts:
    - name: model-storage
      mountPath: /models
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
  volumes:
  - name: model-storage
    persistentVolumeClaim:
      claimName: model-pvc
  serviceAccountName: model-server-sa
  restartPolicy: Always    # Always | OnFailure | Never
```

## Pod Lifecycle

```
Pending → Running → Succeeded
                 ↘ Failed
```

- **Pending**: Pod accepted; waiting for scheduling or image pull
- **Running**: At least one container is running or starting
- **Succeeded**: All containers exited with code 0 (Jobs)
- **Failed**: At least one container exited non-zero and won't restart

## Container States

Within a running Pod, each container can be:
- **Waiting**: `ContainerCreating`, `ImagePullBackOff`, `CrashLoopBackOff`
- **Running**: started and executing
- **Terminated**: exited (with exit code and reason)

## Init Containers

Run sequentially before app containers. Must all succeed before any app container starts. Use cases:
- Wait for a dependency to be ready (`initContainer: wait-for-db`)
- Download/prepare data before the main container starts
- Run database migrations before the app starts

```yaml
initContainers:
- name: wait-for-db
  image: busybox
  command: ['sh', '-c', 'until nc -z postgres-svc 5432; do sleep 2; done']
```

## Common Interview Angles
- "Your Pod is stuck in `Init:0/1`" → init container is failing; check its logs
- "Multiple containers in a Pod — when is that right?" → sidecar pattern (log shipper, proxy), or tightly coupled processes sharing a socket/local file
- "What happens to a Pod when its node dies?" → Pod is evicted; if owned by a ReplicaSet, a new Pod is created on another node

## Sources
- [[K8S overview]]
