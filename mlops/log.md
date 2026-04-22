# Wiki Log

Append-only. Each entry: `## [YYYY-MM-DD] <type> | <title>`
Types: `ingest`, `query`, `lint`, `update`

Grep tip: `grep "^## \[" mlops/log.md | tail -10`

---

## [2026-04-21] query | Top 10 MLOps FAANG interview questions
- Filed answer as: flashcards/mlops-faang-top10
- Cards cover: training-serving skew, drift types, feature store design, zero-downtime deployment, CT vs CI/CD, maturity levels, production monitoring, reproducibility, model versioning/rollback, end-to-end recommendation system design

## [2026-04-21] ingest | MLOps topic — built from training knowledge + user content
- Created: topics/mlops
- Created concepts: feature-store, training-serving-skew, data-drift, model-registry, experiment-tracking, ml-pipeline, mlops-vs-devops
- Created patterns: ml-training-pipeline
- Updated: mlops/index.md, mlops/overview.md (coverage map)
- User provided content on "MLOps vs DevOps" — filed as concepts/mlops-vs-devops

## [2026-04-21] update | Wiki initialized
- Created directory structure: mlops/, sources/, topics/, concepts/, patterns/, problems/, companies/, sources/
- Created: mlops/index.md, mlops/log.md, mlops/overview.md
- Schema written to CLAUDE.md
- Domain: technical interview preparation
