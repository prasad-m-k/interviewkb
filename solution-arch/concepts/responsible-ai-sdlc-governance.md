---
uid: b831714b-cfcc-49b7-bcde-ac61429c2ff2
---

# Responsible AI Governance Over AI-Generated Engineering Artifacts

**Topic:** [[solution-arch/topics/ai-governance-responsible-ai]]
**Related:** [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/concepts/llm-observability-and-evals]], [[solution-arch/companies/microsoft-coreai-responsible-ai]]
**Tags:** #ResponsibleAI

## What it is

Most Responsible AI material (including [[solution-arch/topics/ai-governance-responsible-ai]]) governs the AI **product** you ship to a customer. This concept covers a different, narrower surface: governing the AI **tooling** your own engineering org uses to build software — Copilot-style code generation, AI-drafted requirements, and AI-authored design documents — so that hallucinated or subtly wrong AI output doesn't silently become the requirement, the design, or the merged code. This is the explicit scope named in Microsoft's CoreAI Responsible AI Principal Engineer role: *"take responsibility for the content of AI-generated requirements, design documents, code, and other assets, and incorporate Responsible AI practices into the SDLC to ensure appropriate controls over AI-generated assets."*

The distinction matters in an interview: a candidate who only talks about output moderation and jailbreak defense is answering "how do you govern the AI product." This concept is "how do you govern the AI **used to build** the product" — a second, orthogonal governance surface that a CoreAI-style team must own because it builds both the RAI platform *and* increasingly uses AI copilots internally to build it.

## How it works

```
┌───────────────────────────────────────────────────────────────────┐
│ SDLC STAGE                  →  AI-GENERATED RISK                  │
├───────────────────────────────────────────────────────────────────┤
│ Requirements                →  AI drafts a requirement that       │
│ (AI-assisted spec drafting)    overstates feasibility or omits    │
│                                a constraint no human validated    │
├───────────────────────────────────────────────────────────────────┤
│ Design                      →  AI-authored design doc cites a     │
│                                pattern/API that doesn't exist,    │
│                                or silently drops an NFR because   │
│                                it wasn't in the prompt context    │
├───────────────────────────────────────────────────────────────────┤
│ Code                        →  Hallucinated dependency ("package  │
│ (Copilot/agentic code gen)     hallucination" / slopsquatting     │
│                                risk), insecure pattern over-      │
│                                represented in training data,      │
│                                logic that passes a superficial    │
│                                review but is subtly wrong         │
├───────────────────────────────────────────────────────────────────┤
│ Review                      →  Automation bias: reviewer trusts   │
│                                confident-sounding AI output and   │
│                                rubber-stamps instead of verifying │
├───────────────────────────────────────────────────────────────────┤
│ Test                        →  AI-generated tests assert the      │
│                                AI-generated implementation's      │
│                                (possibly wrong) behavior instead  │
│                                of the actual requirement          │
├───────────────────────────────────────────────────────────────────┤
│ Deploy / Monitor            →  No provenance trail — six months   │
│                                later nobody can say which lines   │
│                                were AI-authored vs human-authored │
│                                when triaging an incident          │
└───────────────────────────────────────────────────────────────────┘
```

### The control stack

```
1. Provenance & disclosure
   Every AI-assisted artifact (doc, PR, commit) is tagged with what
   was AI-generated vs human-authored. Not for blame — so a future
   incident review or audit knows where to look first.

2. Named human owner of record
   Same principle as "a human is accountable for a model's decision"
   in product-facing RAI — extended to engineering artifacts. AI
   drafts; a specific engineer signs off and owns the consequences.
   No artifact ships on AI authorship alone.

3. Dependency/supply-chain verification gate
   Hallucinated package names are a real, named attack surface
   (an attacker registers the plausible-but-fake package name an
   LLM tends to suggest). CI blocks any dependency not already in
   an approved registry/allowlist — same shape as a SBOM gate, now
   also defending against the model's own hallucination, not just
   external supply-chain compromise.

4. Elevated review bar for AI-authored code/design, not a lower one
   Counterintuitive but correct: because automation bias makes
   reviewers under-scrutinize confident AI output, the review
   checklist for AI-authored changes should be MORE structured
   (explicit checklist, not "looks fine") than for human-authored
   changes, not less.

5. Security scanning tuned for AI-specific patterns
   Static analysis / SAST tuned to catch the specific insecure
   patterns AI code generators reproduce disproportionately often
   (e.g., string-concatenated queries, missing input validation on
   generated boilerplate) — an addition to standard SAST, not a
   replacement.

6. Sampling audits
   Periodic audit of a random sample of merged AI-assisted changes,
   independent of the PR review that already happened — the same
   "trust but verify" logic as [[solution-arch/concepts/llm-observability-and-evals]]
   applied to eval-gated model deploys, now applied to eval-gated
   *code* deploys.
```

## Complexity

Not algorithmic. The real cost is organizational: provenance tagging and elevated review only work if tooling enforces them (PR templates with a mandatory "AI-assisted: Y/N + which parts" field, CI gates that block on missing disclosure) rather than relying on engineers to remember. Treat it the same way you'd treat any other non-negotiable compliance gate in the pipeline.

## When to use

```
Always, once an org adopts AI code/doc generation at scale:
  - Any team using Copilot-style code generation in a regulated or
    production-critical codebase
  - Any team where AI drafts requirements or design docs that
    downstream engineers implement against without independently
    re-deriving the requirement
  - Especially relevant for a team (like CoreAI) that BOTH builds
    the RAI platform AND is itself a heavy internal consumer of AI
    coding tools — the team must hold itself to its own standard
```

## Common interview angles

```
Q: "Your org rolled out an internal AI coding assistant. Six months
    later, a production incident traces back to a hallucinated
    dependency the assistant suggested and a reviewer approved
    without checking. How would you have prevented this?"
A: Name the specific control that was missing — a dependency
   verification gate in CI that blocks unregistered/unapproved
   packages regardless of how confidently the assistant suggested
   them. Then generalize: this is a supply-chain control, not a
   'try to write better prompts' problem — the fix belongs in the
   pipeline, not in developer discipline alone.

Q: "How is governing AI-generated CODE different from governing an
    AI PRODUCT's output?"
A: Product governance (see [[solution-arch/concepts/ai-guardrails-and-safety]])
   guards what a model says to an external user, in real time, per
   request. SDLC governance guards what an internal AI tool
   contributes to your own codebase/design corpus, asynchronously,
   before it becomes load-bearing infrastructure other systems will
   depend on. The blast radius is different: a bad model response
   affects one user interaction; a bad AI-authored design doc or
   dependency can become the foundation many future systems build on.

Q: "Doesn't code review already catch this?"
A: Only if the review bar for AI-authored changes is explicitly
   raised, because automation bias means reviewers are measurably
   LESS scrutinous of confident, well-formatted AI output than of
   a colleague's rough draft. The control isn't 'we have code
   review' — it's 'we have a review checklist specifically
   calibrated to counteract automation bias for AI-assisted changes.'
```

## Examples

```
Provenance tag in a PR template:
  AI-Assisted: Yes
  Scope: Function bodies in payment_reconciler.py (3 of 5 files);
         design doc section 4.2 drafted by Copilot, reviewed and
         edited by [engineer]
  Human owner of record: [engineer] — accountable for correctness
  Dependency check: PASSED (no new deps outside approved registry)
```

## Sources
- [[solution-arch/topics/ai-governance-responsible-ai]]
- [[solution-arch/companies/microsoft-coreai-responsible-ai]]
