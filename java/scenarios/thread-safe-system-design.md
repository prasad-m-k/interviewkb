# Scenario: "Designing a High-Throughput Thread-Safe Cache"

**Type:** Concurrency & System Design
**Difficulty:** High
**Frequency:** Common for Senior Backend/Systems roles.

---

## The Question

*"You need to design an in-memory cache for a high-traffic service. The cache will be accessed by hundreds of concurrent threads. Reads are 95% of the traffic, writes are 5%. How do you implement this to maximize throughput while ensuring thread safety?"*

---

## What the Interviewer Is Testing

1.  **Concurrency Knowledge:** Do you know the difference between `synchronized` and `ReadWriteLock`?
2.  **Trade-off Analysis:** Can you explain why a global lock is a bottleneck?
3.  **Modern Java:** Do you know about `StampedLock` (Java 8+)?
4.  **Edge Cases:** How do you handle "Cache Stampede" or "Thundering Herd"?

---

## The "Senior Grade" Answer

### Level 1: The Naive Approach (Junior/Mid)
"I'll use a `HashMap` and wrap all access in a `synchronized` block or use `Hashtable`."
*Critique:* Total serialization. 95% of threads are blocked while one thread reads. This won't scale.

### Level 2: The Practical Approach (Senior)
"I'll use `ConcurrentHashMap`. It uses lock-striping and CAS (Compare-and-Swap) to allow concurrent reads and localized writes."
*Critique:* Good for general use, but if we need custom logic (like complex eviction or multi-key updates), we might need more control.

### Level 3: The Optimized Approach (Staff/Lead)
"Since reads are 95%, I will use **`StampedLock`** with **Optimistic Reading**."

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.StampedLock;

public class HighThroughputCache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final StampedLock lock = new StampedLock();

    public V get(K key) {
        // 1. Try Optimistic Read (No locking!)
        long stamp = lock.tryOptimisticRead();
        V value = map.get(key);

        // 2. Validate the stamp
        if (!lock.validate(stamp)) {
            // 3. If stamp is invalid (a write occurred), upgrade to a read lock
            stamp = lock.readLock();
            try {
                value = map.get(key);
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return value;
    }

    public void put(K key, V value) {
        long stamp = lock.writeLock();
        try {
            map.put(key, value);
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

---

## The "Senior Pivot" Follow-ups

**Q: "What is a Cache Stampede, and how do you prevent it in Java?"**
**Answer:** It happens when a popular key expires and many threads try to reload it from the DB simultaneously.
*Prevention:* Use **`computeIfAbsent`** on a `ConcurrentHashMap`. It ensures that only one thread executes the mapping function while others block for the result.

**Q: "How would you implement an LRU eviction policy in this cache?"**
**Answer:** I would use `LinkedHashMap` and override `removeEldestEntry`. However, `LinkedHashMap` is not thread-safe, so I would need to wrap it in a `ReentrantReadWriteLock` or use a specialized library like Caffeine (which uses a ring-buffer approach for better concurrency).

---

## Key Takeaways for the Interview
- Mention **`StampedLock`** for read-heavy workloads.
- Mention **Lock Striping** as the concept behind `ConcurrentHashMap`.
- Discuss **Memory Locality** if asked about large heap performance.
