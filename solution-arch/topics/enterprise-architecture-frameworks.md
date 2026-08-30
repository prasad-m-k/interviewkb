# Enterprise Architecture Frameworks & Governance

**Related:** [[solution-arch/overview]], [[solution-arch/topics/architectural-styles]], [[solution-arch/topics/ai-governance-responsible-ai]], [[solution-arch/topics/cost-architecture-finops]]

> **Gap-fill note:** the existing Solution Architecture KB covers architectural *styles and patterns* in depth but had no page on the *governance frameworks and organizational role structures* that senior/principal SA interviews frequently probe — TOGAF, cloud Well-Architected Frameworks, and how EA/SA/TA roles differ. This page fills that gap and applies equally whether the system under discussion is AI-based or not.

---

## EA vs SA vs TA — A Role Question Interviewers Actually Ask

```
Enterprise Architect (EA)
  Scope: entire organization, multi-year horizon
  Owns: technology strategy, standards, vendor rationalization,
        portfolio-level trade-offs (build vs buy across the org)
  Artifact: reference architectures, technology roadmaps

Solution Architect (SA)
  Scope: one program/product, months-to-a-year horizon
  Owns: the technical blueprint for a specific business problem,
        integration with adjacent systems, NFR trade-offs
  Artifact: solution design documents, ADRs, C4 diagrams

Technical Architect / Application Architect (TA)
  Scope: one application/service, weeks-to-months horizon
  Owns: internal design of a single system — module boundaries,
        class/component design, technology-specific implementation
        decisions
  Artifact: detailed design docs, code-level architecture

                EA
                 │  sets standards / strategy
                 ▼
                SA  ◀── you are typically here
                 │  designs the solution within EA guardrails
                 ▼
                TA
                    implements within the SA's blueprint
```

**Why this matters in an interview:** being asked "design a system" without asking "what's the scope — one service, or the whole platform?" signals you don't know which of these three hats you're wearing. A strong opening move in any design interview is explicitly scoping: "Is this SA-level — one product's blueprint — or are we also deciding organization-wide standards?"

---

## TOGAF (The Open Group Architecture Framework) — What to Actually Know

Most SA interviews don't require reciting all of TOGAF's ADM phases from memory, but knowing the shape signals EA-adjacent maturity:

```
TOGAF Architecture Development Method (ADM) — simplified:

  Preliminary  → establish architecture capability, principles
       │
       ▼
  A: Architecture Vision   → stakeholder concerns, scope, goals
       │
       ▼
  B: Business Architecture  → business capabilities, processes
       │
       ▼
  C: Information Systems     → data architecture + application
     Architecture              architecture
       │
       ▼
  D: Technology Architecture → infrastructure, platforms
       │
       ▼
  E: Opportunities & Solutions → how to get there (build vs buy,
                                  work packages)
       │
       ▼
  F: Migration Planning      → sequencing, roadmap
       │
       ▼
  G: Implementation Governance → ensure delivery matches architecture
       │
       ▼
  H: Architecture Change Mgmt → handle change requests, drift

  (Requirements Management sits at the center, feeding all phases)
```

**Practical takeaway for an SA interview:** TOGAF's real value isn't the ceremony — it's the discipline of separating **Business → Data/Application → Technology** layers explicitly, and having a **formal change management** step so architecture doesn't silently drift from what was approved. Reference this shape when asked "how do you keep architecture from rotting after the initial design."

---

## Cloud Well-Architected Frameworks — The Modern Equivalent, Cloud-Specific

Every major cloud vendor publishes a pillar-based framework; interviewers expect you to map any design decision back to these pillars.

```
                  AWS              Azure             GCP
Pillar 1:    Operational      Reliability      Operational
             Excellence                        Excellence
Pillar 2:    Security         Security         Security, Privacy
                                                & Compliance
Pillar 3:    Reliability      Cost              Reliability
                              Optimization
Pillar 4:    Performance      Operational       Performance &
             Efficiency       Excellence        Cost Optimization
Pillar 5:    Cost             Performance       (folded into above)
             Optimization     Efficiency
Pillar 6:    Sustainability   —                 —

All three converge on the same core five/six concerns:
  Operations, Security, Reliability, Performance, Cost
  (+ Sustainability as the newest addition)
```

**How this gets used in a real interview:** after you propose a design, a strong follow-up is self-critique against these pillars — "for cost, I'd flag the always-on GPU inference endpoint as the biggest lever; for reliability, the single-region vector store is the weak point." Doing this unprompted is a senior-level signal.

---

## Architecture Governance in Practice

```
┌────────────────────────────────────────────────────────────┐
│              ARCHITECTURE GOVERNANCE LIFECYCLE               │
├────────────────────────────────────────────────────────────┤
│ 1. Architecture Review Board (ARB) / Design Review           │
│    New system or significant change presents a design doc    │
│    + ADRs to a review board before build starts. Purpose:    │
│    catch NFR gaps, security issues, standards violations      │
│    EARLY, when they're cheap to fix.                          │
├────────────────────────────────────────────────────────────┤
│ 2. Architecture Decision Records (ADRs)                       │
│    See [[solution-arch/overview]] for the format. The         │
│    durable, version-controlled record of WHY — this is what   │
│    prevents "why did we build it this way?" from having no    │
│    answer 2 years later.                                       │
├────────────────────────────────────────────────────────────┤
│ 3. Reference architectures / paved roads                      │
│    EA-level standard patterns (e.g. "all new services use     │
│    this event bus, this auth pattern") that SAs build within,  │
│    reducing bespoke risk per project.                          │
├────────────────────────────────────────────────────────────┤
│ 4. Architecture fitness functions                              │
│    Automated checks (in CI) that continuously verify an        │
│    architecture characteristic holds — e.g. a test that fails  │
│    the build if a module reaches into another team's database, │
│    or if a new dependency violates an approved-vendor list.     │
│    Keeps architecture intent enforced in code, not just in a    │
│    document nobody re-reads.                                    │
├────────────────────────────────────────────────────────────┤
│ 5. Post-implementation review                                  │
│    Did the system meet the NFRs the design promised? Feeds      │
│    back into the reference architecture / standards for next    │
│    time.                                                         │
└────────────────────────────────────────────────────────────┘
```

---

## Build vs Buy vs Integrate — The Recurring SA Decision

A framework that recurs constantly in both classic and AI SA interviews:

```
                Does a mature vendor/OSS solution already
                solve 80%+ of the requirement?
                          │
              ┌───────────┴───────────┐
             YES                      NO
              │                        │
              ▼                        ▼
     Is the remaining 20%      Build — but check:
     gap a genuine              is this actually core
     differentiator for         to the business, or
     the business, or is it     undifferentiated heavy
     undifferentiated           lifting? If the latter,
     "glue work"?                reconsider — this is
              │                  often a sign a narrower
     ┌────────┴────────┐         vendor solves it.
    YES                NO
     │                  │
     ▼                  ▼
   Buy + build     Buy / integrate
   a thin custom   as-is
   layer on top
   (integrate)

Weighting factors always worth naming explicitly in an interview:
  - Total cost of ownership (license + integration + ongoing ops)
  - Vendor lock-in risk and exit cost
  - Team's existing skill set
  - Time-to-market pressure
  - Whether the capability is a competitive differentiator
    (build) or table stakes (buy)
```

---

## Sources
- [[solution-arch/overview]]
- [[solution-arch/sources/clean-architecture]]
