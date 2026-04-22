# Clean Architecture

**Type:** book
**Author:** Robert C. Martin (Uncle Bob)
**Ingested:** 2026-04-22
**Topics covered:** [[solution-arch/topics/architectural-styles]], [[solution-arch/topics/cloud-agnostic-principles]]

## Summary

Presents the architectural principles that make software systems maintainable and testable over time. Builds on SOLID principles to define the "dependency rule": source code dependencies must point inward — outer layers (frameworks, UI, DB) depend on inner layers (use cases, entities); never the reverse.

The Hexagonal (Ports and Adapters) architecture described here is the cloud-agnostic ideal: the domain core has zero dependencies on infrastructure. Frameworks, databases, and messaging are plugins.

## Key Takeaways

- The goal of architecture is to minimise the cost of change, not to make the first version work
- Dependency rule: dependencies point inward toward policy (business rules); never outward toward details (DB, framework)
- Use cases are first-class citizens — visible in folder structure, not buried in controllers
- Defer infrastructure decisions as long as possible — they should be easy to change
- Testability and deployability are architectural properties, not afterthoughts
- Hexagonal architecture: swap DB without touching business logic; test without a running server

## What it updated
- Informed: [[solution-arch/topics/architectural-styles]] (hexagonal section), [[solution-arch/topics/cloud-agnostic-principles]], [[solution-arch/patterns/bff]], [[solution-arch/patterns/strangler-fig]], [[solution-arch/diagrams/c4-model]]
