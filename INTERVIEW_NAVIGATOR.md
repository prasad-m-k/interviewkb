# FAANG Interview Navigator (2026)

This navigator provides a high-signal checklist based on your **Role**, **Target Company**, and **Time Remaining**.

---

## 🏗️ Step 1: Choose your Path
*Select your role to prioritize the right Knowledge Bases.*

### **Path A: SRE / Production Engineer**
- **Priority KBs:** [[sre/index]], [[k8s/K8S overview]], [[devops/index]], [[dsa/index]]
- **Focus:** Linux Internals, Troubleshooting, NALSD, Practical Scripting.

### **Path B: MLOps Engineer**
- **Priority KBs:** [[mlops/index]], [[ml/index]], [[solution-arch/index]], [[dsa/index]]
- **Focus:** ML Pipelines, Data Drift, Distributed Systems, Algorithms.

### **Path C: Java / Backend Engineer**
- **Priority KBs:** [[java/index]], [[solution-arch/index]], [[dsa/index]], [[devops/index]]
- **Focus:** Concurrency, JVM Internals, Microservices, System Design.

---

## 🗓️ Step 2: The Timeline Checklist

### **Phase 1: Foundation (T-Minus 30 Days)**
*Goal: Master the load-bearing abstractions.*
- [ ] **DSA Patterns:** Master the "Top 10" patterns in [[dsa/DSA overview]].
- [ ] **System Fundamentals:** 
    - **SRE:** Review [[sre/concepts/networking-fundamentals]] and [[sre/concepts/linux-boot-process]].
    - **MLOps:** Review [[ml/concepts/transformers]] and [[mlops/concepts/feature-store]].
    - **Java/Backend:** Review [[java/concepts/synchronization]] and [[java/concepts/jvm-internals]].
- [ ] **Git Mastery:** Review [[git/cheatsheet]] for history cleanup and branching.

### **Phase 2: Depth & Scenarios (T-Minus 14 Days)**
*Goal: Solve complex real-world problems.*
- [ ] **Infrastructure/Architecture:**
    - **SRE/PE:** Run through [[k8s/K8S overview]] troubleshooting scenarios.
    - **MLOps:** Review [[solution-arch/patterns/event-driven-architecture]] for data pipelines.
- [ ] **Problem Bank:** Solve 2 problems per day from the [[dsa/index#Problems]] section.
- [ ] **Diagramming:** Practice drawing [[solution-arch/diagrams/c4-model]] for system design.

### **Phase 3: Company Drill (T-Minus 7 Days)**
*Goal: Align with specific interview cultures.*
- [ ] **Google:** Focus on **NALSD** and **Streaming** in [[dsa/companies/google]].
- [ ] **Meta:** Focus on **Speed Coding** and **Troubleshooting** in [[dsa/companies/meta]].
- [ ] **Apple:** Focus on **Trees** and **Production Quality Code** in [[dsa/companies/apple]].

### **Phase 4: Final Polish (T-Minus 48 Hours)**
*Goal: Speed, recall, and behavior.*
- [ ] **Flashcards:** Run the [[dsa/flashcards/sre-prodeng-top15]] or [[dsa/flashcards/dsa-mlops-top10]].
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
| **Linux/SRE** | [[sre/index]] | SLO/SLI/SLA Scenarios |

---
*Tip: Use `graphify query "<question>"` to trace connections between these topics across the vault.*
