---
tags:
  - index
  - mlops
  - ml-engineering
  - interview-prep
  - devops
---

# Wiki Index
Last updated: 2026-04-21

## Overview
- [[mlops/overview]] — High-level synthesis of the full knowledge base

## Topics
- [[mlops/topics/mlops]] — ML lifecycle, pipelines, feature stores, model serving, monitoring, CI/CD for ML

## Concepts
- [[mlops/concepts/feature-store]] — Centralized feature management; offline store for training, online store for serving
- [[mlops/concepts/training-serving-skew]] — When training features differ from serving features; causes silent production degradation
- [[mlops/concepts/data-drift]] — Input distribution shifts in production; covariate shift vs concept drift vs label drift
- [[mlops/concepts/model-registry]] — Central store for model artifacts with versioning and lifecycle state (Staging → Production)
- [[mlops/concepts/experiment-tracking]] — Logging parameters, metrics, and artifacts per training run for comparison and reproducibility
- [[mlops/concepts/ml-pipeline]] — Automated, reproducible data-to-model workflow replacing notebook runs
- [[mlops/concepts/mlops-vs-devops]] — Key differences: team composition, iterative process, CT, silent degradation, versioning scope

## Patterns
- [[mlops/patterns/ml-training-pipeline]] — End-to-end pattern: ingest → validate → features → train → evaluate → register

## Flashcards
- [[mlops/flashcards/mlops-faang-top10]] — Top 10 MLOps interview Qs for FAANG; covers drift, feature stores, CT, deployment, monitoring, reproducibility

## Problems
*(none yet)*

## Companies
*(none yet)*

## Sources
*(none yet)*
