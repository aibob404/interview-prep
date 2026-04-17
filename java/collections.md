# Java Collections

## 1. Collections Framework Overview

```
Iterable
└── Collection
    ├── List        → ArrayList, LinkedList, Vector, CopyOnWriteArrayList
    ├── Set         → HashSet, LinkedHashSet, TreeSet, CopyOnWriteArraySet
    └── Queue       → LinkedList, PriorityQueue, ArrayDeque
        └── Deque   → ArrayDeque, LinkedList

Map (separate hierarchy)
    → HashMap, LinkedHashMap, TreeMap, Hashtable, ConcurrentHashMap, EnumMap
```

---

## 2. ArrayList

### Internals
- Backed by an `Object[]` array
- Default initial capacity: **10**
- Growth factor: **1.5× current size** (`newCapacity = oldCapacity + (oldCapacity >> 1)`)

```java
// Resizing cost — O(n) amortized
// When array is full, new array is allocated and all elements copied
// Amortized cost of add() is O(1)

// Pre-size when count is known — avoids resize
List<String> list = new ArrayList<>(expectedSize);
```

### Complexity

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| `get(i)` | O(1) | Random access |
| `add()` (end) | O(1) amortized | Occasional resize O(n) |
| `add(i, e)` (middle) | O(n) | Shift elements |
| `remove(i)` | O(n) | Shift elements |
| `contains()` | O(n) | Linear scan |
| `iterator` | O(1) per step | Fail-fast |

### Fail-Fast Iterator
```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    list.remove(s); // ConcurrentModificationException!
}

// Correct removal during iteration:
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe
}

// Or: removeIf (Java 8+)
list.removeIf(s -> s.equals("b"));
```

---

## 3. LinkedList

- Doubly-linked list implementing both `List` and `Deque`
- Each node: `prev`, `next`, `element` references
- High memory overhead per element (3 pointers + object header ~40 bytes per node)
- Good for: frequent insertions/deletions at known position, use as queue/stack

| Operation | Complexity |
|-----------|-----------|
| `get(i)` | O(n) — traversal |
| `add()` (end/front) | O(1) |
| `add(i, e)` (middle) | O(n) to find + O(1) to link |
| `remove(e)` | O(n) to find + O(1) to unlink |

**In practice:** `ArrayList` usually outperforms `LinkedList` even for insertions in the middle, because array operations are cache-friendly (CPU prefetching) while pointer-chasing is not.

---

## 4. HashMap

### Internals (Java 8+)
```
buckets[]: Node<K,V>[]
  bucket[0] → null
  bucket[1] → Node(key1, val1) → Node(key5, val5) → null   (linked list)
  bucket[2] → TreeNode (red-black tree when bin size > 8)
  ...
  bucket[n] → Node(keyN, valN)
```

### Hash Function
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    // XOR high 16 bits with low 16 bits to spread high bits into low bits
    // Reduces collisions when table size is small
}

// Bucket index:
int index = (table.length - 1) & hash(key); // table.length is always power of 2
```

### Load Factor and Resizing
- Default load factor: **0.75** (75% full triggers resize)
- Resize threshold: `capacity * loadFactor`
- On resize: capacity doubled, all entries **rehashed and redistributed**

```java
new HashMap<>();              // initial capacity 16, load factor 0.75
new HashMap<>(expectedSize);  // set initial capacity to avoid resizes
// To avoid resize: initialCapacity = (int) (expectedSize / 0.75) + 1
```

### Treeification
- When a bucket's linked list exceeds **8 entries** AND table size ≥ 64 → converts to red-black tree (O(log n) per operation)
- When list shrinks to **6 entries** → converts back to linked list (hysteresis prevents oscillation)
- If table size < 64: resize first instead of treeifying

### Null Handling
- One `null` key allowed (always at bucket 0)
- Multiple `null` values allowed

### Pitfalls
```java
// Using mutable objects as keys — changing key after insertion breaks lookup
class User { String name; }
User u = new User("Alice");
map.put(u, 42);
u.name = "Bob"; // hashCode changes → map.get(u) returns null!
// Always use immutable keys (String, Integer, records)

// BigDecimal key pitfall:
map.put(new BigDecimal("1.0"), "a");
map.get(new BigDecimal("1.00")); // null! equals() compares scale too

// Comparing values, not keys — common mistake
map.get(key) == value; // reference comparison for objects!
map.get(key).equals(value); // correct
```

---

## 5. LinkedHashMap

- Maintains **insertion order** (or access order if constructed with `accessOrder=true`)
- Backed by HashMap + doubly-linked list through all entries
- Access-ordered LinkedHashMap = LRU cache skeleton:

```java
// LRU Cache using LinkedHashMap
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder=true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

---

## 6. TreeMap

- Red-black tree — entries kept in **natural order** or by `Comparator`
- O(log n) for get, put, remove
- Additional navigation methods: `floorKey`, `ceilingKey`, `headMap`, `tailMap`, `subMap`

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(5, "five"); map.put(1, "one"); map.put(3, "three");

map.firstKey();                    // 1
map.lastKey();                     // 5
map.floorKey(4);                   // 3 (greatest key ≤ 4)
map.ceilingKey(4);                 // 5 (smallest key ≥ 4)
map.headMap(3);                    // {1=one} (strictly less than 3)
map.subMap(1, true, 3, true);      // {1=one, 3=three}
```

---

## 7. HashSet / LinkedHashSet / TreeSet

All delegate to their Map counterparts (value is a dummy object `PRESENT`):
- `HashSet` → `HashMap` (O(1), unordered)
- `LinkedHashSet` → `LinkedHashMap` (O(1), insertion-ordered)
- `TreeSet` → `TreeMap` (O(log n), sorted)

```java
// Deduplication while preserving order:
Set<String> unique = new LinkedHashSet<>(list);

// Sorted unique values:
Set<Integer> sorted = new TreeSet<>(list);
```

---

## 8. PriorityQueue

- Min-heap by default (smallest element at head)
- O(log n) for `offer`, `poll`; O(1) for `peek`
- Not thread-safe; use `PriorityBlockingQueue` for concurrency

```java
// Natural ordering (min-heap)
PriorityQueue<Integer> minHeap = new PriorityQueue<>();

// Max-heap
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Custom object ordering
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority).reversed()
);

// Top-K elements (keep K largest):
PriorityQueue<Integer> topK = new PriorityQueue<>(k); // min-heap of size k
for (int num : nums) {
    topK.offer(num);
    if (topK.size() > k) topK.poll(); // remove smallest
}
```

---

## 9. ArrayDeque

- Double-ended queue backed by resizable array (circular buffer)
- Preferred over `LinkedList` for stack and queue use
- No null elements allowed
- O(1) amortized for all head/tail operations

```java
ArrayDeque<String> deque = new ArrayDeque<>();

// Use as stack (LIFO)
deque.push("a");   // addFirst
deque.pop();       // removeFirst

// Use as queue (FIFO)
deque.offer("a");  // addLast
deque.poll();      // removeFirst

// Use as deque
deque.offerFirst("front");
deque.offerLast("back");
deque.peekFirst();
deque.peekLast();
```

---

## 10. ConcurrentHashMap

### Java 7: Segment Locking
- Divided into **16 segments** (independent `ReentrantLock`)
- Up to 16 concurrent writes simultaneously

### Java 8+: Node-level CAS + `synchronized`
- No segments — fine-grained locking per bin
- Empty bucket: `CAS` to insert (no lock)
- Non-empty bucket: `synchronized` on head node
- Much better throughput than Java 7

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// Atomic operations — never use get+put pattern
map.computeIfAbsent("key", k -> expensiveCreate());     // create if absent
map.computeIfPresent("key", (k, v) -> v + 1);           // update if present
map.compute("key", (k, v) -> v == null ? 1 : v + 1);   // create or update
map.merge("key", 1, Integer::sum);                       // merge with existing

// WRONG — not atomic
Integer val = map.get("key");
if (val == null) map.put("key", 1); // race condition between get and put

// CORRECT
map.putIfAbsent("key", 1);
// or
map.computeIfAbsent("key", k -> 1);
```

**What ConcurrentHashMap does NOT support:**
- `null` keys or values (throws `NullPointerException`)
- Atomic compound operations across multiple keys
- Consistent iteration over snapshot (iterator may or may not see concurrent updates)

---

## 11. Complexity Reference

| Data Structure | get | add/put | remove | contains/search | Notes |
|----------------|-----|---------|--------|-----------------|-------|
| ArrayList | O(1) | O(1)* | O(n) | O(n) | *amortized |
| LinkedList | O(n) | O(1) | O(n) | O(n) | head/tail O(1) |
| ArrayDeque | O(1) | O(1)* | O(1) | O(n) | head/tail only |
| HashMap | O(1)* | O(1)* | O(1)* | O(1)* | *avg; O(log n) with tree bins |
| LinkedHashMap | O(1)* | O(1)* | O(1)* | O(1)* | ordered |
| TreeMap | O(log n) | O(log n) | O(log n) | O(log n) | sorted |
| HashSet | — | O(1)* | O(1)* | O(1)* | |
| TreeSet | — | O(log n) | O(log n) | O(log n) | sorted |
| PriorityQueue | O(1) peek | O(log n) | O(log n) | O(n) | min-heap |

---

## 12. Choosing the Right Collection

| Scenario | Use |
|----------|-----|
| Fast random access, append | `ArrayList` |
| Frequent insert/delete at front | `ArrayDeque` |
| Stack or queue | `ArrayDeque` |
| Key-value, order doesn't matter | `HashMap` |
| Key-value, insertion order matters | `LinkedHashMap` |
| Key-value, sorted keys | `TreeMap` |
| Unique elements, fast lookup | `HashSet` |
| Sorted unique elements | `TreeSet` |
| Top-K / priority processing | `PriorityQueue` |
| Thread-safe key-value | `ConcurrentHashMap` |
| Thread-safe list, read-heavy | `CopyOnWriteArrayList` |
| Producer-consumer | `ArrayBlockingQueue` / `LinkedBlockingQueue` |
| LRU cache | `LinkedHashMap` with `removeEldestEntry` |
| Enum keys | `EnumMap` (array-backed, very fast) |

---

## 13. Common Interview Questions

| Question | Answer |
|----------|--------|
| How does `HashMap` work internally? | `HashMap` uses an array of buckets. When you `put(key, value)`, it computes `key.hashCode()`, spreads the bits, and maps to a bucket index. If the bucket is empty, a new node is placed there. If occupied, it uses a linked list (chaining). Once a bucket's chain reaches 8 entries and the total size exceeds 64, the chain is converted to a **red-black tree** for O(log n) lookup. The default load factor is 0.75 — when 75% of buckets are occupied, the array doubles in size and all entries are rehashed. |
| What is the difference between `HashMap` and `Hashtable`? | `Hashtable` is a legacy class (Java 1.0) that synchronizes every method with a single lock, making it slow under contention. It also disallows `null` keys or values. `HashMap` is unsynchronized and allows one `null` key and any number of `null` values. For thread-safe use, always prefer `ConcurrentHashMap` over `Hashtable` — it achieves thread safety without a global lock, providing far better concurrency. |
| How does `ConcurrentHashMap` achieve thread safety without a global lock? | In Java 8+, an insert into an empty bucket uses **CAS** (Compare-And-Swap) — a single atomic CPU instruction with no lock. Updates to an occupied bucket use `synchronized` on the **individual bin head node**, so only threads colliding on the same bucket contend. Threads in different buckets run in parallel. The `size()` method uses a `LongAdder`-style striped counter to avoid contention. This design allows near-linear throughput scaling with thread count. |
| What is `LinkedHashMap` and what is it used for? | `LinkedHashMap` extends `HashMap` and maintains a doubly-linked list through all entries, preserving **insertion order** (default) or **access order** (when `accessOrder=true` in the constructor). Its primary use case is implementing an LRU cache: with `accessOrder=true`, every `get()` moves the accessed entry to the tail. Override `removeEldestEntry()` to evict the head (least-recently-used) when the size limit is exceeded. |
| Why should you never use mutable objects as `HashMap` keys? | `HashMap` stores the entry in a bucket determined by the key's `hashCode()` at insertion time. If the key object is mutated after insertion, its `hashCode()` changes, and lookups will compute a different bucket index — the entry becomes unreachable even though it exists in the map. This causes silent data loss and a memory leak (the entry stays in the map but can never be found or removed by key). Always use immutable types (e.g., `String`, `Integer`, records) as map keys. |
| What is the difference between a fail-fast and a fail-safe iterator? | A **fail-fast** iterator (e.g., `HashMap`, `ArrayList`) detects concurrent structural modification via a `modCount` counter. If the map is modified while iterating, it throws `ConcurrentModificationException`. This is a best-effort check, not a guarantee. A **fail-safe** iterator (e.g., `CopyOnWriteArrayList`, `ConcurrentHashMap`'s snapshot iterator) operates on a copy or snapshot of the underlying data structure, so modifications during iteration do not throw — but the iterator may not reflect the latest state. |
| What is the difference between `HashSet` and `TreeSet`? | `HashSet` is backed by a `HashMap`. It offers O(1) average time for add/remove/contains but provides no ordering — iteration order is unpredictable. `TreeSet` is backed by a red-black tree (`TreeMap`). It offers O(log n) for all operations but keeps elements in **sorted order** (natural ordering via `Comparable`, or a custom `Comparator`). Use `HashSet` for performance when order does not matter; use `TreeSet` when you need sorted iteration, range queries (`subSet`, `headSet`, `tailSet`), or `first()`/`last()`. |
| When should you use `EnumMap` instead of `HashMap` with enum keys? | When all keys are of an enum type, `EnumMap` is always preferable. It is backed by a plain array indexed by the enum's ordinal — no hashing, no collision handling, no boxing. Lookup and put are O(1) and extremely fast. Memory usage is minimal: one array slot per enum constant. It also iterates in enum declaration order. The only requirement is that all keys belong to the same enum type, which must be known at construction time. |
| What happens internally when two different keys hash to the same bucket? | A collision: the new entry is appended to the **linked list** at that bucket using `equals()` to confirm it is truly a different key (not a duplicate). If the list grows beyond 8 entries **and** the array has at least 64 buckets, the list is **treeified** into a red-black tree, turning worst-case lookup from O(n) to O(log n). If the array is smaller than 64, the array is resized instead. When entries are removed and the tree shrinks below 6, it is untreeified back to a linked list. |
