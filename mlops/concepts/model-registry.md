# Model Registry

**Topic:** [[topics/mlops]]
**Related:** [[concepts/experiment-tracking]], [[concepts/ml-pipeline]], [[concepts/data-drift]]

## What it is
A model registry is a centralized store that manages trained model artifacts along with metadata, versioning, and lifecycle state. It's the handoff point between the training pipeline and the deployment pipeline.

## What it stores per model version
- Model artifact (weights, serialized object)
- Metrics from validation (accuracy, AUC, latency, etc.)
- Lineage: which dataset version and code commit produced this model
- Parameters / hyperparameters
- Lifecycle stage: `Staging` → `Production` → `Archived`

## Lifecycle flow
```
Training pipeline
      ↓ registers model + metrics
Model Registry
      ↓ CI/CD promotes after validation passes
Serving infrastructure
```

Promotion is typically gated: a validation step (offline evaluation, A/B test result, load test) must pass before a model moves from Staging to Production.

## Why it matters
- **Auditability**: know exactly which model is in production and what data/code produced it
- **Safe rollback**: if a deployed model degrades, promote the previous Production version in one step
- **Decoupling**: training team pushes to registry; serving team pulls from it — no hand-off toil

## Common interview angles
- "How do you safely deploy a new model?" — register → validate in staging → canary or shadow deploy → promote to production
- "How do you roll back a bad model?" — registry holds previous versions; re-promote the last good one
- "What metadata would you track in a model registry?" — artifact, metrics, dataset version, git SHA, parameters, who approved

## Tools
MLflow Model Registry, SageMaker Model Registry, Vertex AI Model Registry, Weights & Biases (Artifacts), DVC (model versioning)

## Sources
*(none yet)*
