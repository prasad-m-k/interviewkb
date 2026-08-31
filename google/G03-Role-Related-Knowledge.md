# Role-Related Knowledge (RRK)

> What Google is testing: deep mastery of your craft — not trivia recall, but whether your technical judgment holds up under real constraints (backward compatibility, production risk, review scrutiny).

**Traits emphasized:** Goal oriented · Honest (about your own draft's weaknesses)
**Adapted from:** [[amazon/LP01-Customer-Obsession]] · [[amazon/LP07-Insist-on-Highest-Standards]]

---

## STAR Format — Backward-Compatible Pagination

**Situation:** At PNC, a downstream team was repeatedly hitting a paginated alert endpoint that returned inconsistent results, because the REST API had no formal pagination support — it returned everything or nothing depending on undocumented caller behavior. Business teams were making decisions on stale or incomplete data.

**Task:** I owned the alert/announcement endpoints and needed to redesign pagination without breaking existing integrations that expected the old flat response structure.

**Action:** I introduced nullable `Integer` `page`/`size` parameters so callers could opt into pagination without a forced migration, built a `PagedResponse` wrapper that preserved backward compatibility, and wrote a JPQL correlated subquery to compute active alert counts accurately at query time — eliminating a stale-count bug that had affected dashboards for months. I validated the new contract with two downstream teams before deployment.

**Result:** The endpoint went live with zero downstream incidents, alert counts matched real-time state, and the design became the reference pattern adopted by two other endpoints in the same service.

**Decision · Reasoning · Outcome**
- **Decision:** Make pagination opt-in via nullable parameters rather than a breaking contract change.
- **Reasoning:** Callers ranged from dashboards to batch jobs with no documented contract anywhere; a forced migration risked incidents I couldn't fully scope in advance.
- **Outcome:** Zero downstream incidents, and the pattern was reused elsewhere without anyone having to ask.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Average response payload cut an estimated 70% for large alert result sets by returning only the requested page instead of the full set.
- p95 latency under peak load estimated to drop from ~1.8s to under 400ms.
- Eliminated an estimated 3–4 stale-count dashboard escalations per month.

**Why this reads L6/L7:** The design decision (nullable-param backward compatibility, not a versioned breaking change) became the de facto pattern for two other endpoints in the service — you set a convention other engineers followed without being told to.

---

## STAR Format — Raising My Own Bar on a Research Paper

**Situation:** An early draft of the Agent Security Policy (ASP) paper — on trust boundaries in agentic AI systems — framed the problem primarily as prompt-injection defense. Technically accurate, but conceptually shallow: it would have passed as a competent survey and advanced nothing.

**Task:** As both author and the only quality gate before submission, I had to decide whether "publishable" was actually the bar, or whether it was "publishable and something I'd stand behind."

**Action:** I reframed the paper around a structural gap — that agentic AI systems lack a trust boundary layer analogous to firewalls in network security or privilege separation in OS security — and rebuilt the problem statement, threat model, and framework from that foundation. I retargeted the venue to USENIX PEPR, which demands a more rigorous security-systems argument than a general AI conference.

**Result:** The reframed paper introduces a construct that doesn't yet exist in the literature. The timeline extended by weeks; the contribution became defensible against a security-systems audience instead of merely competent.

**Decision · Reasoning · Outcome**
- **Decision:** Rebuild the paper's core argument rather than polish and ship the first draft.
- **Reasoning:** No external reviewer was going to catch the shallowness before submission — the bar had to be self-imposed.
- **Outcome:** A weeks-longer timeline traded for a paper that makes a novel, falsifiable claim instead of a forgettable one.

**Quantified impact** *(illustrative estimate — replace with real numbers before using live)*
- Retargeted from a general AI venue (near-certain acceptance, low field visibility) to USENIX PEPR (~15–20% acceptance rate) — a harder bar that signals the reframed contribution actually holds up.
- Introduces a construct (an agentic-system trust boundary layer) with no direct prior art, positioning it to be citable as the agentic AI security space matures.

**Why this reads L6/L7:** Raising your own bar on a paper only you were gating — with zero external pressure forcing it — is a judgment-quality signal, the same instinct a Staff engineer needs when there's no one above them to catch a shallow design before it ships.

## Sample interview questions this answers
- "Describe a complex problem you solved at work." (Problem Solving)
- "Give an example of when you identified a problem before it became critical." (Problem Solving)
- Role-specific deep-dive questions on your primary stack (Spring Boot / API design / distributed systems).

---

*Tags: #Googliness #role-related-knowledge #api-design #pnc #research-quality #asp-paper*
