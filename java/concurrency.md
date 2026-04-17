# Java Concurrency

## 1. Java Memory Model (JMM)

The JMM defines **when writes by one thread become visible to another**. Without synchronization, the JVM, JIT compiler, and CPU are free to reorder instructions for optimization.

### Happens-Before Relationship
A happens-before edge guarantees that all writes before it are visible to all reads after it:
- **Monitor lock:** unlock of a monitor HB all subsequent locks of the same monitor
- **Volatile write:** a write to a volatile variable HB all subsequent reads of that variable
- **Thread start:** `Thread.start()` HB any action in the started thread
- **Thread join:** any action in a thread HB `Thread.join()` returning
- **Transitivity:** if A HB B and B HB C, then A HB C

### Visibility Without Synchronization
```java
// BUG: JIT may hoist the read of 'running' out of the loop into a register
private boolean running = true;

public void stop() { running = false; }

public void run() {
    while (running) { /* may never stop */ }
}

// FIX: volatile guarantees every read goes to main memory
private volatile boolean running = true;
```

### Reordering Example
```java
int x = 0, y = 0; // shared state
boolean flag = false;

// Thread 1:
x = 42;
flag = true; // reordering allowed — may execute before x = 42

// Thread 2:
if (flag) {
    System.out.println(x); // may print 0 if reordering occurred!
}

// Fix: make flag volatile — volatile write creates a happens-before with volatile read
private volatile boolean flag = false;
```

---

## 2. `volatile`

`volatile` provides two guarantees:
1. **Visibility:** write immediately flushed to main memory; read always from main memory
2. **Ordering:** no reordering of reads/writes across a volatile access

`volatile` does **not** provide atomicity for compound operations:
```java
private volatile long counter = 0;

// NOT atomic — read-modify-write is three operations:
counter++; // read counter, increment, write counter — race condition!

// Fix: AtomicLong
private final AtomicLong counter = new AtomicLong(0);
counter.incrementAndGet(); // atomic CAS
```

**When to use `volatile`:**
- Simple flags: `running`, `initialized`, `shutdown`
- Singleton double-checked locking pattern
- Publishing immutable objects

---

## 3. `synchronized`

Guarantees both **visibility** and **atomicity**. Every object has an intrinsic monitor lock.

```java
public class Counter {
    private int count = 0;

    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

// synchronized block — finer granularity
private final Object lock = new Object();

public void doWork() {
    synchronized (lock) {
        // critical section
    }
    // other work outside lock — reduces contention
}
```

**Gotchas:**
- `synchronized` on `this` is public — external code can also lock on the same monitor
- Prefer a private lock object to prevent external interference
- `synchronized static` method locks on the class object, not an instance

---

## 4. `ReentrantLock`

More flexible than `synchronized`:
```java
private final ReentrantLock lock = new ReentrantLock();

public void safeOperation() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock(); // always release in finally
    }
}

// Try-lock — avoids blocking
public boolean tryOperation() {
    if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
        try {
            // critical section
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false; // could not acquire lock
}
```

### ReentrantReadWriteLock
Allows multiple concurrent readers, exclusive writer:
```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();
private Map<String, String> cache = new HashMap<>();

public String get(String key) {
    readLock.lock();
    try { return cache.get(key); }
    finally { readLock.unlock(); }
}

public void put(String key, String value) {
    writeLock.lock();
    try { cache.put(key, value); }
    finally { writeLock.unlock(); }
}
```

### StampedLock (Java 8+)
Supports **optimistic reads** — try to read without locking, validate afterward:
```java
private final StampedLock sl = new StampedLock();
private double x, y;

public double distanceFromOrigin() {
    long stamp = sl.tryOptimisticRead();
    double curX = x, curY = y; // read without lock

    if (!sl.validate(stamp)) { // another thread wrote — retry with read lock
        stamp = sl.readLock();
        try { curX = x; curY = y; }
        finally { sl.unlockRead(stamp); }
    }
    return Math.sqrt(curX * curX + curY * curY);
}
```

---

## 5. Atomic Classes

Based on **Compare-And-Swap (CAS)** — a CPU-level atomic instruction:
```java
// CAS pseudocode: atomically do:
//   if (current == expected) { current = update; return true; }
//   else { return false; }

AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();              // atomic i++
counter.compareAndSet(expected, newVal); // optimistic update
counter.getAndUpdate(x -> x * 2);       // atomic compute

// AtomicReference for object references
AtomicReference<User> userRef = new AtomicReference<>(initialUser);
userRef.compareAndSet(oldUser, newUser);

// LongAdder — better throughput than AtomicLong under high contention
// Uses striped counters, summed on read
LongAdder adder = new LongAdder();
adder.increment(); // less contention than AtomicLong
adder.sum();
```

**AtomicLong vs LongAdder:**
- `AtomicLong`: exact value at all times, better for low contention + when exact real-time value needed
- `LongAdder`: higher throughput under contention, sum() is approximate during concurrent updates

---

## 6. Thread Lifecycle

```
NEW → RUNNABLE → (BLOCKED | WAITING | TIMED_WAITING) → TERMINATED
```

- **NEW:** created but `start()` not called
- **RUNNABLE:** running or ready to run
- **BLOCKED:** waiting to acquire a monitor lock
- **WAITING:** `Object.wait()`, `Thread.join()`, `LockSupport.park()`
- **TIMED_WAITING:** `Thread.sleep(n)`, `Object.wait(n)`, `Thread.join(n)`
- **TERMINATED:** finished execution

```java
// Interruption protocol
public void run() {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            process();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt(); // restore interrupted status
            break; // or let the method return
        }
    }
}
```

---

## 7. ExecutorService and Thread Pools

### ThreadPoolExecutor Parameters
```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                           // corePoolSize: threads kept alive even if idle
    8,                           // maximumPoolSize: max threads under load
    60L, TimeUnit.SECONDS,       // keepAliveTime: idle threads above core are terminated
    new LinkedBlockingQueue<>(100), // workQueue: holds pending tasks
    new ThreadFactory() { ... }, // optional: name threads, set daemon flag
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy
);
```

**Rejection Policies:**
| Policy | Behavior |
|--------|----------|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | Task runs in the calling thread (natural backpressure) |
| `DiscardPolicy` | Silently drops the task |
| `DiscardOldestPolicy` | Drops oldest queued task, retries submission |

**Factory methods:**
```java
Executors.newFixedThreadPool(n);       // fixed pool, unbounded queue — OOM risk!
Executors.newCachedThreadPool();       // grows unbounded — thread explosion risk!
Executors.newSingleThreadExecutor();  // single thread, ordered execution
Executors.newScheduledThreadPool(n);  // for scheduled/periodic tasks
Executors.newVirtualThreadPerTaskExecutor(); // Java 21 — virtual threads
```

> Always prefer explicit `ThreadPoolExecutor` construction in production — factory methods hide dangerous defaults (unbounded queue or unbounded thread creation).

### Proper Shutdown
```java
executor.shutdown(); // accept no new tasks
try {
    if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
        executor.shutdownNow(); // cancel running tasks
        if (!executor.awaitTermination(60, TimeUnit.SECONDS))
            log.error("Pool did not terminate");
    }
} catch (InterruptedException e) {
    executor.shutdownNow();
    Thread.currentThread().interrupt();
}
```

---

## 8. CompletableFuture

Represents a future value that can be composed:

```java
// Chain of async operations
CompletableFuture<OrderConfirmation> result = CompletableFuture
    .supplyAsync(() -> userService.findById(userId), executor)      // async fetch
    .thenApplyAsync(user -> orderService.createOrder(user, cart))   // async transform
    .thenComposeAsync(order -> paymentService.charge(order))        // async flat-map
    .thenApply(payment -> new OrderConfirmation(payment.orderId())); // sync transform

// Combine two futures
CompletableFuture<User> userFuture = fetchUser(userId);
CompletableFuture<Address> addrFuture = fetchAddress(userId);

CompletableFuture<Profile> profile = userFuture.thenCombine(
    addrFuture,
    (user, addr) -> new Profile(user, addr)
);

// Wait for all
CompletableFuture.allOf(f1, f2, f3).thenRun(() -> System.out.println("all done"));

// Wait for first
CompletableFuture.anyOf(f1, f2, f3).thenAccept(result -> System.out.println(result));

// Error handling
CompletableFuture<String> safe = future
    .exceptionally(ex -> "fallback")   // recover from error
    .handle((result, ex) -> {          // handle both success and error
        if (ex != null) return "error: " + ex.getMessage();
        return result;
    });

// Timeout (Java 9+)
future.orTimeout(5, TimeUnit.SECONDS)
      .completeOnTimeout("default", 5, TimeUnit.SECONDS);
```

**Common pitfall:** by default `thenApply`/`thenRun` run on the thread that completed the previous stage. Use `thenApplyAsync` to ensure execution on the executor pool.

---

## 9. Concurrent Collections

```java
// ConcurrentHashMap — segmented locking (Java 7), CAS + synchronized per bin (Java 8+)
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.computeIfAbsent(key, k -> expensiveCompute(k)); // atomic
map.merge(key, 1, Integer::sum); // atomic increment

// CopyOnWriteArrayList — creates new array on every write
// Good for: read-heavy, rarely modified lists (event listener lists)
// Bad for: frequent writes — O(n) per write
CopyOnWriteArrayList<Listener> listeners = new CopyOnWriteArrayList<>();

// BlockingQueue — producer-consumer pattern
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);
queue.put(task);    // blocks if full
queue.take();       // blocks if empty
queue.offer(task, 1, TimeUnit.SECONDS); // timeout variant

// LinkedBlockingDeque — double-ended, work-stealing
// ArrayBlockingQueue — bounded, backed by array
// PriorityBlockingQueue — priority-ordered, unbounded
// SynchronousQueue — no capacity, each put must pair with a take
```

---

## 10. Problems: Deadlock, Livelock, Starvation

### Deadlock
```java
// Thread 1: lock A, then B
// Thread 2: lock B, then A → circular wait
// Prevention: always acquire locks in the same order

// Detect: jstack, VisualVM, ThreadMXBean
ThreadMXBean bean = ManagementFactory.getThreadMXBean();
long[] deadlocked = bean.findDeadlockedThreads();
```

**Conditions for deadlock (all four must hold):**
1. Mutual exclusion
2. Hold and wait
3. No preemption
4. Circular wait

**Prevention:** eliminate circular wait by imposing a lock ordering.

### Livelock
Threads keep responding to each other and changing state, but no progress is made. Example: two threads each yield to the other indefinitely.

### Starvation
Low-priority thread never gets CPU time because high-priority threads always preempt it. Fix: fair locks (`new ReentrantLock(true)`).

---

## 11. Virtual Threads (Java 21)

Virtual threads are **JVM-managed lightweight threads** — not mapped 1:1 to OS threads. The JVM can park a virtual thread when it blocks on I/O without blocking the underlying platform thread.

```java
// Create and start
Thread.ofVirtual().start(() -> {
    System.out.println("virtual: " + Thread.currentThread().isVirtual()); // true
});

// Factory for ExecutorService
try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        exec.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1)); // parks, not blocks
            return "done";
        });
    }
} // waits for all tasks — try-with-resources shuts down

// Spring Boot 3.2+: enable globally
spring.threads.virtual.enabled=true
```

**What virtual threads fix:**
- Thread-per-request model becomes viable for millions of concurrent connections
- No need for reactive programming just for I/O concurrency

**What virtual threads don't fix:**
- CPU-bound work — still limited by available cores
- `synchronized` blocks pin the virtual thread to a platform thread (use `ReentrantLock` instead)
- Thread-locals and `ThreadLocal` are still per-virtual-thread — memory use can be high

**Pinning (the main pitfall):**
```java
// BAD: synchronized inside virtual thread → pinned to platform thread
synchronized (lock) {
    socketRead(); // blocks the platform thread, defeats the purpose
}

// GOOD: use ReentrantLock — allows parking
lock.lock();
try { socketRead(); }
finally { lock.unlock(); }
```

---

## 12. Fork/Join Framework

Designed for **recursive divide-and-conquer** work, using **work-stealing**:

```java
public class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] array;
    private final int start, end;

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left  = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork(); // async
        return right.compute() + left.join(); // right runs in current thread
    }
}

ForkJoinPool pool = new ForkJoinPool();
Long total = pool.invoke(new SumTask(array, 0, array.length));
```

---

## 13. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the Java Memory Model and why does it matter? | The JMM defines rules about when writes made by one thread become visible to other threads. Without it, the compiler and CPU are free to reorder instructions for performance. The key concept is **happens-before**: if action A happens-before action B, then A's writes are visible to B. Happens-before is established by: a `volatile` write/read, acquiring/releasing a `synchronized` lock, `Thread.start()`, and `Thread.join()`. Without these, threads may see stale cached values. |
| What is the difference between `volatile` and `synchronized`? | `volatile` guarantees **visibility** only: reads always go to main memory, writes are immediately flushed. It does not provide atomicity. `synchronized` guarantees both **visibility and atomicity**: only one thread executes the block at a time, and all changes are visible to the next thread that acquires the same lock. Use `volatile` for simple flags or state variables; use `synchronized` (or `AtomicXxx`) when the operation involves read-modify-write sequences. |
| What is a race condition and how do you prevent it? | A race condition occurs when the correctness of a program depends on the relative timing of thread execution. The classic example is two threads doing `count++` simultaneously — both read the same value, both increment, both write back, and one increment is lost. Prevention: use `synchronized` blocks, `ReentrantLock`, `AtomicInteger` for counters, or eliminate shared mutable state entirely (e.g., using immutable objects or thread-local variables). |
| How does `ConcurrentHashMap` achieve thread safety without a global lock? | In Java 8+, `ConcurrentHashMap` uses CAS (Compare-And-Swap) for inserting into empty buckets and `synchronized` on the individual bucket (bin) head node for updates. This means only threads competing on the same bucket contend — threads accessing different buckets run in parallel. There is no global lock. `size()` is tracked via `LongAdder`-like cells. This approach gives near-linear throughput scaling with thread count. |
| What are virtual threads and what problem do they solve? | Virtual threads (Java 21, Project Loom) are lightweight threads managed by the JVM, not the OS. A JVM can host millions of them. They solve the platform-thread-per-request model bottleneck: when a virtual thread blocks on I/O, it is unmounted from its carrier OS thread, which is then free to run another virtual thread. This makes blocking code scale like async code without callback complexity. They are not beneficial for CPU-bound work, and `synchronized` blocks that hold locks during I/O can cause pinning (the carrier OS thread blocks too). |
| What is the happens-before relationship? | Happens-before is a guarantee that memory writes in action A are visible to action B. It is established by: writing then reading a `volatile` variable; releasing then acquiring the same `synchronized` lock; `Thread.start()` (parent's actions before start are visible in the new thread); `Thread.join()` (the joined thread's actions are visible to the joining thread after join returns). Without a happens-before edge, a thread may see a stale value from its local CPU cache. |
| What is `ThreadLocal` and when is it dangerous in thread pools? | `ThreadLocal` provides each thread with its own independent copy of a variable — no synchronization needed since no sharing occurs. Danger in thread pools: pool threads are reused across requests. If a thread sets a `ThreadLocal` value while handling request A and does not call `remove()` when done, the next request B handled by the same thread will inherit A's value. This causes data leakage between requests. Always call `threadLocal.remove()` in a `finally` block, or use frameworks that do it automatically (e.g., Spring's `TransactionSynchronizationManager`). |
| What is the difference between `CompletableFuture.thenApply` and `thenApplyAsync`? | `thenApply(fn)` runs the function on the thread that completed the previous stage — typically the thread that called `complete()` or the calling thread if the future is already done. `thenApplyAsync(fn)` schedules the function on a thread pool (ForkJoinPool.commonPool() by default, or a provided executor). Use `thenApplyAsync` when the transformation is slow or blocking so you avoid tying up a thread that may be needed elsewhere. |
| What are the four conditions for deadlock, and how do you prevent it? | Deadlock requires all four: (1) **Mutual exclusion** — resources are non-shareable; (2) **Hold and wait** — a thread holds one resource while waiting for another; (3) **No preemption** — resources cannot be forcibly taken; (4) **Circular wait** — a cycle of threads each waiting for the next. Prevention strategies: impose a global lock ordering so all threads acquire locks in the same sequence; use `tryLock(timeout)` to back off and retry; use higher-level constructs (`java.util.concurrent`) that avoid explicit locking. |
| What is work-stealing in `ForkJoinPool`? | Each worker thread maintains its own double-ended task queue (deque). When a thread finishes its own tasks, instead of blocking it **steals** tasks from the tail of another busy thread's deque. The owner pushes and pops from the head; stealers take from the tail — minimizing contention. This keeps all CPUs busy during divide-and-conquer workloads where subtasks are unequal in size. `ForkJoinPool` is used by parallel streams and `CompletableFuture`'s default executor. |
| What is `ReentrantLock` and when should you prefer it over `synchronized`? | `ReentrantLock` is an explicit lock with the same mutual-exclusion semantics as `synchronized` but with extra capabilities: `tryLock(timeout)` to avoid indefinite blocking; `lockInterruptibly()` to allow a waiting thread to be interrupted; `Condition` objects for multiple wait sets per lock; and fairness mode (though fairness reduces throughput). Use `synchronized` by default for simplicity. Use `ReentrantLock` when you need timed/interruptible locking, need to poll, or need multiple conditions. |
