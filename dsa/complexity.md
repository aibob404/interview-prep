# Complexity Analysis

## 1. Big O Notation

Big O describes the **upper bound** of growth rate — how runtime or space scales as input size grows.

### Common Complexities (fast → slow)
| Notation | Name | Example |
|----------|------|---------|
| O(1) | Constant | HashMap get, array access |
| O(log n) | Logarithmic | Binary search, balanced BST |
| O(n) | Linear | Linear scan, single loop |
| O(n log n) | Linearithmic | Merge sort, heap sort, most sorting |
| O(n²) | Quadratic | Nested loops, bubble sort |
| O(n³) | Cubic | Triple nested loops |
| O(2ⁿ) | Exponential | Recursive Fibonacci without memo, all subsets |
| O(n!) | Factorial | All permutations |

### Drop Constants and Non-Dominant Terms
```
O(2n)    → O(n)
O(n + n²) → O(n²)
O(n + 500) → O(n)
```

---

## 2. Amortized Analysis

Some operations are occasionally expensive but cheap on average.

**ArrayList.add():**
- Most adds: O(1) — just write to next slot
- Occasional resize: O(n) — copy entire array
- Amortized over N adds: O(1) per add

**Intuition:** spreading the cost of the rare expensive operation across many cheap ones.

---

## 3. Space Complexity

Include: recursion call stack, auxiliary data structures, output.

```java
// O(1) space — modifies in-place
void reverseArray(int[] arr) {
    int left = 0, right = arr.length - 1;
    while (left < right) { ... swap ... }
}

// O(n) space — recursion stack
int fib(int n) {
    if (n <= 1) return n;
    return fib(n-1) + fib(n-2); // stack depth = n
}

// O(log n) space — binary search recursion
int search(int[] arr, int target, int left, int right) {
    // stack depth = log n
}
```

---

## 4. Data Structures Complexity Reference

### Linear Structures
| Operation | Array | ArrayList | LinkedList | ArrayDeque |
|-----------|-------|-----------|------------|------------|
| Access by index | O(1) | O(1) | O(n) | O(1) |
| Add (end) | — | O(1)* | O(1) | O(1)* |
| Add (front) | — | O(n) | O(1) | O(1)* |
| Add (middle) | — | O(n) | O(n) | O(n) |
| Remove (end) | — | O(1)* | O(1) | O(1)* |
| Remove (front) | — | O(n) | O(1) | O(1)* |
| Search | O(n) | O(n) | O(n) | O(n) |

*amortized

### Map / Set
| Operation | HashMap | TreeMap | LinkedHashMap |
|-----------|---------|---------|---------------|
| get / put / remove | O(1) avg | O(log n) | O(1) avg |
| Contains key | O(1) avg | O(log n) | O(1) avg |
| Iteration | O(n) | O(n) sorted | O(n) insertion-ordered |
| Min/Max key | O(n) | O(log n) | O(n) |
| Range query | No | Yes, O(log n) | No |

### Trees and Heaps
| Structure | Search | Insert | Delete | Notes |
|-----------|--------|--------|--------|-------|
| BST (unbalanced) | O(n) | O(n) | O(n) | Degenerates on sorted input |
| BST (balanced / TreeMap) | O(log n) | O(log n) | O(log n) | Red-black tree |
| Heap / PriorityQueue | O(1) peek | O(log n) | O(log n) | Only min/max efficiently |
| Trie | O(m) | O(m) | O(m) | m = key length; fast prefix search |

---

## 5. Sorting Algorithms

| Algorithm | Best | Average | Worst | Space | Stable | Notes |
|-----------|------|---------|-------|-------|--------|-------|
| Bubble Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Only for learning |
| Selection Sort | O(n²) | O(n²) | O(n²) | O(1) | No | |
| Insertion Sort | O(n) | O(n²) | O(n²) | O(1) | Yes | Good for small/nearly-sorted |
| Merge Sort | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes | Guaranteed performance |
| Quick Sort | O(n log n) | O(n log n) | O(n²) | O(log n) | No | Best in practice, cache-friendly |
| Heap Sort | O(n log n) | O(n log n) | O(n log n) | O(1) | No | Good worst case, poor cache |
| Counting Sort | O(n+k) | O(n+k) | O(n+k) | O(k) | Yes | Only integers, limited range |
| Radix Sort | O(nk) | O(nk) | O(nk) | O(n+k) | Yes | For integers or fixed-length strings |

**Java's `Arrays.sort()`:**
- Primitives: Dual-Pivot Quicksort — O(n log n) average, not stable
- Objects: TimSort (Merge sort + Insertion sort) — O(n log n) worst, stable

---

## 6. Graph Algorithms

| Algorithm | Time | Space | Use case |
|-----------|------|-------|---------|
| BFS | O(V + E) | O(V) | Shortest path (unweighted) |
| DFS | O(V + E) | O(V) | Traversal, cycle detection, topological sort |
| Dijkstra | O((V + E) log V) | O(V) | Shortest path (non-negative weights) |
| Bellman-Ford | O(VE) | O(V) | Shortest path (negative weights) |
| Floyd-Warshall | O(V³) | O(V²) | All-pairs shortest path |
| Kruskal | O(E log E) | O(V) | Minimum spanning tree |
| Prim | O(E log V) | O(V) | Minimum spanning tree |
| Topological Sort | O(V + E) | O(V) | Dependency ordering |

---

## 7. Recognizing Complexity

```java
// O(1)
return arr[0];
map.get(key);

// O(log n)
while (n > 1) { n /= 2; ops++; }  // halving: log₂(n) iterations
// Binary search, balanced BST operations

// O(n)
for (int i = 0; i < n; i++) { ... }  // single loop

// O(n log n)
// Any sorting
// Dividing in half AND doing O(n) work: merge sort
mergeSort(arr);  // T(n) = 2T(n/2) + O(n) → O(n log n) by Master Theorem

// O(n²)
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++) { ... }  // nested loops

// O(2ⁿ)
// Recursive solution with two branches, no memoization
int fib(int n) { return fib(n-1) + fib(n-2); }

// O(n!)
// Generating all permutations
```

---

## 8. Space Complexity of Recursive Algorithms

The call stack uses space:

```java
// Recursion depth = space used

// DFS on tree of depth h: O(h) space
//   Balanced tree: O(log n)
//   Skewed tree: O(n)

// Merge sort: O(log n) stack + O(n) merge arrays = O(n)

// Backtracking: O(depth) for stack + O(n) for current path
```

---

## 9. Common Trade-offs

| Trade-off | Approach |
|-----------|---------|
| Time vs Space | Memoize/cache results to avoid recomputation |
| Simple vs Fast | ArrayList O(n) insert is simpler; LinkedList O(1) is faster for front |
| Memory vs Correctness | CopyOnWrite is safe but uses 2× memory on write |
| Exact vs Approximate | HyperLogLog for cardinality: O(1) space, ~2% error vs O(n) exact |
| Sort first | O(n log n) upfront often simplifies O(n) algorithm (beats O(n²) brute force) |

---

## 10. Complexity Cheat Sheet for Interviews

```
If you see:           Think:
n ≤ 10                O(n!) OK — generate all permutations
n ≤ 20                O(2ⁿ) OK — bitmask DP, all subsets
n ≤ 100               O(n³) OK — Floyd-Warshall, matrix chain
n ≤ 1,000             O(n²) OK — O(n²) DP, bubble sort
n ≤ 100,000           O(n log n) — sorting, heap, BST
n ≤ 1,000,000         O(n) — linear scan, sliding window
n ≤ 1,000,000,000     O(log n) — binary search
```

---

## 11. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the difference between O(n) and O(n²) in practice? | O(n) grows linearly — doubling the input doubles the time. O(n²) grows quadratically — doubling the input quadruples the time. For n=10,000: O(n) ≈ 10,000 operations; O(n²) ≈ 100,000,000 operations. In interviews, an O(n²) solution is usually the "obvious" nested-loop approach; the interviewer expects you to optimize it. Common O(n²) → O(n) transitions: nested loops → hash map; repeated comparisons → sorting first; recalculating sums → prefix sums. |
| What is amortized O(1) and how does `ArrayList` use it? | Amortized analysis averages the cost of an operation over a sequence of operations. `ArrayList.add()` is O(1) amortized: most adds just set an array element (O(1)), but occasionally the array is full and must be resized — a new array double the size is allocated and all elements copied (O(n)). This expensive copy happens rarely enough that the average cost per add over n operations is still O(1). The "doubling" strategy is key — if the array grew by 1 each time, each add would be O(n) and the total would be O(n²). |
| What is the space complexity of a recursive DFS on a tree of height h? | O(h) — the call stack holds one frame per level of recursion, and the maximum depth of recursion equals the tree height. For a balanced binary tree, h = O(log n). For a degenerate (skewed) tree, h = O(n), risking `StackOverflowError` on large inputs. When asked about the space complexity of a recursive algorithm, always include the call stack. To avoid this, convert deep recursion to iterative DFS using an explicit `Deque`. |
| What is the time complexity of `HashMap.get()` in the worst case and why? | O(n) in the worst case. If all keys hash to the same bucket (e.g., a crafted adversarial input), the bucket becomes a linked list of n entries and lookup requires scanning the entire list. In Java 8+, buckets with more than 8 entries are converted to red-black trees, making worst-case O(log n) per bucket. Average case with a good hash function remains O(1). In interviews, always state "O(1) average, O(n) worst case" unless you can guarantee no collisions. |
| When is O(n log n) the best achievable complexity for a comparison-based sort? | Always — it is a proven lower bound. Any comparison-based sorting algorithm must make at least Ω(n log n) comparisons in the worst case. The proof: n elements have n! possible orderings; each comparison eliminates half the possibilities (like a binary decision tree); the decision tree has at least n! leaves, so its height is at least log₂(n!) ≈ n log n by Stirling's approximation. Non-comparison sorts (Counting Sort, Radix Sort) can beat this but only in specific circumstances (bounded integers, small alphabet). |
| What is the complexity of Dijkstra's algorithm and what data structure drives it? | With a **binary heap (PriorityQueue)**: O((V + E) log V) — each vertex is extracted once (V × log V for heap ops) and each edge is relaxed once (E × log V for potential heap updates). With a **Fibonacci heap**: O(E + V log V) — decrease-key is O(1) amortized, but complex to implement. In interviews, the binary heap version is standard. Note: Dijkstra does not work with negative edge weights — use Bellman-Ford instead (O(V × E)). |
| What is the difference between `O(log n)` and `O(n log n)` algorithms? Give examples. | O(log n): the problem is halved at each step — binary search, balanced BST lookup, heap push/pop. The input size grows exponentially before the time doubles. O(n log n): you do O(log n) work for each of n elements — merge sort, heap sort, building a heap from n elements, most divide-and-conquer sorting algorithms. In interviews: if you see "sorted array + search," think O(log n). If you see "sort then process," think O(n log n). |
