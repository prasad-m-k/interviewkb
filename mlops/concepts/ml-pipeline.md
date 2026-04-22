# ML Pipeline

**Topic:** [[topics/mlops]]
**Related:** [[concepts/feature-store]], [[concepts/experiment-tracking]], [[concepts/model-registry]], [[patterns/ml-training-pipeline]]

## What it is
An ML pipeline is an automated, reproducible sequence of steps that takes raw data and produces a trained, validated model artifact. It replaces ad-hoc notebook runs with an orchestrated, testable, version-controlled workflow.

## Typical steps
```
Data ingestion → Data validation → Feature engineering → Training → Evaluation → Model registration
```

Each step is a discrete, rerunnable unit. Steps pass artifacts (datasets, feature tables, model files) to each other.

## Why pipelines over notebooks
| Notebooks | Pipelines |
|---|---|
| Hard to run repeatedly | Scheduled / triggered runs |
| Global state, hidden order | Explicit data flow between steps |
| No versioning of runs | Every run logged and reproducible |
| Manual handoff to production | Automated promotion via model registry |

## Key properties of a good pipeline
- **Idempotent**: re-running the same input produces the same output
- **Modular**: each step can be tested and rerun independently
- **Parameterized**: hyperparameters and config injected at runtime, not hardcoded
- **Versioned**: code + data + environment are all pinned

## Continuous Training (CT)
A CT pipeline adds automation on top of a standard training pipeline:
- **Trigger**: new data arrives, drift is detected, or a schedule fires
- **Run**: pipeline executes end-to-end automatically
- **Gate**: automated evaluation checks if new model beats baseline
- **Promote**: if gate passes, new model is promoted in the [[concepts/model-registry]]

## Common interview angles
- "Walk me through an ML training pipeline you would design" — cover all steps, orchestration tool, artifact passing, model registration
- "What is continuous training?" — automated retraining triggered by data, drift, or schedule
- "How do you test an ML pipeline?" — unit test transforms, integration test the pipeline end-to-end on a small data slice, validate output schema

## Tools
Kubeflow Pipelines, Vertex AI Pipelines, SageMaker Pipelines, Metaflow, Airflow + custom operators, Prefect, ZenML

## Sources
*(none yet)*
