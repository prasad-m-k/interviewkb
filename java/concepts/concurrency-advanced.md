# Advanced Concurrency — Thread Pools, CompletableFuture, Virtual Threads

**Topic:** [[java/topics/concurrency]]
**Related:** [[java/concepts/synchronization]], [[java/concepts/java-memory-model]]

## The Thread Pool Hierarchy

```
Executor (interface)
└── ExecutorService (interface: submit, shutdown, awaitTermination)
    ├── ThreadPoolExecutor (the core implementation)
    ├── ScheduledThreadPoolExecutor
    └── ForkJoinPool (work-stealing; used by parallel streams + CompletableFuture)
```

Factory methods via `Executors`:
```java
Executors.newFixedThreadPool(n)        // n threads, unbounded queue
Executors.newCachedThreadPool()        // grows/shrinks dynamically, 60s idle timeout
Executors.newSingleThreadExecutor()    // 1 thread, sequential execution
Executors.newScheduledThreadPool(n)    // for delayed/periodic tasks
```

**Warning:** Prefer creating `ThreadPoolExecutor` directly — factory methods use unbounded queues (`LinkedBlockingQueue`) which can OOM under load.

---

## ThreadPoolExecutor — The Full Constructor

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                               // corePoolSize: always-alive threads
    16,                              // maximumPoolSize: max threads during burst
    60L, TimeUnit.SECONDS,           // keepAliveTime: idle non-core thread lifetime
    new ArrayBlockingQueue<>(1000),  // workQueue: bounded! prevents OOM
    new ThreadFactory() {            // custom thread naming (for thread dumps)
        int n = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "worker-" + n++);
            t.setDaemon(true);       // don't prevent JVM shutdown
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);
```

### Thread Pool Lifecycle
1. A new task arrives.
2. If `currentThreads < corePoolSize` → create new thread (even if idle threads exist).
3. If `corePoolSize ≤ currentThreads` → try to enqueue in `workQueue`.
4. If queue is full → create new thread up to `maximumPoolSize`.
5. If `currentThreads == maximumPoolSize` AND queue is full → apply **RejectedExecutionHandler**.

### Rejection Policies
| Policy | Behavior |
|---|---|
| `AbortPolicy` (default) | Throw `RejectedExecutionException` |
| `CallerRunsPolicy` | The calling thread runs the task (backpressure) |
| `DiscardPolicy` | Silently drop the task |
| `DiscardOldestPolicy` | Drop the oldest queued task, then retry |

**Senior answer:** Use `CallerRunsPolicy` for backpressure — it slows the producer naturally when the pool is saturated.

---

## Future vs CompletableFuture

### `Future` Limitations
```java
ExecutorService pool = Executors.newFixedThreadPool(4);
Future<String> future = pool.submit(() -> fetchData());

// Problems:
future.get();             // BLOCKS the calling thread
future.get(1, SECONDS);   // timeout, but still blocks
// No way to chain futures, no callback, no composition
```

### CompletableFuture — Non-Blocking Async Chains

```java
CompletableFuture
    .supplyAsync(() -> fetchUser(userId), pool)          // runs async
    .thenApplyAsync(user -> fetchOrders(user.id), pool)  // chain: user → orders
    .thenCombineAsync(                                    // parallel: merge two futures
        CompletableFuture.supplyAsync(() -> fetchPrefs(userId), pool),
        (orders, prefs) -> buildResponse(orders, prefs)
    )
    .exceptionally(ex -> fallbackResponse(ex))           // error handling
    .thenAccept(response -> sendToClient(response));      // terminal action
```

### Key CompletableFuture Methods

```java
// Creation
CompletableFuture.supplyAsync(supplier)         // async task with return value
CompletableFuture.runAsync(runnable)             // async task no return value
CompletableFuture.completedFuture(value)         // immediately completed

// Transforming (single future)
.thenApply(fn)            // transform result (sync, same thread)
.thenApplyAsync(fn, pool) // transform result (async, different thread)
.thenCompose(fn)          // flatMap — fn returns another CompletableFuture

// Combining (two futures)
.thenCombine(other, fn)   // both complete → merge results
.thenAcceptBoth(other, fn)// both complete → consume results, no return
.applyToEither(other, fn) // whichever completes first

// Error handling
.exceptionally(fn)        // catch exception, provide fallback
.handle((result, ex) -> ...)  // both success and failure paths

// Multiple futures
CompletableFuture.allOf(f1, f2, f3)   // wait for all
CompletableFuture.anyOf(f1, f2, f3)   // wait for first

// Completion
.join()                   // like get() but throws unchecked exception
.get()                    // throws checked exceptions
.getNow(defaultValue)     // return current value or default if not done
```

### Real Example: Parallel Fetch with Timeout

```java
public Product getProductDetails(String productId) {
    CompletableFuture<ProductInfo> infoFuture =
        CompletableFuture.supplyAsync(() -> productService.getInfo(productId), pool);

    CompletableFuture<List<Review>> reviewFuture =
        CompletableFuture.supplyAsync(() -> reviewService.getReviews(productId), pool);

    CompletableFuture<Price> priceFuture =
        CompletableFuture.supplyAsync(() -> pricingService.getPrice(productId), pool);

    return CompletableFuture.allOf(infoFuture, reviewFuture, priceFuture)
        .orTimeout(2, TimeUnit.SECONDS)        // Java 9+: timeout the whole chain
        .thenApply(v -> new Product(
            infoFuture.join(),
            reviewFuture.join(),
            priceFuture.join()
        ))
        .exceptionally(ex -> Product.unavailable(productId))
        .join();
}
```

---

## Concurrent Collections Deep Dive

### ConcurrentHashMap

**Java 7:** Lock striping — divided into 16 segments, each with its own `ReentrantLock`. Reads don't lock; writes lock only the relevant segment.

**Java 8+:** Completely redesigned. No segments. Uses:
- **CAS (Compare-and-Swap)** for most operations — lock-free
- **synchronized on the first node of each bucket** only when needed (e.g., when a new key hashes to an occupied bucket head)
- Red-Black Tree within heavily loaded buckets (same as HashMap)

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic operations that are NOT possible with synchronized HashMap:
map.putIfAbsent("key", 0);
map.computeIfAbsent("key", k -> expensiveCalculation(k));  // only one thread runs fn per key
map.compute("key", (k, v) -> v == null ? 1 : v + 1);       // atomic increment
map.merge("key", 1, Integer::sum);                           // atomic merge

// Do NOT use: not atomic
if (!map.containsKey("key")) {     // check
    map.put("key", value);         // act ← race condition between check and act!
}
// Use instead:
map.putIfAbsent("key", value);
```

### BlockingQueue — Producer-Consumer

```java
// Bounded blocking queue (backpressure: producer blocks when full)
BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);

// Producer thread
void produce(String item) throws InterruptedException {
    queue.put(item);  // BLOCKS if queue is full
    // non-blocking alternative: queue.offer(item, 100, MILLISECONDS)
}

// Consumer thread
void consume() throws InterruptedException {
    String item = queue.take();  // BLOCKS if queue is empty
    process(item);
}
```

**Implementations:**
| Class | Characteristics |
|---|---|
| `LinkedBlockingQueue` | Optionally bounded; linked nodes |
| `ArrayBlockingQueue` | Bounded; array-backed; fair ordering option |
| `PriorityBlockingQueue` | Unbounded; priority-ordered |
| `DelayQueue` | Elements delayed until `delay()` expires |
| `SynchronousQueue` | No internal capacity; every put must wait for a take |
| `LinkedTransferQueue` | Combines LinkedBlockingQueue + SynchronousQueue |

---

## Atomic Variables and CAS

### Compare-And-Swap (CAS)
```
CAS(address, expectedValue, newValue):
  if (*address == expectedValue:
      *address = newValue
      return true
  else:
      return false  (try again)
```

This is a single CPU instruction (`CMPXCHG` on x86) — the hardware guarantees atomicity.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();                        // CAS-based ++ 
counter.compareAndSet(expected, update);          // explicit CAS

// Lock-free stack (Treiber stack)
AtomicReference<Node> head = new AtomicReference<>(null);
void push(T value) {
    Node newNode = new Node(value);
    Node oldHead;
    do {
        oldHead = head.get();
        newNode.next = oldHead;
    } while (!head.compareAndSet(oldHead, newNode));  // retry if head changed
}
```

**ABA Problem:** CAS checks the value but not whether it was changed and changed back. Fix: `AtomicStampedReference` pairs the value with a version number.

### LongAdder vs AtomicLong

Under high contention:
- `AtomicLong`: all threads spin on the same CAS → high CPU waste
- `LongAdder`: distributes the counter across multiple cells; aggregates on `sum()` — much better throughput

Use `LongAdder` for metrics/counters with many writers and infrequent reads. Use `AtomicLong` when you need the exact current value for every operation.

---

## ForkJoinPool and Work-Stealing

`ForkJoinPool` is designed for **recursive, divide-and-conquer** tasks. Each thread has its own work deque; idle threads steal from the back of busy threads' deques.

```java
ForkJoinPool pool = ForkJoinPool.commonPool(); // shared pool used by parallel streams

class SumTask extends RecursiveTask<Long> {
    private final long[] array;
    private final int start, end;

    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Base case: compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();                     // async submit left
        return right.compute() + left.join();  // compute right inline, wait for left
    }
}

long total = pool.invoke(new SumTask(array, 0, array.length));
```

**`CompletableFuture.supplyAsync()` with no executor** uses `ForkJoinPool.commonPool()`. In production, provide a custom pool to avoid thread starvation from blocking tasks.

---

## Virtual Threads (Java 21 — Project Loom)

**The problem:** Platform threads (OS threads) are expensive — 1MB+ stack, context switches. A server doing I/O-heavy work (DB calls, HTTP) wastes most time blocking. Increasing thread pool size risks OOM.

**The solution:** Virtual threads are lightweight, user-mode threads managed by the JVM. Millions can exist; they mount/unmount from a small pool of carrier (platform) threads.

```java
// Create a virtual thread
Thread vt = Thread.ofVirtual().start(() -> {
    // Blocking I/O is fine — the virtual thread parks, carrier thread is freed
    String result = httpClient.get("https://api.example.com");
    process(result);
});

// ExecutorService backed by virtual threads (one virtual thread per task)
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = IntStream.range(0, 100_000)
        .mapToObj(i -> executor.submit(() -> fetchData(i)))
        .toList();
    // 100,000 concurrent I/O tasks with no thread pool tuning needed
}
```

**When NOT to use virtual threads:**
- CPU-bound work (no benefit — blocked threads are the point)
- Synchronized blocks that call blocking I/O (pinning — the carrier thread is blocked too, defeating the purpose). Use `ReentrantLock` instead.

**Structured Concurrency (Java 21 preview):**
```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<User> user = scope.fork(() -> fetchUser(id));
    Future<List<Order>> orders = scope.fork(() -> fetchOrders(id));
    scope.join();           // wait for both
    scope.throwIfFailed();  // propagate first failure
    return new Profile(user.resultNow(), orders.resultNow());
}  // scope closes: cancels any outstanding tasks
```

---

## Interview Questions

**Q: What is the difference between `submit()` and `execute()` on an `ExecutorService`?**
`execute(Runnable)` returns void — no way to check completion or catch exceptions. `submit(Callable/Runnable)` returns a `Future` — you can check for completion, get the result, or catch exceptions via `future.get()`.

**Q: Why should you always shut down an `ExecutorService`?**
`ExecutorService` threads are non-daemon by default — they prevent JVM shutdown. Always call `shutdown()` (graceful: waits for in-progress tasks) or `shutdownNow()` (interrupts running tasks). Use try-with-resources for virtual thread executors.

**Q: What is the difference between `thenApply` and `thenCompose` in `CompletableFuture`?**
`thenApply(fn)` is like `map` — fn transforms the result to a value. `thenCompose(fn)` is like `flatMap` — fn returns another `CompletableFuture`, and the result is that future's result (not a `CompletableFuture<CompletableFuture<T>>`).

**Q: What is thread pinning in virtual threads?**
When a virtual thread executes a `synchronized` block and calls a blocking operation, the carrier platform thread is pinned (cannot unmount to serve other virtual threads). This negates the benefit of virtual threads. Fix: replace `synchronized` with `ReentrantLock` in code that does blocking I/O.

## Sources
- [[java/concepts/synchronization]]
- [[java/concepts/java-memory-model]]
