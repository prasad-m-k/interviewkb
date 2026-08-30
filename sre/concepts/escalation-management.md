# Escalation Management

**Topic:** [[sre/topics/linux-cli]]
**Related:** [[devops/patterns/incident-response]], [[sre/concepts/rca-basics]], [[sre/concepts/slo-sli-sla]], [[obs/topics/alerting]]

## What it is

Escalation is the deliberate act of pulling in more people, more authority, or more resources than the current responder has, because the situation has outgrown what they can resolve alone. It's a distinct skill from troubleshooting itself — knowing WHEN and HOW to escalate, and knowing HOW to CLOSE an escalation cleanly, are both things interviewers explicitly probe, separate from "can you fix the bug."

## How it works

### Escalation tiers

```
Tier 1 — Primary on-call
  First responder. Owns triage, initial mitigation attempts, and
  the decision to escalate. Most incidents resolve here.

Tier 2 — Secondary on-call / subject-matter expert
  Pulled in when Tier 1 lacks context on the specific failing
  component, or when Tier 1's mitigation attempts haven't worked
  within a reasonable time box.

Tier 3 — Team lead / staff engineer / cross-team dependency owner
  Pulled in for architectural judgment calls, cross-team blast
  radius, or when the fix requires authority Tier 1/2 don't have
  (e.g. approving an emergency out-of-band deploy).

Executive / Incident Commander escalation
  Pulled in for customer-facing severity-1 incidents, PR/legal
  exposure, or anything requiring cross-org resource allocation
  the IC can't authorize alone. See the IC role in
  [[devops/patterns/incident-response]].
```

### When to escalate — a concrete checklist, not a feeling

```
Escalate when ANY of these is true:
  ☐ Time-boxed and no progress: you set yourself a 15-30 minute
    box to try an initial fix and it didn't work — escalating
    isn't giving up, it's respecting the time box you set
  ☐ Blast radius is growing, not shrinking, despite your mitigation
  ☐ The fix requires access, authority, or expertise you don't have
    (e.g. you need someone with prod DB write access, or the
    on-call for a dependency team you don't own)
  ☐ You're guessing, not diagnosing — if your next action is "let's
    try this and see," that's a signal you need a second opinion,
    not another guess
  ☐ Customer/exec visibility is likely regardless of outcome — better
    to loop in early on your own terms than have someone escalate
    ON you after finding out late
```

**Interview framing:** the WRONG answer to "when do you escalate" is "when I can't fix it" — that's too late and too vague. The RIGHT answer names a concrete trigger (time box, growing blast radius, missing authority) decided in advance, before the pressure of an active incident makes that judgment call harder.

### How to escalate effectively — the handoff, not just the ping

```
A good escalation message answers, in order:
  1. WHAT is broken (user-visible symptom, not internal jargon)
  2. WHO/HOW MANY are affected (blast radius — one customer? all?)
  3. WHAT you've already tried (so the next responder doesn't
     repeat wasted effort)
  4. WHAT you need from them specifically (a decision? access?
     domain expertise? just another pair of eyes?)
  5. CURRENT STATE (still degrading? stable but broken? recovering?)

BAD escalation: "prod is down, need help"
GOOD escalation: "checkout API error rate at 15% since 10:34 UTC,
  affects all US-region traffic (~40% of total). Rolled back the
  10:28 deploy — no change, so it's likely not the deploy. Need
  someone with payment-svc DB access to check for a connection
  pool exhaustion; I don't have prod DB creds. Error rate still
  climbing as of 10:52."
```

The bad version forces the next responder to re-derive everything the escalator already knows — wasting the exact time escalation was supposed to save.

### How to close an escalation

Closing is not "the page stopped firing." A properly closed escalation has an explicit exit checklist, because an escalation left implicitly open is how the SAME issue reopens an hour later with a different responder starting from zero:

```
An escalation is closed only when ALL of these are true:
  ☐ Technical resolution VERIFIED, not assumed — metrics/dashboards
    confirmed back to baseline, not just "the error stopped showing
    up in the last 2 minutes" (check a long-enough window to rule
    out a lull, not a fix)
  ☐ The escalated party/team explicitly confirms handback — the
    person who escalated doesn't unilaterally decide it's resolved
    if someone else was pulled in to fix it; that person confirms
  ☐ Stakeholders who were notified of the escalation are notified
    of the resolution too — an escalation that goes silent after
    the fix, with no "all clear" message, erodes trust in the next
    escalation's urgency
  ☐ Follow-up items are FILED (ticketed, owned, dated) before the
    incident channel goes quiet — "we'll fix that properly later"
    that isn't ticketed is a promise that evaporates
  ☐ Severity is formally downgraded/closed in whatever tracking
    system declared it (not left at SEV-1 indefinitely) — this
    matters because SLO/error-budget accounting and org-level
    incident metrics depend on accurate open/close timestamps, see
    [[sre/concepts/slo-sli-sla]]
  ☐ RCA is scheduled (even if not yet completed) for anything above
    a minor severity — see [[sre/concepts/rca-basics]] for the
    methodology; closing WITHOUT scheduling this is how the same
    root cause quietly recurs
```

## Complexity

Not algorithmic. The cost of getting escalation wrong runs both directions: escalating too late wastes the blast-radius-growth window (the earlier you pull in the right expertise, the smaller the damage); escalating too often/vaguely (pinging Tier 3 for everything, or with incomplete context) burns trust and trains people to deprioritize your pages — the same "alert fatigue" dynamic covered in [[obs/topics/alerting]], now applied to human escalation chains instead of automated alerts.

## When to use

```
- Any active incident where the checklist above triggers
- Cross-team dependencies: escalate to the OWNING team's on-call,
  not just anyone who might know — a misdirected escalation costs
  the same time as no escalation at all
- Security incidents: escalate to security/legal EARLY even before
  full technical triage is done — the disclosure clock (see the
  security-patch SLA example in [[sre/concepts/slo-sli-sla]]) often
  starts from discovery, not from resolution
```

## Common interview angles

```
Q: "How do you decide when to escalate versus keep trying to fix it
    yourself?"
A: Name a concrete pre-committed trigger, not a feeling — a time
   box you set for yourself, a blast radius that's still growing
   despite mitigation, or a fix that needs access/expertise you
   don't have. Escalating on a pre-set trigger, rather than waiting
   until you're stuck, is the senior-engineer answer.

Q: "You escalated an incident to another team, and they fixed it.
    What do you do next?"
A: Confirm the fix is VERIFIED (not just "errors stopped"), get
   explicit handback confirmation from the team that fixed it,
   notify whoever was told about the escalation that it's resolved,
   file any follow-up items with owners, and formally close the
   severity — not just let the incident channel go quiet.

Q: "An escalation you sent got ignored for 20 minutes during an
    active outage. What went wrong, and what would you do
    differently?"
A: Likely one of: wrong tier/team was paged (misdirected
   escalation), the message didn't convey enough urgency/blast
   radius to justify interrupting someone, or general alert
   fatigue from a history of low-quality escalations. Fix going
   forward: escalate to the correct owning team directly, lead the
   message with concrete blast radius, and audit past escalations
   for false alarms that eroded trust in genuine ones.
```

## Examples

```
Escalation message template (Slack/incident channel):
  🔺 ESCALATING: [what's broken] — [blast radius]
  Tried: [what you already attempted]
  Need: [specific ask — access / decision / expertise]
  Status: [degrading / stable / recovering] as of [timestamp]

Closing message template:
  ✅ RESOLVED: [what was broken] is back to baseline as of
  [timestamp], confirmed via [dashboard/metric].
  Handback confirmed by: [name]
  Follow-ups filed: [ticket links]
  RCA scheduled: [date/owner]
```

## Sources
- [[devops/patterns/incident-response]]
- [[sre/concepts/rca-basics]]
- [[sre/concepts/slo-sli-sla]]
- [[obs/topics/alerting]]
