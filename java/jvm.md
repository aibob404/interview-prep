# JVM Internals

## 1. JVM Architecture

```
┌─────────────────────────────────────────┐
│           Java Source (.java)           │
│               ↓ javac                   │
│           Bytecode (.class)             │
│               ↓                         │
│  ┌──────────────────────────────────┐   │
│  │         Class Loader Subsystem   │   │
│  │  Bootstrap → Platform → App      │   │
│  └──────────────────────────────────┘   │
│               ↓                         │
│  ┌──────────────────────────────────┐   │
│  │      Runtime Data Areas          │   │
│  │  Heap | Stack | Metaspace | ...  │   │
│  └──────────────────────────────────┘   │
│               ↓                         │
│  ┌──────────────────────────────────┐   │
│  │       Execution Engine           │   │
│  │  Interpreter | JIT | GC          │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

---

## 2. Runtime Data Areas

### Heap
- Stores **all object instances** and arrays
- Shared across all threads
- Divided into **generations** (in generational collectors):
  - **Young Generation:** Eden + Survivor S0 + Survivor S1
  - **Old Generation (Tenured):** long-lived objects
- Subject to Garbage Collection

### Stack (Thread Stack)
- **Per-thread**, stores stack frames
- Each frame holds: local variables, operand stack, reference to constant pool
- `StackOverflowError` when stack depth exceeded (deep recursion)
- `-Xss` controls stack size per thread

### Metaspace (Java 8+, replaced PermGen)
- Stores **class metadata**: class definitions, method bytecode, static fields
- Grows dynamically (limited by native memory, not heap)
- `-XX:MaxMetaspaceSize=256m` limits it
- OOM in Metaspace usually means class loader leak

### Other Areas
- **Program Counter (PC) Register:** per-thread, address of current bytecode instruction
- **Native Method Stack:** for JNI (native methods)
- **Code Cache:** compiled native code from JIT

---

## 3. Class Loading

### Class Loader Hierarchy
```
Bootstrap ClassLoader      (JVM built-in, loads java.*, javax.*)
    └── Platform ClassLoader   (formerly Extension, loads JDK modules)
            └── Application ClassLoader  (loads classpath)
                    └── Custom ClassLoaders  (OSGi, Spring, web containers)
```

### Delegation Model
When a class is requested:
1. Check if already loaded (cache)
2. Delegate to parent
3. If parent fails, load itself

This ensures `java.lang.String` is always loaded by Bootstrap — prevents spoofing.

### Class Loading Phases
1. **Loading:** find and read `.class` file → create `Class<?>` object
2. **Linking:**
   - *Verification:* bytecode is valid and safe
   - *Preparation:* allocate memory for static fields, set default values
   - *Resolution:* resolve symbolic references to actual references
3. **Initialization:** run static initializers, `<clinit>()` method

```java
// Triggers class initialization:
// - Creating an instance: new Foo()
// - Accessing static field or method
// - Class.forName("Foo")
// - Subclass initialization triggers superclass init

// Does NOT trigger:
// - Accessing a static final constant (compile-time constant)
// - Array of type: Foo[] arr (only Foo[] class loads, not Foo)
```

### Custom ClassLoader
```java
public class PluginClassLoader extends URLClassLoader {
    public PluginClassLoader(URL[] urls) {
        super(urls, null); // null parent — isolates from app classloader
    }

    // Override to reverse delegation (child-first, used in OSGi/containers)
    @Override
    protected Class<?> loadClass(String name, boolean resolve)
            throws ClassNotFoundException {
        // try self first for plugin classes
        if (name.startsWith("com.plugin.")) {
            return findClass(name);
        }
        return super.loadClass(name, resolve); // delegate for JDK classes
    }
}
```

---

## 4. Garbage Collection

### Object Lifecycle in Heap
```
New object → Eden (Young Gen)
    ↓ Minor GC (stops-the-world, fast)
Survived 1 GC → Survivor S0
    ↓ Minor GC
Survived again → Survivor S1
    ... after N survivals (default 15) ...
Promoted → Old Gen (Tenured)
    ↓ Major/Full GC (slower, larger)
Reclaimed when unreachable
```

### GC Roots (Starting Points for Reachability)
- Active thread stacks (local variables, parameters)
- Static fields of loaded classes
- JNI references
- Active monitor locks

### GC Algorithms

#### Serial GC (`-XX:+UseSerialGC`)
- Single-threaded stop-the-world for both minor and major GC
- For single-core or small heap (<100MB)

#### Parallel GC (`-XX:+UseParallelGC`)
- Multi-threaded stop-the-world
- Maximizes throughput, tolerates longer pauses
- Default before Java 9

#### G1 GC (`-XX:+UseG1GC`) — Default Java 9+
- Heap divided into equal-size **regions** (1MB–32MB)
- Concurrent marking runs alongside application
- Targets pause goals (`-XX:MaxGCPauseMillis=200`)
- Mixed collections: young + some old regions
- Avoids full GC (but can still happen)

```
G1 Heap Layout:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ E │ S │ O │ E │ E │ H │ O │ S │
└───┴───┴───┴───┴───┴───┴───┴───┘
E=Eden, S=Survivor, O=Old, H=Humongous (>50% region size)
```

#### ZGC (`-XX:+UseZGC`) — Java 15+
- Concurrent, almost all phases run with application
- Pause times under 1ms, even for TB-scale heaps
- Colored pointers + load barriers

#### Shenandoah (`-XX:+UseShenandoahGC`)
- Ultra-low pause, concurrent evacuation
- Available in OpenJDK, not Oracle JDK

### GC Comparison

| Collector | Pause | Throughput | Heap | Use case |
|-----------|-------|-----------|------|---------|
| Serial | High | Low | Small | Single-core, embedded |
| Parallel | Medium-High | High | Medium | Batch, throughput-focused |
| G1 | Low-Medium | Medium-High | Large | General purpose, balanced |
| ZGC | <1ms | Slightly lower | TB scale | Latency-sensitive |
| Shenandoah | <1ms | Slightly lower | Medium-Large | Latency-sensitive |

---

## 5. GC Tuning

```bash
# Heap size
-Xms2g -Xmx4g          # Initial and max heap (set equal to avoid resize)

# G1 tuning
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200          # Pause goal (G1 tries to meet this)
-XX:G1HeapRegionSize=16m          # Region size (1–32MB)
-XX:G1ReservePercent=10           # Reserve space to avoid promotion failure
-XX:ConcGCThreads=4               # Concurrent marking threads
-XX:InitiatingHeapOccupancyPercent=45  # Start concurrent cycle at this heap %

# GC Logging
-Xlog:gc*:file=/var/log/gc.log:time,uptime,pid:filecount=5,filesize=20m

# Print GC causes and pauses
-XX:+PrintGCDetails -XX:+PrintGCDateStamps

# Metaspace
-XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=256m
```

### Diagnosing GC Issues
- **Long GC pauses:** check heap sizing, object promotion rates, old gen fragmentation
- **Frequent minor GC:** objects promoted too fast, Eden too small — increase `-Xmn`
- **Full GC:** old gen exhaustion, large objects (humongous allocations in G1)
- **GC overhead limit:** JVM spending >98% time in GC — heap too small, memory leak

Tools: `jstat`, `jconsole`, `VisualVM`, GC log analyzers (GCEasy, GCViewer)

---

## 6. JIT Compilation

### Interpretation vs Compilation
1. **Interpreter:** executes bytecode line-by-line (slow, starts immediately)
2. **C1 Compiler (Client):** quick compilation, light optimization
3. **C2 Compiler (Server):** deep optimization, takes more time

### Tiered Compilation (Default)
```
Interpreted → C1 (profiling) → C2 (optimized native code)
```

JVM profiles method execution counts. After thresholds:
- Methods called frequently → compiled by C1
- Hot methods (10,000+ calls or loops) → re-compiled by C2 with aggressive optimization

### JIT Optimizations
- **Inlining:** replaces method call with method body at call site (reduces overhead)
- **Escape analysis:** if object doesn't escape a method → allocated on stack (no GC pressure)
- **Loop unrolling:** duplicates loop body to reduce branch overhead
- **Dead code elimination:** removes unreachable branches
- **Intrinsics:** replaces known methods with hand-optimized native code (e.g., `Math.sqrt`, `Arrays.copyOf`)

```java
// Escape analysis example:
public int sumPoint(int x, int y) {
    Point p = new Point(x, y); // p doesn't escape
    return p.x + p.y;          // JIT may eliminate the allocation entirely
}
```

### Deoptimization
JIT can **undo** compiled code when assumptions are invalidated:
```java
// JIT inlines method assuming only one implementation exists
// If a new class is loaded that overrides it → deoptimize, re-interpret
```

---

## 7. Memory Leaks in JVM

Common causes:

```java
// 1. Static collections growing unbounded
private static final Map<String, byte[]> CACHE = new HashMap<>();
// Fix: use Caffeine or WeakHashMap

// 2. ThreadLocal not removed in thread pool threads
static ThreadLocal<Connection> CONN = new ThreadLocal<>();
public void doWork() {
    CONN.set(getConnection());
    try { work(); }
    finally { CONN.remove(); } // MUST remove — thread pool reuses threads

// 3. Listener / callback registration without deregistration
eventBus.register(this); // never calls eventBus.unregister(this)

// 4. ClassLoader leaks (frameworks, hot reload)
// Each reload creates new ClassLoader; old one can't be GC'd if a reference
// to any of its classes remains (static field, thread-local, etc.)

// 5. Unclosed resources (streams, connections)
// Fix: try-with-resources
```

### Diagnosing Memory Leaks
```bash
# Heap dump
jmap -dump:format=b,file=heap.hprof <pid>
# Or on OOM:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp

# Analyze with Eclipse MAT, VisualVM, or IntelliJ Profiler
# Look for: dominator tree, retained heap, GC roots of suspect objects
```

---

## 8. JVM Flags Cheat Sheet

```bash
# Memory
-Xms<size>                    # Initial heap
-Xmx<size>                    # Max heap
-Xss<size>                    # Stack size per thread (default ~512k-1m)
-XX:MaxMetaspaceSize=<size>

# GC
-XX:+UseG1GC
-XX:+UseZGC
-XX:MaxGCPauseMillis=200
-XX:+PrintGCDetails
-Xlog:gc*

# Diagnostics
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heap.hprof
-XX:+PrintCompilation              # see JIT compilation decisions
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintInlining

# Performance
-server                            # enable server JIT (default on 64-bit)
-XX:+TieredCompilation             # enable tiered compilation (default)
-XX:ReservedCodeCacheSize=256m     # JIT code cache size
```

---

## 9. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the difference between Heap and Stack memory? | The **heap** is a single shared area where all objects live; it is managed by the garbage collector and grows dynamically up to `-Xmx`. The **stack** is per-thread: each thread has its own stack of frames, one per active method call, holding local variables and operand values. Stack allocation is LIFO and nearly free. Heap memory causes `OutOfMemoryError` when exhausted; stack memory causes `StackOverflowError` on deep recursion (default size ~512 KB–1 MB per thread, configurable via `-Xss`). |
| What is Metaspace and how does it differ from PermGen? | Metaspace (Java 8+) stores class metadata — class structures, method bytecode, and constant pools. It replaced PermGen, which had a fixed upper bound and frequently caused `OutOfMemoryError: PermGen space`. Metaspace lives in native memory (off-heap), grows dynamically, and is only bounded by available system memory or a configured `-XX:MaxMetaspaceSize`. The main risk is classloader leaks: if old classloaders are not GC'd (e.g., after app redeployments), their classes stay in Metaspace indefinitely. |
| What is the difference between minor GC, major GC, and full GC? | **Minor GC** collects only the Young Generation (Eden + Survivor spaces). It is frequent and fast — usually under a few milliseconds — because most objects die young (generational hypothesis). **Major GC** collects the Old Generation and is slower because it is larger. **Full GC** collects everything: Young + Old + Metaspace, causing the longest stop-the-world pause. Full GC in production usually signals a problem — either heap is undersized, there is a memory leak, or GC tuning is needed. |
| How does G1 GC work and when should you use it? | G1 divides the heap into equal-sized **regions** (1–32 MB each) tagged dynamically as Eden, Survivor, or Old. It runs concurrent marking to build a liveness map, then at collection time selects the regions with the highest garbage density ("Garbage First") to meet a user-defined pause target (`-XX:MaxGCPauseMillis`). This makes it predictable on large heaps (4 GB+). Use G1 for general-purpose workloads. Prefer ZGC or Shenandoah when pause times must stay under 1 ms regardless of heap size. |
| What is the parent-delegation model of class loading? | When a `ClassLoader` is asked to load a class it first delegates to its **parent**. Only if the parent cannot find the class does the child try itself. The chain is: application loader → platform loader → bootstrap loader. This ensures core Java classes (e.g., `java.lang.String`) always come from the trusted bootstrap loader, preventing application code from accidentally or maliciously replacing them. OSGi and custom plugin systems intentionally break this model via child-first loading. |
| What is JIT compilation and what is tiered compilation? | The JVM starts by interpreting bytecode. When a method becomes "hot" (called frequently), the JIT compiles it to native code. **Tiered compilation** (default since Java 8) uses two compilers: **C1** (client) compiles quickly with light optimization and collects profiling data; **C2** (server) uses that profiling data to apply heavy optimizations — method inlining, loop unrolling, escape analysis, branch elimination. Methods move through tiers as they accumulate call counts, balancing startup speed against peak throughput. |
| How do you diagnose a Java memory leak in production? | (1) Monitor heap growth via `jstat -gc <pid>` or metrics dashboards — heap that keeps growing after GC is a warning sign. (2) Capture a heap dump with `jmap -dump:format=b,file=heap.hprof <pid>` or enable `-XX:+HeapDumpOnOutOfMemoryError`. (3) Open in Eclipse MAT: examine the **dominator tree** for objects retaining the most heap. (4) Find their GC roots — common culprits are static collections, forgotten event listeners, unclosed database connections, or `ThreadLocal` values not cleaned up in thread pools. |
| What causes `OutOfMemoryError: Metaspace` and how do you fix it? | The JVM ran out of native memory for class metadata. The most common cause is a **classloader leak**: dynamic proxy generation (CGLIB, ByteBuddy), OSGi bundles, or application server hot-redeployment creates new classloaders, but old ones are never garbage-collected because a stale reference holds them. Each leaked classloader keeps all its classes in Metaspace. Fix: set `-XX:MaxMetaspaceSize` to fail fast, profile classloader instances with a heap dump, and find the object holding the stale reference. |
| What is escape analysis and what optimizations does it enable? | Escape analysis determines at JIT compile time whether an object's reference can "escape" its creating method or thread. If it is **method-local** (no external references), the JIT can allocate it on the stack or eliminate it via **scalar replacement** (replacing the object with its individual fields as local variables) — removing GC pressure entirely. If it is **thread-local**, synchronization on it is redundant and can be eliminated (**lock elision**). These optimizations are transparent and highly effective for short-lived builder objects and iterators. |
| What is a `StackOverflowError` and how do you resolve it? | Thrown when a thread's call stack exceeds its allocated size. Every method call pushes a new frame; deep or infinite recursion eventually runs out of stack space. Resolution: (1) find and fix the recursion — add a correct base case or convert to an iterative approach using an explicit `Deque` stack; (2) if deep recursion is intentional (e.g., tree traversal on very deep trees), increase stack size with `-Xss` (e.g., `-Xss4m`), but note this multiplies memory usage by thread count. |
