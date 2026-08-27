# Phase 02 — DSA & Collections

> **Mục tiêu:** Hiểu các cấu trúc dữ liệu và thuật toán đủ để chọn đúng tool cho đúng bài toán — và giải thích được tại sao. Không cần tự viết lại JDK Collections.

---

## Nguyên tắc học

```
Không học thuộc code
        ↓
Hiểu tại sao structure đó tồn tại
        ↓
Implement cơ bản để nắm internal
        ↓
Dùng JDK version trong thực tế
        ↓
Benchmark & quan sát behavior
```

---

## Part 1 — Data Structures

### Linear Structures
| Structure | Internal | Khi nào dùng | Trade-off |
|-----------|----------|--------------|-----------|
| Array | Contiguous memory | Random access O(1) | Fixed size, insert O(n) |
| ArrayList | Dynamic array | General purpose list | Resize cost, not thread-safe |
| LinkedList | Doubly linked | Frequent insert/remove at ends | No random access O(n), high memory |
| Stack | LIFO | DFS, call stack, undo | — |
| Queue | FIFO | BFS, task queue | — |
| Deque | Both ends | Sliding window, palindrome | `ArrayDeque` thường tốt hơn LinkedList |

### Hash-based Structures
| Structure | Internal | Khi nào dùng |
|-----------|----------|--------------|
| HashMap | Array + LinkedList/Tree (Java 8+) | Key-value lookup O(1) avg |
| LinkedHashMap | HashMap + doubly linked list | Maintain insertion order |
| TreeMap | Red-Black Tree | Sorted key, range queries |
| HashSet | HashMap (value = dummy) | Unique elements, O(1) contains |
| TreeSet | TreeMap | Sorted unique elements |

**Implement để hiểu (optional):**
- `MyHashMap<K, V>` — array of buckets, separate chaining, resize
- `MyArrayList<T>` — dynamic array, grow by 1.5x

### Tree Structures
| Structure | Đặc điểm | Dùng khi |
|-----------|---------|---------|
| BST | Left < root < right | Ordered search — nhưng có thể degenerate |
| Heap / PriorityQueue | Complete binary tree, heap property | Top-K, scheduling, Dijkstra |
| Trie | Prefix tree | Autocomplete, dictionary |

### Graph
| Representation | Khi nào | Trade-off |
|---------------|---------|-----------|
| Adjacency Matrix | Dense graph | Space O(V²) |
| Adjacency List | Sparse graph (thường gặp hơn) | Space O(V+E) |

**Union-Find (Disjoint Set):**
- `find(x)` với path compression
- `union(x, y)` với union by rank
- Dùng: connected components, cycle detection

---

## Part 2 — Algorithms

### Sorting
| Algorithm | Time (avg) | Space | Stable | Ghi chú |
|-----------|-----------|-------|--------|---------|
| Merge Sort | O(n log n) | O(n) | Yes | Divide & conquer, Java's `TimSort` base |
| Quick Sort | O(n log n) | O(log n) | No | In-place, cache-friendly |
| Heap Sort | O(n log n) | O(1) | No | Guaranteed, nhưng chậm trong thực tế |
| `Arrays.sort()` | O(n log n) | — | — | Dual-pivot quicksort (primitive), TimSort (object) |

### Searching
| Technique | Yêu cầu | Complexity |
|-----------|---------|-----------|
| Linear Search | None | O(n) |
| Binary Search | Sorted array | O(log n) |
| Two Pointers | Sorted / specific structure | O(n) |
| Sliding Window | Subarray/substring problems | O(n) |
| Prefix Sum | Range sum queries | O(1) query after O(n) build |

### Recursion & Backtracking
```
Recursion = solve(subproblem) + combine
Backtracking = Recursion + undo step khi path không hợp lệ
```
- Base case luôn phải rõ ràng
- Stack overflow: max depth = stack size (~500-1000 frames mặc định)

### Graph Traversal
| Algorithm | Dùng | Cấu trúc |
|-----------|------|---------|
| BFS | Shortest path (unweighted), level-order | Queue |
| DFS | Connected components, topological sort, cycle detection | Stack (explicit hoặc recursion) |
| Dijkstra | Shortest path (weighted, non-negative) | PriorityQueue |

### Dynamic Programming
```
Optimal substructure + Overlapping subproblems
        ↓
Memoization (top-down) hoặc Tabulation (bottom-up)
```
- Nhận dạng bài DP: tìm min/max, đếm số cách, kiểm tra khả thi
- Pattern: Fibonacci, Knapsack, LCS, LIS, Coin Change

---

## Part 3 — Java Collections Deep Dive

### Khi nào dùng gì

```
Cần list đơn giản          → ArrayList
Cần insert/remove đầu/cuối → ArrayDeque
Cần key-value lookup       → HashMap
Cần key-value theo thứ tự  → LinkedHashMap (insert order) / TreeMap (sorted)
Cần unique elements        → HashSet
Cần unique + sorted        → TreeSet
Cần priority queue         → PriorityQueue
Thread-safe map            → ConcurrentHashMap (phase 07)
```

### HashMap internals
```
put(key, value):
  hash = hash(key.hashCode())
  bucket = hash % capacity
  nếu bucket trống → tạo node
  nếu collision → chaining (LinkedList → Tree khi ≥ 8 entries)
  nếu load factor > 0.75 → resize (double capacity, rehash)
```

**Tại sao `hashCode()` phải nhất quán với `equals()`?**
- Nếu `a.equals(b)` thì bắt buộc `a.hashCode() == b.hashCode()`
- Ngược lại không bắt buộc (collision là bình thường)
- Vi phạm → `HashMap.get()` trả về `null` dù key "tồn tại"

---

## Project — Algorithm & Collections Lab

**Không build application đầy đủ. Đây là lab với nhiều experiment nhỏ.**

```
src/
├── ds/
│   ├── MyArrayList.java          ← tự implement để hiểu internals
│   ├── MyHashMap.java            ← array of buckets + chaining
│   └── MyMinHeap.java            ← heap property, heapify
├── sorting/
│   ├── MergeSort.java
│   ├── QuickSort.java
│   └── SortingBenchmark.java     ← so sánh performance
├── searching/
│   ├── BinarySearch.java
│   ├── TwoPointers.java
│   └── SlidingWindow.java
├── graph/
│   ├── BFS.java
│   ├── DFS.java
│   ├── Dijkstra.java
│   └── UnionFind.java
├── dp/
│   ├── Fibonacci.java            ← memoization vs tabulation
│   ├── Knapsack.java
│   └── LongestIncreasingSubsequence.java
└── collections/
    ├── HashMapExperiment.java    ← collision, load factor, resize
    ├── TreeMapVsHashMap.java     ← benchmark
    └── PriorityQueueDemo.java
```

---

## Checklist — 6 câu hỏi

Ví dụ với `HashMap`:

| # | Câu hỏi | Trả lời |
|---|---------|---------|
| 1 | Là gì? | Hash table — key-value store, O(1) avg get/put |
| 2 | Giải quyết gì? | Lookup nhanh thay vì scan toàn bộ list |
| 3 | Hoạt động thế nào? | hash → bucket → chaining/tree, resize khi load > 0.75 |
| 4 | Khi nào dùng? | Cần lookup nhanh theo key |
| 5 | Khi nào không? | Cần sorted → TreeMap; cần insertion order → LinkedHashMap |
| 6 | Debug thế nào? | Nếu chậm bất thường → kiểm tra hashCode() collision; nếu key không tìm thấy → kiểm tra equals()/hashCode() contract |

---

## Run

```bash
mvn test -pl phase-02-dsa-collections
```
