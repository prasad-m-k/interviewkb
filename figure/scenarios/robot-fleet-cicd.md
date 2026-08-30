# CI/CD Pipeline for a Humanoid Robot Fleet

**Difficulty:** Hard
**Topic:** Infrastructure design
**Companies:** [[figure/index]]
**Related:** [[devops/concepts/ci-cd-pipeline]], [[devops/concepts/deployment-strategies]], [[devops/concepts/kubernetes-architecture]]

---

## Problem

Design a CI/CD pipeline for deploying software updates to a fleet of 1,000 humanoid robots operating in active manufacturing environments. Updates include: OS patches, control software, and ML model weights. Downtime on the manufacturing floor costs $10,000/minute.

---

## Clarifying Questions (ask first)

- What software components are being deployed? (OS, application stack, ML models — each has different risk profiles)
- What does the robot's connectivity look like? (Always-on WiFi? Intermittent? Cellular for off-site?)
- What are the acceptance criteria before a version is approved for fleet-wide rollout?
- Are robots in active operation during deployment, or can they be taken out of service for updates?
- What is the maximum acceptable robot downtime window per update? Per robot? Per fleet?
- Is there a hardware-in-the-loop (HIL) test environment?

---

## The Core Complexity

A robot fleet CI/CD pipeline is not a web service rolling deployment scaled up. Four things make it fundamentally different:

1. **Physical validation is required.** A software change that passes all automated tests might still cause a robot to fall or damage a workpiece. Automated tests can only cover what you simulate. Real hardware testing is a required gate.

2. **Rollback is not always instant.** Rolling back a web service means re-routing traffic. Rolling back a robot may mean the robot must complete its current task cycle, gracefully park, pull the previous artifact, and restart. On a production line, that might take 5–15 minutes per robot.

3. **Deployment artifacts are heterogeneous.** OS packages, ROS2 packages, PyTorch model weights, firmware binaries, and configuration files each require different deployment mechanisms, different testing strategies, and different rollback procedures.

4. **Safety gates are non-negotiable.** The pipeline must prevent unsafe software from reaching production even if it passes all unit tests. This requires dedicated safety validation infrastructure.

---

## Architecture

```
Developer pushes code
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  SCM / Source Control (self-hosted GitLab or Gitea)              │
│  Branch protection: require 2 approvals + CI green               │
└──────────────────────────────────────────────────────────────────┘
        │ merge to main
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gate 1: Build + Unit Tests (cloud CI, ~5 min)                   │
│  • Compile, lint, static analysis                                │
│  • Unit tests                                                    │
│  • Container / artifact build (OCI image or .deb package)        │
│  • Software composition analysis (dependency CVE scan)           │
│  • Artifact signed with code signing key                         │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gate 2: Simulation Testing (cloud, physics simulator, ~30 min)  │
│  • Launch robot in physics simulator (MuJoCo, IsaacSim, PyBullet)│
│  • Run integration test suite: gait, manipulation, safety rules  │
│  • ML model regression: compare predictions vs. golden set       │
│  • Safety validation: force limits, collision avoidance checks   │
│  • Rollback if safety checks fail — block entire pipeline        │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gate 3: Hardware-in-the-Loop (HIL) Lab Robot (on-prem, ~2h)     │
│  • Deploy to 1–3 physical robots in a test cell                  │
│  • Automated physical test suite: pick-and-place, gait patterns  │
│  • Human QA sign-off (safety engineer approves for production)   │
│  • Performance benchmarks: cycle time, accuracy, energy          │
│  Artifact promoted to "production candidate" on pass             │
└──────────────────────────────────────────────────────────────────┘
        │ human approval gate
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gate 4: Canary Rollout (production fleet, ~24h)                 │
│  • Deploy to 1–5% of fleet during a scheduled maintenance window │
│  • Monitor: error rates, joint health, cycle time deviation,     │
│    safety incident count (must be zero)                          │
│  • Automatic promotion if metrics within threshold for 24h       │
│  • Automatic rollback if any safety metric breaches              │
└──────────────────────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────────────┐
│  Gate 5: Progressive Fleet Rollout                               │
│  • 10% → 25% → 50% → 100% with hold periods at each stage       │
│  • Each stage: monitor same metrics as canary                    │
│  • Manufacturing shift awareness: do not deploy during peak shift │
│  • Emergency stop: fleet-wide rollback command within 30s        │
└──────────────────────────────────────────────────────────────────┘
```

---

## Artifact Management

Different components require different artifact stores and deployment mechanisms:

| Component | Artifact Type | Store | Deploy Mechanism |
|---|---|---|---|
| OS packages | .deb / .rpm | Self-hosted APT/YUM repo (Nexus, Pulp) | apt upgrade, rebooted in maintenance window |
| Application stack | OCI container or tarball | Self-hosted Harbor registry | Robot pulls and hot-swaps (if supported) |
| ML model weights | Large binary (GB range) | Object store (MinIO or Azure Blob) | Separate downloader service; atomic swap |
| Firmware | Binary + checksum | Secure firmware server | Vendor toolchain; robot must be idle |
| Configuration | YAML/JSON | Git-based config store (GitOps) | Applied via config sync agent on robot |

**Key design point:** model weights should be versioned separately from control software. A new control software release may be compatible with multiple model versions. Decoupling allows independent rollout cadences and targeted rollbacks.

---

## Delta Updates

Pushing full software images to 1,000 robots over factory WiFi is impractical. Each robot may have a 10–50 GB software stack; pushing 50 TB simultaneously collapses the network.

Solutions:
- **Binary delta updates (bsdiff/casync):** generate a patch between old and new versions; push only the diff
- **Content-addressable storage:** robots identify which blocks have changed by hash; download only new blocks (similar to BitTorrent or OCI layer sharing)
- **Peer-to-peer distribution:** once a few robots have the update, they seed it to peers within the factory network; reduces load on the update server

---

## Fleet State Management

With 1,000 robots at different software versions during a rollout, the system must track:
- Per-robot: current software version, target version, deployment status
- Fleet-level: version distribution histogram, how many at each version
- Safety status: are any robots at a version with a known safety issue?

**Implementation:** a fleet management service (similar to a mobile device management system) that each robot checks in with periodically. The service assigns target versions per robot based on rollout policy and tracks actual state.

```
Fleet Management DB:
  robot_id: figure-01-BMW-042
  current_version: 2.4.1
  target_version: 2.4.2
  status: downloading  # idle | downloading | staging | applying | complete | failed
  last_heartbeat: 2026-06-29T14:32:00Z
  safety_clearance: approved
  deployment_window: 02:00-04:00 UTC (shift gap)
```

---

## Deployment Windows and Manufacturing Awareness

The pipeline must be shift-aware. BMW Spartanburg runs three shifts. Deployments must happen during:
- Shift gap (typically 30 min between shifts)
- Weekend planned maintenance windows
- Not during active part production

The fleet management service integrates with the manufacturing scheduling system (MES API) to:
1. Query current production schedule
2. Select robots not actively in a critical task
3. Schedule deployments in approved windows
4. Abort if a robot is unexpectedly pulled into a task during deployment

---

## Rollback Strategy

| Scenario | Rollback Mechanism | Time |
|---|---|---|
| Failed simulation gate | Block merge; no artifact promoted | Instant |
| Failed HIL gate | Artifact quarantined; previous version stays on lab robots | Minutes |
| Canary production failure | Automatic: fleet management rolls robots back to previous | 5–15 min |
| Post-full-rollout issue | Emergency fleet-wide rollback command | 30 min for full fleet |
| Firmware rollback | Manual physical intervention may be required | Hours; avoid this |

**The firmware problem:** firmware rollback may be impossible or require factory reset. This is why firmware updates have the strictest gate requirements and the longest hold periods. Never automate firmware deployment without explicit human sign-off at each stage.

---

## SLOs for the CI/CD System

The CI/CD pipeline is itself a critical service. Define its reliability:

| SLI | Target |
|---|---|
| Pipeline availability (can developers submit and run) | 99.9% (< 8.7h downtime/year) |
| Simulation gate duration (P95) | < 45 minutes |
| HIL gate scheduling latency (time to get a lab robot) | < 4 hours |
| Fleet rollout completion (full fleet, excluding windows) | < 72 hours |
| Emergency rollback time (canary to zero) | < 15 minutes |

---

## Security Considerations

- **Code signing:** every artifact must be signed with a key held in HSM (Hardware Security Module); robots reject unsigned artifacts
- **Supply chain security:** dependencies scanned with Syft/Grype (SBOM generation + CVE scan) at Gate 1
- **Update channel isolation:** production robots only pull from the production-approved channel; canary robots from a separate channel
- **Air-gap support:** factory networks may have limited internet access; the update server must be on-prem, reachable from factory floor, with cloud sync happening on a controlled schedule

---

## Interview Follow-Ups to Anticipate

- "What if a robot is mid-task when the update needs to apply? How do you interrupt it safely?"
- "How do you handle a robot that has been offline for 3 months and needs to catch up 10 versions?"
- "How do you test that a rollback actually works before you need it in production?"
- "Your canary robot shows an anomaly, but you're not sure if it's the software or the robot hardware. How do you triage?"
- "How would you design the CI system to handle 10 engineering teams pushing changes concurrently?"

---

## Sources
- [[devops/concepts/ci-cd-pipeline]]
- [[devops/concepts/deployment-strategies]]
- [[figure/company-context]]
- [[sre/concepts/slo-sli-sla]]
