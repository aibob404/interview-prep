# DSA Patterns

## 1. Two Pointers

**Use when:** array is sorted, need to find pairs with a condition.

```java
// Two Sum II — sorted array
public int[] twoSum(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) return new int[]{left, right};
        else if (sum < target) left++;
        else right--;
    }
    return new int[]{};
}
// Time: O(n), Space: O(1)

// Remove duplicates from sorted array
public int removeDuplicates(int[] nums) {
    int slow = 0;
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) {
            nums[++slow] = nums[fast];
        }
    }
    return slow + 1;
}
```

---

## 2. Sliding Window

**Use when:** subarray/substring of variable or fixed size satisfying a condition.

```java
// Maximum sum subarray of size k (fixed window)
public int maxSum(int[] arr, int k) {
    int sum = 0, maxSum = 0;
    for (int i = 0; i < k; i++) sum += arr[i];
    maxSum = sum;
    for (int i = k; i < arr.length; i++) {
        sum += arr[i] - arr[i - k];
        maxSum = Math.max(maxSum, sum);
    }
    return maxSum;
}

// Longest substring without repeating characters (variable window)
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int max = 0, left = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1; // shrink window
        }
        lastSeen.put(c, right);
        max = Math.max(max, right - left + 1);
    }
    return max;
}
// Time: O(n), Space: O(min(n, alphabet))

// Minimum window substring — covers all chars of t
public String minWindow(String s, String t) {
    Map<Character, Integer> need = new HashMap<>();
    for (char c : t.toCharArray()) need.merge(c, 1, Integer::sum);
    int left = 0, have = 0, required = need.size(), minLen = Integer.MAX_VALUE, start = 0;
    Map<Character, Integer> window = new HashMap<>();

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.merge(c, 1, Integer::sum);
        if (need.containsKey(c) && window.get(c).equals(need.get(c))) have++;

        while (have == required) {
            if (right - left + 1 < minLen) { minLen = right - left + 1; start = left; }
            char lc = s.charAt(left++);
            window.merge(lc, -1, Integer::sum);
            if (need.containsKey(lc) && window.get(lc) < need.get(lc)) have--;
        }
    }
    return minLen == Integer.MAX_VALUE ? "" : s.substring(start, start + minLen);
}
```

---

## 3. Fast & Slow Pointers

**Use when:** detect cycle, find middle, nth from end.

```java
// Detect cycle in linked list
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true; // met → cycle exists
    }
    return false;
}

// Find middle of linked list
public ListNode findMiddle(ListNode head) {
    ListNode slow = head, fast = head;
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
    }
    return slow; // slow is at middle
}
```

---

## 4. Binary Search

**Use when:** sorted array, or answer lies in a monotonic range.

```java
// Classic: find target in sorted array
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoid integer overflow
        if (nums[mid] == target) return mid;
        else if (nums[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}

// Find first true in FFFFFFTTTT pattern (leftmost satisfying condition)
public int firstTrue(boolean[] arr) {
    int left = 0, right = arr.length - 1, result = -1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid]) { result = mid; right = mid - 1; } // maybe better to left
        else left = mid + 1;
    }
    return result;
}

// Binary search on answer space
// "What is the minimum capacity such that you can ship in D days?"
public int shipWithinDays(int[] weights, int days) {
    int left = Arrays.stream(weights).max().getAsInt(); // min possible: heaviest item
    int right = Arrays.stream(weights).sum();           // max possible: all in one day

    while (left < right) {
        int mid = left + (right - left) / 2;
        if (canShip(weights, days, mid)) right = mid; // try smaller
        else left = mid + 1;
    }
    return left;
}
```

---

## 5. BFS — Breadth-First Search

**Use when:** shortest path in unweighted graph, level-order traversal.

```java
// Shortest path in unweighted graph
public int shortestPath(int[][] grid, int[] start, int[] end) {
    int rows = grid.length, cols = grid[0].length;
    Queue<int[]> queue = new LinkedList<>();
    boolean[][] visited = new boolean[rows][cols];

    queue.offer(start);
    visited[start[0]][start[1]] = true;
    int steps = 0;
    int[][] dirs = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!queue.isEmpty()) {
        int size = queue.size();
        for (int i = 0; i < size; i++) {
            int[] curr = queue.poll();
            if (curr[0] == end[0] && curr[1] == end[1]) return steps;
            for (int[] d : dirs) {
                int r = curr[0] + d[0], c = curr[1] + d[1];
                if (r >= 0 && r < rows && c >= 0 && c < cols
                        && !visited[r][c] && grid[r][c] == 0) {
                    visited[r][c] = true;
                    queue.offer(new int[]{r, c});
                }
            }
        }
        steps++;
    }
    return -1;
}
```

---

## 6. DFS — Depth-First Search

**Use when:** explore all paths, tree traversal, connected components, backtracking.

```java
// Count islands
public int numIslands(char[][] grid) {
    int count = 0;
    for (int r = 0; r < grid.length; r++) {
        for (int c = 0; c < grid[0].length; c++) {
            if (grid[r][c] == '1') {
                dfs(grid, r, c);
                count++;
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int r, int c) {
    if (r < 0 || r >= grid.length || c < 0 || c >= grid[0].length || grid[r][c] != '1') return;
    grid[r][c] = '0'; // mark visited (in-place)
    dfs(grid, r+1, c); dfs(grid, r-1, c);
    dfs(grid, r, c+1); dfs(grid, r, c-1);
}
```

---

## 7. Backtracking

**Use when:** generate all combinations/permutations/subsets, satisfy constraints.

```java
// All permutations
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), new boolean[nums.length], result);
    return result;
}

private void backtrack(int[] nums, List<Integer> current, boolean[] used,
                        List<List<Integer>> result) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    for (int i = 0; i < nums.length; i++) {
        if (used[i]) continue;
        used[i] = true;
        current.add(nums[i]);
        backtrack(nums, current, used, result);
        current.remove(current.size() - 1); // undo
        used[i] = false;
    }
}

// Subsets
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, List<Integer> current,
                        List<List<Integer>> result) {
    result.add(new ArrayList<>(current));
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);
        backtrack(nums, i + 1, current, result);
        current.remove(current.size() - 1);
    }
}
```

---

## 8. Dynamic Programming

### Memoization (Top-Down)
```java
// Fibonacci
Map<Integer, Long> memo = new HashMap<>();
public long fib(int n) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n);
    long result = fib(n-1) + fib(n-2);
    memo.put(n, result);
    return result;
}
```

### Tabulation (Bottom-Up)
```java
// Coin change — minimum coins to make amount
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // initialize with "impossible" value
    dp[0] = 0;

    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}

// Longest common subsequence
public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m+1][n+1];

    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1))
                dp[i][j] = dp[i-1][j-1] + 1;
            else
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
        }
    }
    return dp[m][n];
}
```

### DP Problem Categories
| Pattern | Example problems |
|---------|-----------------|
| 1D DP | Fibonacci, climbing stairs, house robber |
| 2D DP | LCS, edit distance, unique paths |
| Knapsack | 0/1 knapsack, coin change, partition equal subset |
| Interval DP | Matrix chain multiplication, burst balloons |
| String DP | Longest palindromic substring, word break |

---

## 9. Topological Sort

**Use when:** dependency ordering, course schedule (detect cycle in directed graph).

```java
// BFS-based (Kahn's algorithm)
public int[] topologicalSort(int numCourses, int[][] prerequisites) {
    int[] inDegree = new int[numCourses];
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < numCourses; i++) adj.add(new ArrayList<>());

    for (int[] pre : prerequisites) {
        adj.get(pre[1]).add(pre[0]);
        inDegree[pre[0]]++;
    }

    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < numCourses; i++) {
        if (inDegree[i] == 0) queue.offer(i);
    }

    int[] order = new int[numCourses];
    int idx = 0;
    while (!queue.isEmpty()) {
        int course = queue.poll();
        order[idx++] = course;
        for (int next : adj.get(course)) {
            if (--inDegree[next] == 0) queue.offer(next);
        }
    }
    return idx == numCourses ? order : new int[]{}; // empty if cycle detected
}
```

---

## 10. Pattern Recognition Guide

| Clue in problem | Try |
|-----------------|-----|
| "Sorted array", "find pair/triplet" | Two Pointers |
| "Subarray/substring of size K" | Sliding Window |
| "Linked list cycle / middle" | Fast & Slow Pointers |
| "Sorted input, find target efficiently" | Binary Search |
| "Minimum path", "level by level", "shortest" | BFS |
| "All paths", "connected components", "tree" | DFS |
| "All combinations/permutations/subsets" | Backtracking |
| "Optimal substructure", "overlapping subproblems" | DP |
| "Dependencies/ordering", "detect cycle in directed graph" | Topological Sort |
| "Top K", "Kth largest/smallest" | Heap (PriorityQueue) |
| "Interval overlap", "meeting rooms" | Sort + sweep |
| "Frequency count", "anagram" | HashMap / sliding window + map |

---

## 11. Common Interview Questions

| Question | Answer |
|----------|--------|
| When do you use BFS vs DFS? | **BFS** explores level by level using a queue. Use it for **shortest path** in an unweighted graph, level-order traversal, or "minimum number of steps" problems — BFS guarantees the first time a node is reached is via the shortest path. **DFS** explores a branch fully before backtracking. Use it for detecting cycles, finding all paths, connected components, topological sort, and tree traversals. DFS uses O(depth) space; BFS uses O(width) space — BFS can be expensive for wide graphs. |
| How does two-pointer technique work and what are its prerequisites? | Two pointers work by maintaining two indices into the input — often starting at opposite ends or moving in the same direction at different speeds. The key prerequisite for the "converging" variant is a **sorted array** (or a structure with a known relationship between elements). Typical problems: finding pairs that sum to a target, removing duplicates in-place, checking palindromes. The same-direction variant (fast/slow pointers) works on linked lists to detect cycles (`Floyd's algorithm`) or find the middle. Time complexity is O(n) since each pointer moves at most n steps. |
| What is the sliding window technique and when does a fixed vs variable window apply? | A sliding window maintains a contiguous subarray/substring range with two pointers (`left`, `right`). **Fixed window** (`right - left == k`): slide both pointers together — used when the subarray size is given. **Variable window**: expand `right` to grow the window, shrink `left` when a constraint is violated — used for "longest subarray with at most K distinct elements" or "smallest subarray with sum ≥ target." The window avoids recomputing from scratch each step, reducing O(n²) nested-loop solutions to O(n). |
| What is binary search on the answer space? | Instead of searching for a target in an array, you binary search over the range of possible **answers**. You define a monotonic predicate (e.g., `canAchieve(mid)`) and find the boundary where it switches from false to true. Classic examples: "minimum capacity to ship packages in D days," "Koko eating bananas," "find the smallest divisor." The key insight: if a solution with value X is feasible, then any value > X is also feasible (or vice versa) — the feasibility function is monotone, enabling binary search. Reduces O(n×answer) brute force to O(n log(answer)). |
| How do you approach a dynamic programming problem from scratch? | (1) **Identify overlapping subproblems** — can the problem be broken into subproblems that are solved repeatedly? (2) **Define the state** — what parameters uniquely identify a subproblem? (3) **Write the recurrence** — express the solution in terms of smaller states. (4) **Choose top-down (memoization)** — add a cache to recursion, easiest to write; or **bottom-up (tabulation)** — fill a DP table iteratively, better for space optimization. (5) **Optimize space** if only the last row/value is needed. Common patterns: 1D (Fibonacci, house robber), 2D (grid paths, edit distance), knapsack (include/exclude). |
| What is backtracking and how does pruning make it efficient? | Backtracking is a DFS over a decision tree: at each step you make a choice, recurse, then undo the choice. It explores all possibilities but uses **pruning** to cut branches early when a partial solution already violates constraints. Example: N-Queens — after placing a queen, immediately prune any column/diagonal that is now invalid. Without pruning, backtracking is brute-force O(n!). With effective pruning, it is much faster in practice. The template: `if valid, add choice → recurse → remove choice`. Use when asked for "all combinations/permutations/subsets." |
| What is topological sort and what data structure does it use? | Topological sort produces a linear ordering of nodes in a **directed acyclic graph (DAG)** such that for every edge u→v, u appears before v. It is used for dependency resolution, build systems, and course scheduling. **Kahn's algorithm** (BFS-based): start with all nodes of in-degree 0, process them, decrement neighbors' in-degrees, enqueue any that reach 0. Uses a queue. **DFS-based**: post-order DFS, push each node to a stack when all descendants are visited; reverse the stack. Cycle detection is a byproduct — if not all nodes are processed, a cycle exists. |
| When should you use a heap (PriorityQueue) in an algorithm? | Use a heap when you repeatedly need the **minimum or maximum element** from a changing set. Key patterns: (1) **Top K elements** — maintain a min-heap of size K; if a new element is larger than the heap's min, replace it. O(n log K). (2) **K-way merge** — merge K sorted lists by tracking the current minimum across all lists. O(n log K). (3) **Dijkstra's shortest path** — always relax the unvisited node with the smallest tentative distance. (4) **Sliding window maximum** — monotonic deque (deque, not heap, but same idea). A heap does not support arbitrary deletion in O(1); use a "lazy deletion" trick for those cases. |
