# Concept: Java Synchronization & Thread Safety

**Status:** High-Frequency Interview Topic
**Focus:** Intrinsic Locks, Monitors, and `java.util.concurrent`.

---

## 1. How Synchronization Works: The "Monitor"

In Java, every object has an associated **Monitor** (or Intrinsic Lock). When a thread enters a `synchronized` block, it must "acquire" the monitor of that object.

### The `synchronized` keyword
```java
public class Counter {
    private int count = 0;

    // Synchronizing on the "this" instance monitor
    public synchronized void increment() {
        count++;
    }

    // Synchronizing on a private lock object (Recommended for better encapsulation)
    private final Object lock = new Object();
    public void safeIncrement() {
        synchronized(lock) {
            count++;
        }
    }
}
```

### Under the Hood: The "Wait Set" and "Entry Set"
1.  **Entry Set:** Threads waiting for the monitor to be released.
2.  **Owner:** The thread that currently holds the monitor.
3.  **Wait Set:** Threads that called `wait()` on the monitor and are waiting to be `notify()`ed.

---

## 2. Memory Visibility & The `volatile` Keyword

Synchronization isn't just about **atomicity** (locking); it's about **visibility**.

*   **The Problem:** CPU caches may store local copies of variables. Thread A might update a value, but Thread B still sees the old cached version.
*   **The Solution:** `volatile` ensures that reads and writes go directly to **Main Memory**.

```java
public class Flag {
    private volatile boolean running = true; // Visibility guaranteed

    public void stop() { running = false; }
    
    public void work() {
        while (running) {
            // Do work
        }
    }
}
```
*Note: `volatile` does NOT guarantee atomicity (e.g., `i++` is still not thread-safe).*

---

## 3. Beyond Intrinsic Locks: `ReentrantLock`

The `java.util.concurrent.locks` package provides more flexibility than `synchronized`.

| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| **Fairness** | No (First-come-first-served not guaranteed) | Yes (Optional `fair` parameter) |
| **Interruptibility** | No (Thread stays blocked) | Yes (`lockInterruptibly()`) |
| **Timeout** | No | Yes (`tryLock(timeout)`) |
| **Condition Variables** | One per object (`wait/notify`) | Multiple (`Condition` objects) |

### Example: Using `ReentrantLock`
```java
import java.util.concurrent.locks.ReentrantLock;

public class DataService {
    private final ReentrantLock lock = new ReentrantLock();

    public void accessResource() {
        if (lock.tryLock()) { // Non-blocking attempt
            try {
                // Access resource
            } finally {
                lock.unlock(); // Always unlock in finally!
            }
        } else {
            System.out.println("Could not acquire lock, doing something else...");
        }
    }
}
```

---

## 4. The "Senior Pivot" Interview Questions

### Q: "What is Lock Striping?"
**Answer:** Instead of using one lock for an entire data structure, you divide it into buckets (stripes) each with its own lock. This allows multiple threads to access different parts of the structure concurrently.
*Example:* `ConcurrentHashMap` uses this internally (though modern versions use CAS - Compare and Swap).

### Q: "Explain the 'Happens-Before' relationship."
**Answer:** It's a guarantee provided by the Java Memory Model (JMM). If action A "happens-before" action B, then the results of A are guaranteed to be visible to B.
*Key Rules:*
- A `unlock` happens-before every subsequent `lock` on that same monitor.
- A write to a `volatile` field happens-before every subsequent read of that same field.

### Q: "When would you use `ReadWriteLock`?"
**Answer:** When you have many readers but few writers. It allows multiple readers to hold the lock simultaneously as long as no writer is active.

---

## Related Topics
- [[java/concepts/jvm-internals]] — How the stack and heap interact with threads.
- [[java/concepts/collections-deep-dive]] — Specifically `ConcurrentHashMap`.
