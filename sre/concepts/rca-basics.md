# Root Cause Analysis (RCA) Basics

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[devops/patterns/incident-response]], [[sre/concepts/escalation-management]], [[sre/concepts/slo-sli-sla]]

## What it is

RCA is the structured process of finding WHY an incident happened, deep enough that fixing what you find actually prevents recurrence — as opposed to fixing only the symptom that paged you. It's the methodology that fills the "Root Cause" section of the postmortem template in [[devops/patterns/incident-response]]; this page covers the technique, that page covers the overall incident/postmortem process it feeds into.

## How it works

### Root cause vs. contributing factor vs. trigger — the distinction interviewers test first

```
Trigger event
  The immediate thing that set off the incident. Often not the
  real problem — just the spark.
  Example: "A deploy at 10:28 UTC introduced a null-check bug."

Contributing factors
  Conditions that made the trigger turn into a full incident
  instead of a non-event. Usually MULTIPLE factors compound.
  Example: "Integration tests didn't cover this null case; canary
  period was only 5 minutes, too short to catch a 1%-of-traffic
  tail case; no alert existed on this specific error type."

Root cause
  The systemic reason the failure was POSSIBLE at all — usually a
  process or design gap, not a single line of code.
  Example: "The test suite has no requirement to cover nullable
  fields from legacy data, and canary duration was set by
  convention, never revisited against actual traffic patterns."
```

**The test for "have I actually found the root cause":** if you fixed ONLY the trigger (patch the null check) and changed nothing else, could the SAME CLASS of bug reappear next week in a different field? If yes, you found a contributing factor, not the root cause — keep going.

### The 5 Whys — the simplest, most commonly asked technique

```
Problem: Checkout API returned 500s for 15% of users for 17 minutes.

Why #1: Why did it return 500s?
  → payment-svc threw an unhandled NullPointerException.

Why #2: Why was there a null value it couldn't handle?
  → card_type was null for accounts created before 2019, before
    that field existed.

Why #3: Why didn't testing catch this?
  → Integration tests use only synthetic post-2019 test accounts;
    no test fixture represents pre-2019 legacy accounts.

Why #4: Why do test fixtures not cover legacy account shapes?
  → No requirement/checklist exists for new code changes to
    consider backward-compatible data shapes, only forward-looking
    ones.

Why #5: Why is there no such requirement?
  → The team's code review checklist was written when the service
    was new (all data was "new shape") and was never revisited as
    the service accumulated years of legacy data.

ROOT CAUSE: the code review checklist has no prompt to consider
legacy/pre-migration data shapes — a process gap, not a code bug.
Fixing just the null check (stop at Why #1) leaves this exact class
of bug free to recur on the next legacy field.
```

**When 5 Whys breaks down:** it assumes a single linear causal chain. Real incidents are often MULTI-CAUSAL (several independent contributing factors that all had to align) — 5 Whys pursued rigidly down one branch can miss the others entirely. This is precisely what Fishbone/Ishikawa is for.

### Fishbone (Ishikawa) diagram — for multi-causal incidents

```
Used when an incident has several INDEPENDENT contributing causes,
not one linear chain. Organize candidate causes into standard
categories, then ask "why" within each branch:

    People/Process           Code/Design               Testing            Deployment/Process  
   (no legacy-data       (missing null check     (fixtures only cover     (canary period too  
      checklist)            on card_type)        post-2019 accounts)      short for 1% tail)  
          │                       │                       │                       │           
──────────────────────────────────────────────────────────────────────────────────────────────
                                              ▼                                               
                           [INCIDENT: checkout 500s, 15% of users]                            

Each branch gets its own "why" chain. The value over 5 Whys: it
forces you to check EVERY category (people/process, code, testing,
deployment, monitoring) instead of stopping once ONE plausible
chain is found — which is exactly the most common RCA mistake.
```

### Corrective vs. preventive action — the fix has two different jobs

```
Corrective action
  Fixes THIS specific instance of the problem.
  Example: add the null check for card_type.

Preventive action
  Fixes the SYSTEMIC gap so the next similar-but-different bug
  doesn't reach production the same way.
  Example: add a "legacy data shape" prompt to the code review
  checklist; extend canary period from 5 to 30 minutes; add
  alerting on this specific exception class.

A postmortem action-item list that's ALL corrective (just patches)
without any preventive items has not actually completed RCA — it's
addressed the trigger and stopped there.
```

### Common RCA pitfalls

```
1. Stopping at the proximate cause
   "The deploy caused it" is TRUE but not USEFUL — deploys cause
   most incidents by definition (something changed). Keep asking
   why until you reach a process/design gap, not just "a change
   was made."

2. "Human error" as a terminal answer
   "The engineer forgot to check X" is never a root cause — it's a
   restatement of the trigger. The real question is: why did the
   system allow that mistake to reach production without a
   safeguard (a review checklist, a linter, a test, an alert)?
   Blameless RCA asks "what about the SYSTEM allowed this," not
   "who made the mistake."

3. Single-cause bias
   Real incidents are usually multi-causal (see Fishbone above).
   Stopping the RCA once ONE plausible cause is found — even if
   correct — misses the other contributing factors that will cause
   the NEXT incident even after this one's cause is fixed.

4. Confusing "could have caught it" with "root cause"
   "Better monitoring would have caught it sooner" is true of
   nearly every incident and is a valid contributing-factor finding
   — but it explains DETECTION time, not WHY the underlying failure
   was possible. Don't let it substitute for actually finding the
   design/process gap.

5. Action items with no owner or date
   Not strictly an RCA-methodology mistake, but the most common way
   good RCA work gets wasted — see the postmortem template in
   [[devops/patterns/incident-response]] for the owner/due-date
   discipline that makes RCA findings actually stick.
```

## Complexity

Not algorithmic. The cost trade-off is time: a rigorous RCA (multiple why-chains, fishbone across categories) takes longer than a quick "deploy caused it, rolled back, done" — but an incomplete RCA is a recurring-incident generator, so the time investment scales with the incident's severity and likelihood of recurrence, not applied uniformly to every minor blip.

## When to use

```
- Any incident above a minor severity (see the severity table in
  [[devops/patterns/incident-response]]) — always schedule RCA
  before formally closing the escalation, per
  [[sre/concepts/escalation-management]]'s close checklist
- Any RECURRING issue, even individually minor ones — three small
  incidents with the same underlying cause deserve the same rigor
  as one large one, because the pattern itself IS the signal
- Skip full RCA rigor for genuinely one-off, non-recurring blips
  (e.g. a transient cloud-provider network glitch with no code/
  process gap on your side) — matching RCA depth to actual
  recurrence risk, not treating every ticket identically
```

## Common interview angles

```
Q: "Walk me through how you'd do root cause analysis on a
    production incident."
A: Start from the trigger, apply 5 Whys down the most obvious
   chain — but explicitly check whether the incident is multi-
   causal (fishbone categories: people/process, code, testing,
   deployment, monitoring) rather than assuming one linear chain
   explains everything. Stop only when the answer is a process/
   design gap, not a restatement of "a change was made" or "someone
   made a mistake."

Q: "Why isn't 'human error' an acceptable root cause?"
A: It's a restatement of the trigger, not an explanation of why the
   system allowed that error to reach production unguarded. The
   useful question is what missing safeguard — review checklist,
   test coverage, linter rule, alert — would have caught it
   regardless of which specific person made the mistake.

Q: "You did an RCA and found five contributing factors. How do you
    prioritize which to fix?"
A: Weight by (a) how directly each factor enabled THIS incident's
   trigger, and (b) how likely each factor is to enable a DIFFERENT
   future incident if left unfixed — a factor like "no test
   coverage for legacy data shapes" scores high on both, while a
   narrow one-off factor might only warrant a corrective fix, not a
   full preventive action item.

Q: "What's the difference between corrective and preventive action
    items in a postmortem?"
A: Corrective fixes THIS instance (patch the specific bug).
   Preventive fixes the systemic gap that allowed it (process,
   testing, or monitoring change) so a DIFFERENT-but-similar bug
   doesn't repeat the same failure mode. A postmortem with only
   corrective items hasn't finished the RCA.
```

## Examples

```
RCA summary as it would appear in a postmortem's Root Cause section:

  Trigger: Deploy of payment-svc v2.3.1 (10:28 UTC) introduced an
  unhandled NullPointerException on card_type.

  Root cause: The code review checklist has no prompt to consider
  legacy/pre-migration data shapes, so backward-compatibility gaps
  like this one routinely pass review undetected.

  Contributing factors:
    - Integration test fixtures only cover post-2019 account shapes
    - Canary period (5 min) too short to surface a 1%-of-traffic
      tail case
    - No alert existed on this specific exception class

  Corrective action: Add null check for card_type. [done, same-day]
  Preventive actions:
    - Add "legacy data shape" item to code review checklist [owner, date]
    - Extend canary period 5min → 30min [owner, date]
    - Add alert on this exception class [owner, date]
```

## Sources
- [[devops/patterns/incident-response]]
- [[sre/concepts/escalation-management]]
- [[sre/concepts/slo-sli-sla]]
