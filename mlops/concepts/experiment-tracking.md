# Experiment Tracking

**Topic:** [[topics/mlops]]
**Related:** [[concepts/model-registry]], [[concepts/ml-pipeline]]

## What it is
Experiment tracking is the practice of logging every training run with its parameters, metrics, artifacts, and environment so runs can be compared, reproduced, and promoted. Without it, the best model is wherever the data scientist last saved a pickle file.

## What to log per run
- **Parameters**: hyperparameters, model architecture choices, preprocessing flags
- **Metrics**: train/val loss, accuracy, AUC, per-epoch curves
- **Artifacts**: model checkpoint, feature importance plots, confusion matrix
- **Environment**: library versions, git commit SHA, data version/hash
- **Tags**: free-form labels (experiment name, team, dataset variant)

## How it works (MLflow example)
```python
with mlflow.start_run():
    mlflow.log_params({"lr": 0.01, "n_estimators": 100})
    mlflow.log_metric("val_auc", 0.91)
    mlflow.sklearn.log_model(model, "model")
```

The tracking server stores runs; the UI lets you compare runs side-by-side, filter by metric, and promote the best run to the [[concepts/model-registry]].

## Reproducibility checklist
- [ ] Code: git commit SHA logged
- [ ] Data: dataset version or hash logged
- [ ] Environment: `requirements.txt` or Docker image logged
- [ ] Random seeds: set and logged
- [ ] Parameters: all hyperparameters logged (not hardcoded)

## Common interview angles
- "How do you ensure reproducibility of training runs?" — above checklist
- "How does experiment tracking connect to the model registry?" — best run from tracking is registered; registry manages its deployment lifecycle
- "What's the difference between experiment tracking and a model registry?" — tracking is about comparing and debugging runs; registry is about managing the lifecycle of models going to production

## Tools
MLflow (open source, self-hosted), Weights & Biases, Neptune.ai, Comet ML, ClearML, SageMaker Experiments

## Sources
*(none yet)*
