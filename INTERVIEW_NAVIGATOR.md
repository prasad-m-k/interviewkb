# FAANG Interview Navigator (2026)

This navigator provides a high-signal checklist based on your **Role**, **Target Company**, and **Time Remaining**.

---

## 🏗️ Step 1: Choose your Path
*Select your role to prioritize the right Knowledge Bases.*

### **Path A: SRE / Production Engineer**
- **Priority KBs:** [[sre/index]], [[k8s/K8S overview]], [[devops/index]], [[dsa/index]]
- **Focus:** Linux Internals, Troubleshooting, NALSD, Practical Scripting.
- **IaC depth:** [[devops/concepts/infrastructure-as-code]], [[devops/concepts/terraform-state]], [[devops/flashcards/terraform-iac-top15]]

### **Path B: MLOps Engineer**
- **Priority KBs:** [[mlops/index]], [[ml/index]], [[solution-arch/index]], [[dsa/index]]
- **Focus:** ML Pipelines, Data Drift, Distributed Systems, Algorithms.

### **Path D: Google AI/ML Engineer**
- **Priority KBs:** [[ml/index]], [[ml/companies/google-ml]], [[mlops/index]], [[dsa/index]]
- **Focus:** ML Theory + Coding, LLM Fundamentals, ML System Design, Googleyness.
- **Google-specific prep:** [[ml/flashcards/google-ml-top20]], [[ml/scenarios/youtube-recommendations]], [[ml/scenarios/google-ads-ctr]], [[ml/scenarios/llm-service-design]]

### **Path E: Figure AI — Staff SRE (AI Robotics)**
- **Priority KBs:** [[figure/index]], [[sre/index]], [[devops/index]]
- **Focus:** Hybrid cloud/on-prem infrastructure, robot fleet CI/CD, SaaS-to-self-hosted migration, manufacturing SLOs.
- **Start here:** [[figure/company-context]] → [[figure/staff-sre-role]] → [[figure/scenarios/robot-fleet-cicd]]
- **IaC is explicitly required:** [[devops/concepts/terraform-state]], [[devops/concepts/terraform-patterns]], [[devops/concepts/ansible]], [[devops/scenarios/terraform-interview]]

### **Path C: Java / Backend Engineer**
- **Priority KBs:** [[java/index]], [[solution-arch/index]], [[dsa/index]], [[devops/index]]
- **Focus:** Concurrency, JVM Internals, Microservices, System Design.

### **Path F: AI Agents / Agentic AI (Instructor Prep & Interview)**
- **Priority KBs:** [[agents/index]], [[ml/index]], [[ml/concepts/rag]]
- **Focus:** Agent vs. workflow distinction, the agent loop, tool use/MCP, memory & context engineering, ReAct/plan-execute/multi-agent patterns, guardrails, agent evaluation.
- **Start here:** [[agents/Agentic AI overview]] → [[agents/concepts/what-is-an-agent]] → [[agents/concepts/agent-loop]]
- **Applied design drill:** [[agents/scenarios/agent-debugging-playbook]], [[agents/scenarios/research-agent-design]]

### **Path G: Microsoft — Principal SWE, Responsible AI (CoreAI)** #ResponsibleAI
- **Priority KBs:** [[solution-arch/companies/microsoft-coreai-responsible-ai]], [[solution-arch/topics/ai-governance-responsible-ai]], [[mlops/index]], [[dsa/index]]
- **Focus:** Large-scale distributed cloud services (HA/scalability/observability) FIRST, then the two Responsible AI surfaces — product-facing guardrails/content-safety AND SDLC-facing governance of AI-generated requirements/design docs/code — plus ML model lifecycle ownership and cross-org influence without authority.
- **Start here:** [[solution-arch/companies/microsoft-coreai-responsible-ai]] → [[solution-arch/concepts/azure-ai-content-safety]] → [[solution-arch/concepts/responsible-ai-sdlc-governance]]
- **RAI-specific depth:** [[solution-arch/concepts/ai-guardrails-and-safety]], [[solution-arch/topics/ai-governance-responsible-ai]], [[mlops/concepts/model-registry]], [[mlops/concepts/data-drift]]

### **Path H: Microsoft — Senior/Principal SWE, AI Frameworks** #ResponsibleAI
- **Priority KBs:** [[ml/companies/microsoft-ai-frameworks]], [[ml/concepts/distributed-training]], [[ml/scenarios/llm-service-design]], [[dsa/index]]
- **Focus:** GPU performance engineering (roofline model, CUDA/ROCm/Triton, kernel fusion) FIRST — this is a systems-engineering role, not an ML-theory role. Then compilers/runtimes (torch.compile, ONNX Runtime, model onboarding) and benchmarking/regression-detection systems. Hardware-vendor portability (NVIDIA + AMD + Microsoft silicon) is a named, recurring design constraint — volunteer it unprompted.
- **Start here:** [[ml/companies/microsoft-ai-frameworks]] → [[ml/concepts/gpu-performance-engineering]] → [[ml/concepts/ml-compilers-and-runtimes]] → [[ml/concepts/ml-benchmarking-and-regression-detection]]
- **Practice:** [[ml/flashcards/microsoft-ai-frameworks-top15]]

---

## 🗓️ Step 2: The Timeline Checklist

### **Phase 1: Foundation (T-Minus 30 Days)**
*Goal: Master the load-bearing abstractions.*
- [ ] **DSA Patterns:** Master the "Top 10" patterns in [[dsa/DSA overview]].
- [ ] **System Fundamentals:** 
    - **SRE:** Review [[sre/concepts/networking-fundamentals]] and [[sre/concepts/linux-boot-process]].
    - **MLOps:** Review [[ml/concepts/transformers]] and [[mlops/concepts/feature-store]].
    - **Java/Backend:** Review [[java/concepts/synchronization]] and [[java/concepts/jvm-internals]].
    - **Google AI/ML:** Review [[ml/concepts/llm-fundamentals]], [[ml/concepts/distributed-training]], and [[ml/companies/google-ml]].
    - **Figure AI SRE / Any SRE:** Review [[devops/concepts/infrastructure-as-code]] (tool decision matrix) and [[devops/concepts/terraform-state]] (state, locking, split-state).
    - **Microsoft Responsible AI (CoreAI):** Review [[solution-arch/topics/scalability-and-reliability]] (Principal SWE bar comes first) and [[solution-arch/topics/ai-governance-responsible-ai]].
- [ ] **Git Mastery:** Review [[git/cheatsheet]] for history cleanup and branching.

### **Phase 2: Depth & Scenarios (T-Minus 14 Days)**
*Goal: Solve complex real-world problems.*
- [ ] **Infrastructure/Architecture:**
    - **SRE/PE:** Run through [[k8s/K8S overview]] troubleshooting scenarios.
    - **MLOps:** Review [[solution-arch/patterns/event-driven-architecture]] for data pipelines.
    - **IaC (all SRE paths):** Work through [[devops/scenarios/terraform-interview]] — 5 walkthroughs covering new-company design, state recovery, CFN migration, GitOps pipeline, and module refactoring.
- [ ] **Problem Bank:** Solve 2 problems per day from the [[dsa/index#Problems]] section.
- [ ] **Diagramming:** Practice drawing [[c4-model]] for system design.

### **Phase 3: Company Drill (T-Minus 7 Days)**
*Goal: Align with specific interview cultures.*
- [ ] **Google SRE:** Focus on **NALSD** and **Streaming** in [[dsa/companies/google]].
- [ ] **Google AI/ML:** Run through [[ml/scenarios/youtube-recommendations]], [[ml/scenarios/google-ads-ctr]], and [[ml/scenarios/llm-service-design]]. Practice ML coding from [[ml/companies/google-ml#Round 1: ML Coding]].
- [ ] **Figure AI:** Run the [[figure/flashcards/figure-sre-top15]] deck. Review [[devops/concepts/ansible]] (patch-fleet scenario) and [[devops/patterns/terraform-module-design]] (self-hosted infra structure).
- [ ] **Microsoft (CoreAI Responsible AI):** Run through [[solution-arch/companies/microsoft-coreai-responsible-ai#Practice Questions]] — distributed systems, RAI governance, and MLOps lifecycle questions. Drill the two-surface RAI distinction in [[solution-arch/concepts/responsible-ai-sdlc-governance]] until you can state it in one breath.
- [ ] **Microsoft (AI Frameworks):** Run through [[ml/companies/microsoft-ai-frameworks#Practice Questions]] — GPU diagnosis, compiler/onboarding, and benchmarking questions. Drill the roofline model (compute-bound vs memory-bound) as your default FIRST move on any performance question, and rehearse naming hardware-vendor portability (NVIDIA/AMD/Microsoft silicon) unprompted.
- [ ] **Meta:** Focus on **Speed Coding** and **Troubleshooting** in [[dsa/companies/meta]].
- [ ] **Apple:** Focus on **Trees** and **Production Quality Code** in [[dsa/companies/apple]].

### **Phase 4: Final Polish (T-Minus 48 Hours)**
*Goal: Speed, recall, and behavior.*
- [ ] **Flashcards:** Run the [[dsa/flashcards/sre-prodeng-top15]], [[dsa/flashcards/dsa-mlops-top10]], [[ml/flashcards/google-ml-top20]] (Google AI/ML), or [[devops/flashcards/terraform-iac-top15]] (any SRE/DevOps role).
- [ ] **Behavioral:** Prepare "Impact" stories using the Meta/Google values found in their company pages.
- [ ] **Napkin Math:** Review throughput and latency numbers in [[dsa/companies/google#NALSD]].

---

## 🎯 Quick Navigation by Topic

| Topic | Quick Link | Key Artifact |
|---|---|---|
| **Algorithms** | [[dsa/topics/algorithms]] | BFS/DFS Templates |
| **Java / JVM** | [[java/index]] | Synchronization Deep Dive |
| **Kubernetes** | [[k8s/K8S overview]] | Troubleshooting Mindmap |
| **System Design** | [[solution-arch/index]] | Scalability Patterns |
| **Machine Learning** | [[ml/index]] | Model Evaluation |
| **AI Agents / Agentic AI** | [[agents/index]] | Agent Loop + ReAct + Multi-Agent Patterns |
| **Google AI/ML** | [[ml/companies/google-ml]] | ML Coding + LLM Fundamentals |
| **Linux/SRE** | [[sre/index]] | SLO/SLI/SLA Scenarios |
| **Figure AI SRE** | [[figure/index]] | Robot Fleet CI/CD + Self-Hosted Migration |
| **IaC / Terraform** | [[devops/topics/infrastructure-as-code]] | State Management + Module Design |
| **Ansible** | [[devops/concepts/ansible]] | Playbooks, Roles, Idempotency |
| **Microsoft Responsible AI** | [[solution-arch/companies/microsoft-coreai-responsible-ai]] | Content Safety + SDLC Governance #ResponsibleAI |
| **Microsoft AI Frameworks** | [[ml/companies/microsoft-ai-frameworks]] | GPU Perf + Compilers + Benchmarking #ResponsibleAI |

---
*Tip: Use `graphify query "<question>"` to trace connections between these topics across the vault.*
