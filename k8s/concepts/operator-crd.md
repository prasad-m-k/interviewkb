# Operator Pattern and CRDs

**Topic:** [[k8s/topics/architecture]]
**Related:** [[k8s/topics/workloads]]

## What it is
An **Operator** is an application-specific controller that extends Kubernetes to manage complex stateful applications using domain knowledge encoded as automation. It uses **Custom Resource Definitions (CRDs)** to define new API types, and a **controller loop** to reconcile them.

## Custom Resource Definition (CRD)

Extends the K8s API with new resource types. Once created, you can `kubectl get`, `kubectl apply`, and watch your custom resources just like built-in ones.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: trainingjobs.mlops.mycompany.com
spec:
  group: mlops.mycompany.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              modelImage: {type: string}
              gpuCount:   {type: integer}
              datasetPath: {type: string}
  scope: Namespaced
  names:
    plural: trainingjobs
    singular: trainingjob
    kind: TrainingJob
```

Now you can create:
```yaml
apiVersion: mlops.mycompany.com/v1
kind: TrainingJob
metadata:
  name: resnet-training-run-42
spec:
  modelImage: my-registry/resnet:latest
  gpuCount: 4
  datasetPath: s3://my-bucket/imagenet
```

## The Controller Loop

The operator watches for `TrainingJob` objects and reconciles:

```python
while True:
    desired = list_training_jobs()          # what users declared
    actual  = list_pods_and_jobs()          # what's running
    for job in desired:
        if not running(job):
            create_k8s_job(job)             # provision the workload
        elif completed(job):
            update_status(job, "Succeeded") # update status subresource
        elif failed(job):
            retry_or_fail(job)              # handle failure policy
```

## Well-Known ML Operators

| Operator | What it manages |
|---|---|
| **Kubeflow Training Operator** | PyTorchJob, TFJob, MXJob — distributed ML training |
| **KServe** | InferenceService — model serving with canary, autoscaling |
| **Argo Workflows** | Workflow (DAG of steps) — ML pipelines |
| **Spark Operator** | SparkApplication — Apache Spark on K8s |
| **KEDA** | ScaledObject — event-driven autoscaling |

## Why Use an Operator Instead of Raw K8s Objects?

| Raw K8s | Operator |
|---|---|
| You manage all lifecycle steps manually | Operator encodes domain-specific lifecycle (init, scaling, backup, upgrade) |
| No built-in failure handling | Operator implements retry, backoff, alerting logic |
| Config changes require manual coordination | Operator reconciles config changes automatically |
| No custom status fields | Operator exposes meaningful status (`trainingLoss`, `epoch`, `checkpointPath`) |

## Common Interview Angles
- "How does Kubeflow Training Operator manage a PyTorchJob?" → creates a Job per worker; configures env vars for rendezvous (master address); watches all Jobs; marks PyTorchJob succeeded when all workers complete
- "What is a finalizer?" → a field on a resource that prevents deletion until the controller removes it; used for cleanup (e.g., delete cloud resources before the K8s object is removed)
- "How do you debug a stuck CRD object?" → check operator Pod logs; check the object's status subresource and events via `kubectl describe`

## Sources
- [[K8S overview]]
