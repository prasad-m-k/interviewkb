# MLOps vs DevOps

**Topic:** [[topics/mlops]]
**Related:** [[concepts/ml-pipeline]], [[concepts/experiment-tracking]]

## Summary
MLOps and DevOps share core principles (automation, CI/CD, collaboration, feedback loops) but differ significantly because ML systems have unique properties: they are data-dependent, iterative, and can degrade silently without any code change.

## Key differences

| Dimension | DevOps | MLOps |
|---|---|---|
| **Team composition** | Mostly DevOps/SRE engineers | Data scientists + ML engineers + data engineers + DevOps |
| **Development process** | Relatively linear: code → test → deploy | Iterative and experimental: explore data, try features, tune models |
| **What gets versioned** | Code | Code + data + models + experiments |
| **CI/CD** | CI + CD | CI + CD + **CT** (Continuous Training) |
| **"It broke" signals** | Errors, crashes, latency | Silent accuracy degradation, drift — no exception thrown |
| **Artifacts** | Binaries, containers | Model weights, feature pipelines, training datasets |
| **Feedback loops** | Test results, logs | Model performance metrics, data distribution stats |

## Why Continuous Training is the key addition
In DevOps, once software is deployed it doesn't degrade on its own (absent bugs). In ML, a model trained on last year's data may degrade as the world changes — without a single line of code being touched. This makes **Continuous Training (CT)** a necessary third pillar alongside CI and CD.

```
DevOps:  CI → CD
MLOps:   CI → CD → CT (retrain on new data, redeploy)
```

## Collaboration model
DevOps solved the wall between Dev and Ops. MLOps has to solve a bigger wall: between **data scientists** (who build models in notebooks, care about accuracy) and **engineers** (who care about reliability, latency, and maintainability). The operational failure mode without MLOps is: data scientist hands a pickle file to an engineer and says "put this in production."

## Common interview angles
- "How is MLOps different from DevOps?" — this page
- "Why can't you just apply DevOps practices to ML?" — data + model versioning, CT, silent degradation, experiment reproducibility are not covered by standard DevOps
- "What does a mature MLOps setup look like?" — connect to MLOps maturity levels in [[topics/mlops]]

## Sources
- User-provided content (2026-04-21)
