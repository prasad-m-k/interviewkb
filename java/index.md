# Core Java Interview Knowledge Base

**Audience:** Senior Software Engineers / Backend Engineers
**Focus:** FAANG-level depth, concurrency, JVM internals, and performance.

---

## 🏗️ Core Resources

| Resource | Description |
|---|---|
| [[java/log]] | **Study Log:** Track progress and action items. |
| [[java/flashcards/core-java-top20]] | **Active Recall:** Top 20 high-frequency cards. |
| [[java/scenarios/thread-safe-system-design]] | **Scenario:** High-throughput cache design. |

---

## 🗺️ Roadmap to Mastery

### 1. Concurrency & Multithreading (The "High-Stakes" Topic)
*   [[java/concepts/synchronization]] — Monitors, intrinsic locks, and `reentrantLock`.
*   **Memory Visibility:** `volatile`, Happens-before relationship.
*   **Thread Pools:** `ExecutorService`, `ForkJoinPool`.
*   **Concurrent Collections:** `ConcurrentHashMap` (segmentation vs. CAS).
*   **Scenario:** [[java/scenarios/thread-safe-system-design]] — Designing a cache.

### 2. JVM Internals & Memory Management
*   [[java/concepts/jvm-internals]] — Memory Model (Heap/Stack) and GC.
*   **Garbage Collection:** G1, ZGC, and tuning flags.
*   **Diagnostics:** Thread dumps, Heap dumps, JFR (Java Flight Recorder).

### 3. Collections Framework (Under the Hood)
*   **HashMap:** Bucket structure, tree-ification (Java 8+), hash collisions.
*   **ArrayList vs LinkedList:** Memory locality and Big-O trade-offs.
*   **Fail-fast vs Fail-safe:** Iterators and `CopyOnWriteArrayList`.

### 4. Language Fundamentals & Java 17+
*   [[java/concepts/streams-lambdas]] — Functional programming, pipelines, and lazy evaluation.
*   **Modern Java:** Sealed Classes, Records, Pattern Matching.
*   **Functional Programming:** Streams API, Lambdas, Optional.
*   **Generics:** Type erasure vs. reification.

---

## 🎯 The "Senior Pivot" for Java Interviews
...
Interviewers at this level don't ask "How do you create a thread?" They ask:
1.  *"How does `ConcurrentHashMap` achieve high throughput without locking the whole map?"*
2.  *"Why would you use a `StampedLock` over a `ReentrantReadWriteLock`?"*
3.  *"Can you explain a scenario where a Garbage Collection pause might cause a cascading failure in a distributed system?"*

---
*Tip: For coding problems, use Java 17+ features where possible to show modern proficiency.*
