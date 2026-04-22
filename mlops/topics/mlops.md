# MLOps

**Related concepts:** [[concepts/ml-pipeline]], [[concepts/feature-store]], [[concepts/model-registry]], [[concepts/data-drift]], [[concepts/training-serving-skew]], [[concepts/experiment-tracking]]
**Related patterns:** [[patterns/ml-training-pipeline]], [[patterns/model-serving]]

## What is MLOps
MLOps (Machine Learning Operations) is the set of practices for reliably deploying, monitoring, and maintaining ML models in production. It bridges the gap between data science (building models) and operations (running systems at scale). Think of it as DevOps + data engineering + model lifecycle management.

## Why it matters in interviews
ML engineering and MLOps roles now expect candidates to understand the full lifecycle — not just training, but how a model gets from a notebook to production and stays healthy there. System design interviews at ML-heavy companies (Google, Meta, Airbnb, Uber, Netflix) often ask you to design an ML platform component.

## Core pillars

### 1. Data Management
- Data versioning (DVC, Delta Lake, LakeFS)
- Feature engineering and [[concepts/feature-store|feature stores]]
- Data validation (Great Expectations, TFX Data Validation)
- Preventing [[concepts/training-serving-skew]]

### 2. Experimentation
- [[concepts/experiment-tracking]] (MLflow, W&B, Neptune)
- Reproducibility: pinning data versions, code, environment, hyperparameters
- Hyperparameter tuning at scale (Optuna, Ray Tune)

### 3. Training Pipelines
- Orchestrated, reproducible training runs
- [[patterns/ml-training-pipeline|ML training pipeline pattern]]
- Tools: Kubeflow Pipelines, Metaflow, Airflow, Vertex AI Pipelines, SageMaker Pipelines

### 4. Model Registry & Versioning
- [[concepts/model-registry]]: central store of trained models with metadata and lifecycle state
- Promotion flow: Staging → Validation → Production
- Tools: MLflow Model Registry, SageMaker Model Registry, Vertex AI Model Registry

### 5. Model Serving
- [[patterns/model-serving|Serving patterns]]: online (REST/gRPC), batch, streaming
- Deployment strategies: blue/green, canary, shadow mode
- Tools: TorchServe, TF Serving, Triton Inference Server, BentoML, Seldon, Ray Serve, KServe

### 6. Monitoring & Observability
- [[concepts/data-drift]]: input distribution shifts over time
- Concept drift: the relationship between inputs and targets changes
- Model performance monitoring: track metrics, trigger retraining alerts
- Tools: Evidently AI, WhyLogs, Arize, Fiddler

### 7. CI/CD for ML
- CT (Continuous Training): automated retraining pipelines triggered by data or drift
- CD (Continuous Delivery): automated model validation and promotion
- Testing: unit tests for transforms, integration tests for pipelines, shadow mode validation

## MLOps Maturity Levels
A common interview framework (from Google's MLOps whitepaper):

| Level | Description |
|---|---|
| 0 | Manual: data scientists run notebooks, push models manually |
| 1 | ML pipeline automation: training pipeline is code, CT enabled |
| 2 | CI/CD pipeline automation: automated build, test, deploy of the ML pipeline itself |

Most companies are at Level 0 or 1. Interviews often ask "how would you move from Level 0 to Level 1?"

## Common interview questions
- Design a feature store for a ride-sharing recommendation system
- How do you detect and respond to model degradation in production?
- What is training-serving skew and how do you prevent it?
- Walk me through how you'd deploy a model with zero downtime
- How would you build a retraining pipeline that triggers automatically?
- What's the difference between data drift and concept drift?
- How do you version ML models and roll back safely?

## Key tools landscape

| Category | Tools |
|---|---|
| Orchestration | Airflow, Prefect, Metaflow, Kubeflow, Vertex AI Pipelines |
| Experiment tracking | MLflow, Weights & Biases, Neptune |
| Feature store | Feast, Tecton, Vertex AI Feature Store, Databricks |
| Model registry | MLflow, SageMaker, Vertex AI |
| Serving | TF Serving, TorchServe, Triton, BentoML, Ray Serve, KServe |
| Monitoring | Evidently, WhyLogs, Arize, Fiddler |
| Data versioning | DVC, Delta Lake, LakeFS |

## Sources
*(none yet — add sources as you ingest them)*
