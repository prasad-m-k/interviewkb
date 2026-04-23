# Java Memory Model (JMM)

**Topic:** [[java/topics/concurrency]]
**Related:** [[java/concepts/synchronization]], [[java/concepts/concurrency-advanced]]

## What is the JMM?

The **Java Memory Model** specifies how threads interact through memory. Without the JMM, a compiler or CPU could reorder instructions, and a value written in one thread might never be visible to another thread.

The JMM answers: "Under what conditions is a write by thread A guaranteed to be visible to a read by thread B?"

The answer: **Happens-Before**.

---

## The CPU Reordering Problem

Modern CPUs and compilers reorder instructions for performance. This is invisible in single-threaded code but catastrophic in multi-threaded code:

```java
// Thread A
result = compute();    // (1)
flag = true;           // (2)

// Thread B
if (flag) {            // (3)
    use(result);       // (4) may see stale result!
}
```

Without synchronization, the CPU may execute (2) before (1), or Thread B may see `flag = true` but `result` is still 0 due to CPU cache. The JMM defines when these reorderings are forbidden.

---

## Happens-Before Rules

If A **happens-before** B, then the result of A is guaranteed to be visible to B — even across threads.

### The 8 Core Happens-Before Rules

| Rule | Happens-Before relationship |
|---|---|
| **Program Order** | Each action in a thread happens-before every subsequent action in that thread |
| **Monitor Lock** | `unlock()` on a monitor happens-before every subsequent `lock()` on that same monitor |
| **Volatile Write** | A write to a `volatile` field happens-before every subsequent read of that same field |
| **Thread Start** | `Thread.start()` happens-before every action in the started thread |
| **Thread Join** | All actions in a thread happen-before `Thread.join()` returns in another thread |
| **Transitivity** | If A HB B and B HB C, then A HB C |
| **Object Constructor** | All actions in the constructor happen-before `Object.finalize()` |
| **Static Initializer** | Static field initialization happens-before first use of the class |

### Practical Example

```java
// This is thread-safe ONLY because of happens-before:
int x = 0;
volatile boolean ready = false;

// Thread A (writer):
x = 42;          // (1) program order: before (2)
ready = true;    // (2) volatile write: HB every subsequent read of ready

// Thread B (reader):
if (ready) {     // (3) volatile read: sees the write (2)
    // Guaranteed to see x = 42 because:
    // (1) HB (2) [program order]
    // (2) HB (3) [volatile write HB volatile read]
    // (1) HB (3) [transitivity]
    assert x == 42;  // SAFE
}
```

**Why `volatile` isn't enough for `i++`:**
`i++` is `read(i) → increment → write(i)` — three operations. `volatile` guarantees visibility of reads/writes but not atomicity of compound operations. Use `AtomicInteger` or `synchronized`.

---

## Memory Barriers

The JMM is implemented via **memory barriers** (also called memory fences) — CPU instructions that prevent reordering across the barrier.

| Barrier type | Prevents |
|---|---|
| **LoadLoad** | Load1 cannot be reordered after Load2 |
| **StoreStore** | Store1 cannot be reordered after Store2 |
| **LoadStore** | Load cannot be reordered after Store |
| **StoreLoad** | Store cannot be reordered after Load (most expensive — full fence) |

`volatile` write inserts a **StoreStore** barrier before and **StoreLoad** after.
`volatile` read inserts a **LoadLoad** and **LoadStore** barrier after.
`synchronized` block entry/exit insert full memory barriers.

---

## Safe Publication

**Safe publication** means making an object available to other threads in a way that guarantees they see its fully initialized state.

### Unsafe Publication (Broken)

```java
// Class with non-final fields
class Resource {
    int value;
    Resource(int v) { this.value = v; }
}

// Unsafe: another thread may see Resource with value = 0
static Resource instance;

void create() {
    instance = new Resource(42);  // not safely published!
}
```

The JVM can partially construct the object and publish the reference before the constructor completes.

### Safe Publication Mechanisms

```java
// 1. volatile field
static volatile Resource instance;   // volatile guarantees visibility of full state

// 2. static initializer (class loader ensures thread safety)
static final Resource instance = new Resource(42);

// 3. final fields — the "final field" guarantee
class ImmutableResource {
    final int value;
    ImmutableResource(int v) { this.value = v; }
}
// After constructor completes, any thread that sees the reference sees final value = v

// 4. synchronized publication
synchronized void create() { instance = new Resource(42); }
synchronized Resource get() { return instance; }
```

**The `final` field guarantee:** If an object's constructor completes normally, any thread that subsequently reads a reference to that object is guaranteed to see the correctly initialized values of its `final` fields — even without synchronization.

---

## Double-Checked Locking — The Classic Bug and Fix

```java
// BROKEN (Java 5 and earlier, sometimes broken even later)
class Singleton {
    private static Singleton instance;
    public static Singleton getInstance() {
        if (instance == null) {                // check 1: no lock
            synchronized (Singleton.class) {
                if (instance == null) {        // check 2: with lock
                    instance = new Singleton(); // may be partially visible!
                }
            }
        }
        return instance;
    }
}

// FIXED: volatile prevents reordering of constructor steps and reference assignment
class Singleton {
    private static volatile Singleton instance;  // ← volatile!
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// BEST: Initialization-on-demand holder (no volatile, no synchronization, thread-safe)
class Singleton {
    private Singleton() {}
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();  // class loader guarantees
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
```

---

## The `final` Field Freeze

The JMM provides a special guarantee for `final` fields: after a constructor completes, all threads see the final fields' values as set in the constructor — without any explicit synchronization.

```java
// Safe to share without synchronization — only because value is final
class Point {
    final double x, y;
    Point(double x, double y) { this.x = x; this.y = y; }
}
```

**This is why immutable objects are inherently thread-safe.** If all fields are `final` and the object doesn't expose mutable state, no synchronization is needed.

---

## `volatile` vs `synchronized` vs `AtomicXxx`

| | `volatile` | `synchronized` | `AtomicInteger` |
|---|---|---|---|
| Visibility guarantee | Yes | Yes | Yes |
| Atomicity | No | Yes (block) | Yes (operation) |
| Mutual exclusion | No | Yes | No (CAS, non-blocking) |
| Reordering prevention | Partial | Full | CAS includes memory barrier |
| Performance | Fast | Slow (contention) | Very fast (no blocking) |
| Use when | Simple flag, status | Complex atomic operation | Single variable atomic ops |

---

## Common JMM Violations

### 1. Benign Data Races (Not Actually Safe)
```java
// "Benign" race — single boolean, seems OK
boolean done = false;
// Thread A: done = true;
// Thread B: while (!done) {}  // may spin forever — JIT can hoist the read!
// Fix: volatile boolean done = false;
```

### 2. Partially Initialized Object via `this` Escape
```java
class EventSource {
    private final int id;
    EventSource(Registry r) {
        r.register(this);  // ← "this" escapes before constructor completes!
        this.id = 42;
        // Another thread calling id on the registered object might see id = 0
    }
}
```

### 3. Long/Double Non-Atomicity (32-bit JVMs)
On 32-bit JVMs, reads/writes of `long` and `double` are not atomic — the 64-bit value can be written in two 32-bit operations. Use `volatile long` or `AtomicLong`. Not an issue on 64-bit JVMs in practice, but the JMM technically permits it.

---

## Interview Questions

**Q: Can `volatile` replace `synchronized`?**
`volatile` provides visibility and prevents reordering but doesn't provide mutual exclusion (atomicity). `i++` with volatile is still a race condition because read-increment-write is not atomic. Use `AtomicInteger` for single-variable atomicity without a lock; use `synchronized` when the invariant spans multiple fields.

**Q: What is the "happens-before" relationship?**
A guarantee in the JMM that if action A happens-before B, the result of A is visible to B. Key rules: monitor unlock HB subsequent lock; volatile write HB subsequent volatile read; thread start HB started thread's actions; all actions HB thread join.

**Q: What makes double-checked locking broken without volatile?**
Without volatile, the JVM can publish a partially constructed object. Object creation (`new Singleton()`) involves: allocate memory → initialize fields → publish reference. JIT/CPU can reorder to: allocate memory → publish reference → initialize fields. Thread B sees a non-null reference and proceeds to use an incompletely initialized object.

## Sources
- [[java/concepts/synchronization]]
- [[java/concepts/jvm-internals]]
