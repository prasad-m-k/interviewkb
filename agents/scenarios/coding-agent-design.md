# Coding Agent — System Design

**Difficulty:** Hard
**Topic:** [[agents/topics/agent-architectures]]
**Pattern:** [[agents/patterns/plan-and-execute]], [[agents/patterns/react-pattern]]
**Related:** [[agents/concepts/context-engineering]], [[agents/concepts/guardrails-and-safety]]

---

## Problem

Design an agent that can navigate a large existing codebase, make multi-file changes, run tests, and iterate — the class of system this vault's own working environment (Claude Code) belongs to. Good for teaching because students already have direct intuition for what "good" looks like.

---

## Clarifying Questions

- Is the codebase already on disk locally, or does the agent need to clone/fetch it first?
- What's the blast radius of a mistake? (a personal project vs. a production monorepo with CI/CD)
- Should the agent be allowed to run arbitrary shell commands, or only a constrained tool set (read/write/search/test)?
- Interactive (human watching, can interrupt) or fully autonomous (fire-and-forget on a ticket)?

---

## Requirements

| Type | Requirement |
|---|---|
| Functional | Explore a codebase it's never seen, locate relevant files, make correct multi-file edits, verify via tests |
| Non-functional | Bounded cost/time per task, context stays usable even in a large repo |
| Safety | No destructive action (force-push, `rm -rf`, credential exposure) without explicit gating |

---

## Architecture

```
Task: "Fix the bug where pagination returns duplicate results on page 2"

┌────────────────────────────────────────────────────────────────┐
│                       CODING AGENT LOOP                        │
│                                                                │
│ Tools:                                                         │
│ - search(query)      — grep/semantic search across the repo    │
│ - read_file(path)                                              │
│ - edit_file(path, diff)                                        │
│ - run_command(cmd)   — sandboxed shell (tests, linters, build) │
│ - run_tests(scope)                                             │
└────────────────────────────────┬───────────────────────────────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           ▼                     ▼                     ▼
         1. LOCATE             2. UNDERSTAND         3. FIX + VERIFY
         search for            read the relevant     edit_file() →
         "pagination"          files fully before    run_tests() →
         across the repo       editing blindly       if fail: reason
                                                     about why, retry
                                                     (reflection pass)
```

This maps directly onto [[agents/patterns/plan-and-execute]] at the coarse level (locate → understand → fix → verify) with a [[agents/patterns/react-pattern]] loop inside each step for the actual tool-by-tool work, and a [[agents/patterns/reflection-pattern]] pass before accepting a fix as complete.

---

## Context Engineering — the central challenge

A large codebase cannot fit in context. This is the practical, high-stakes version of everything in [[agents/concepts/context-engineering]]:

| Technique | Applied here |
|---|---|
| Search before read | Never dump the whole repo into context; locate relevant files via search first, read only what's needed |
| Trim tool output | `run_tests` output should surface failures, not thousands of lines of passing test noise |
| Sub-agent isolation | A large refactor can delegate "update all callers of this function" to a sub-agent that returns only a summary of what changed |
| Incremental editing | Edit and verify one logical change at a time rather than batching many unrelated edits into one giant diff |

---

## Guardrails

| Risk | Control |
|---|---|
| Destructive shell commands (`rm -rf`, force-push) | Command allow-list or explicit deny-list; irreversible git operations require approval — see [[agents/concepts/guardrails-and-safety]] |
| Credential/secret exposure | Never read `.env`/credential files into context unless explicitly required and permitted; scrub before logging |
| Runaway iteration (repeatedly failing the same test) | Max retry count per sub-task, loop detection — see [[agents/concepts/agent-loop]] |
| Silent wrong fix | Require tests to pass (or a human review step) before considering a task complete — never "fix and assume" |

---

## Evaluation

- **Task success:** does the change actually fix the bug and pass tests, not just "look plausible"?
- **Trajectory quality:** did it search efficiently, or thrash (repeated redundant searches, edits it then reverted)? See [[agents/concepts/agent-evaluation]].
- **Blast radius:** did it touch only files relevant to the task, or make unrelated changes?
- **Regression suite:** re-run a fixed set of representative coding tasks on every change to the agent's prompt/tool set, per the discipline in [[agents/concepts/agent-evaluation]].

---

## Key Insight

The hardest part of coding-agent design is not "can it write correct code" — models are already reasonably good at that in isolation. It's **context management at scale** (finding the right slice of a large codebase without reading all of it) and **verification discipline** (never trusting a change until tests confirm it) — both are context-engineering and guardrail problems, not raw model-capability problems.

---

## Sources
- [[agents/concepts/context-engineering]]
- [[agents/patterns/plan-and-execute]]
- [[agents/patterns/reflection-pattern]]
- [[agents/concepts/guardrails-and-safety]]
