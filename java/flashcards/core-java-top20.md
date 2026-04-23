# Java Interview Flashcards

**Format:** Front / Back
**Goal:** Rapid recall of high-signal Java concepts.

---

## 🏗️ Core Language & JVM

**Q: Does Java use Pass-by-Value or Pass-by-Reference?**
**A:** Java is strictly **Pass-by-Value**. For objects, the "value" passed is the reference (address). Changing the object state works, but reassigning the reference inside a method won't affect the caller.

**Q: What is the difference between `final`, `finally`, and `finalize`?**
**A:** 
- `final`: Modifier (class can't be extended, method can't be overridden, variable is constant).
- `finally`: Block used in try-catch to ensure execution of cleanup code.
- `finalize`: Method called by GC before object destruction (Deprecated in Java 9+; use `Cleaner` or `PhantomReference`).

**Q: What is "Type Erasure" in Java Generics?**
**A:** The process where the compiler removes all generic type information at compile-time and replaces it with raw types (usually `Object`). This ensures backward compatibility with older Java versions.

---

## 🧵 Concurrency

**Q: What is the "Diamond Problem" in Java and how is it solved?**
**A:** It occurs with multiple inheritance of interfaces having `default` methods with the same signature. Java solves this by requiring the implementing class to explicitly override the conflicting method.

**Q: Difference between `Runnable` and `Callable`?**
**A:**
- `Runnable`: `run()` returns `void`, cannot throw checked exceptions.
- `Callable`: `call()` returns a `V` (via `Future`), can throw checked exceptions.

**Q: What is a "ThreadLocal" variable?**
**A:** A variable that is local to a specific thread. Each thread has its own independently initialized copy. Common use: Database connections or User sessions in web apps.

---

## 📦 Collections

**Q: Why does `HashMap` require the `hashCode()` and `equals()` contract?**
**A:** `hashCode()` determines the bucket index. `equals()` distinguishes between objects if multiple objects end up in the same bucket (collision). If you override one, you **must** override the other.

**Q: What happens if a `HashMap` bucket exceeds 8 elements (Java 8+)?**
**A:** The linked list is converted into a **Balanced Tree (Red-Black Tree)** to improve search time from O(n) to O(log n). This is called "Tree-ification."

---

## 🧹 Garbage Collection

**Q: What is a "Memory Leak" in Java?**
**A:** When objects are no longer needed but are still referenced by a live object (e.g., a static Map or an unclosed resource), preventing the GC from reclaiming them.

**Q: What is the "G1 Garbage Collector" goal?**
**A:** To provide high throughput and predictable pause times (latency) for applications with large heaps.
