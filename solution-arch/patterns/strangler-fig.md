# Strangler Fig Pattern

**Topic:** [[solution-arch/topics/architectural-styles]]
**Related concepts:** [[solution-arch/concepts/api-gateway]], [[solution-arch/patterns/sidecar-ambassador]]

## What it solves
Migrating a legacy monolith to microservices incrementally, without a "big bang" rewrite. Named after the strangler fig tree that grows around a host tree and eventually replaces it. The old system coexists with the new until the new fully replaces it.

## Template / Skeleton

```
Phase 1: Facade (intercept traffic)
  All traffic ──▶ Strangler Facade (proxy/API Gateway)
                        │
                        ▼
                  Legacy Monolith (100% of traffic)

Phase 2: Extract first service
  All traffic ──▶ Strangler Facade
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
       New User Service      Legacy Monolith
       (/api/users → new)    (everything else)

Phase 3: Extract more services
  All traffic ──▶ Strangler Facade
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   User Service   Order Service   Legacy Monolith
                                  (shrinking)

Phase N: Complete migration
  All traffic ──▶ API Gateway ──▶ Microservices
                                  Legacy decommissioned
```

## Migration Sequence

```
For each domain to migrate:
  1. Identify bounded context (e.g., User Management)
  2. Build new service alongside monolith
  3. Set up facade to route /api/users to new service
  4. Shadow traffic: send to both, compare responses (canary validation)
  5. Switch 100% traffic to new service
  6. Remove code from monolith
  7. Repeat for next domain
```

## Data Migration Strategy

The hardest part: both monolith and new service may need the same data during transition.

```
Option A: Shared DB (temporary)
  New service ──▶ Monolith DB   (safe, but couples new service to old schema)

Option B: Sync via events
  Monolith ──writes──▶ Monolith DB ──CDC──▶ New Service DB
  (bidirectional sync during transition period)

Option C: Cutover
  Migrate data, switch traffic in one maintenance window
  (only feasible for small datasets or low-traffic systems)
```

## Signal phrases
- "Migrate from monolith to microservices without downtime"
- "Incremental modernisation"
- "Can't afford a full rewrite"
- "Reduce risk of migration"

## Complexity

```
Low risk:  Facade routing is well-understood (nginx, API Gateway)
High risk: Data migration and sync during transition
           Keeping both systems in sync is the hardest part
```

## Key Rules
- **Never rewrite functionality while migrating** — separate concerns: first migrate, then improve
- **Make the facade the chokepoint** — all traffic must go through it to control routing
- **Strangler should be transparent** — clients should not know or care about the internal split
- **Exit criteria per domain** — define done: old code deleted, new service owned by team

## Sources
- [[solution-arch/sources/clean-architecture]]
