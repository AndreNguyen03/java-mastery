# Phase 02 — DSA & Collections · Lý thuyết & Lab

---

## 1. Lý thuyết

### 1.0 Collections Framework Hierarchy

**Java Collections Framework — toàn cảnh:**
```
Iterable
  └── Collection
        ├── List        → ordered, allow duplicates: ArrayList, LinkedList, ArrayDeque
        ├── Set         → no duplicates: HashSet, LinkedHashSet, TreeSet
        └── Queue/Deque → FIFO/LIFO: LinkedList, ArrayDeque, PriorityQueue
Map (not Collection)    → key-value: HashMap, LinkedHashMap, TreeMap
```

**Chọn đúng collection — quyết định performance:**
| Cần gì | Collection | Reason |
|---|---|---|
| Random access by index | `ArrayList` | O(1) get(i) |
| Insert/delete ở đầu | `ArrayDeque` | O(1) addFirst (ArrayList: O(n) shift) |
| No duplicates, fast lookup | `HashSet` | O(1) contains |
| No duplicates, sorted | `TreeSet` | O(log n) — Red-Black Tree |
| No duplicates, insertion order | `LinkedHashSet` | O(1) + preserves order |
| Key-value, fast lookup | `HashMap` | O(1) get/put |
| Key-value, sorted keys | `TreeMap` | O(log n) — sorted iteration |
| Key-value, insertion order | `LinkedHashMap` | O(1) + LRU cache use case |
| Priority queue (min/max) | `PriorityQueue` | O(log n) offer/poll, O(1) peek |

---

### 1.00 LinkedList — Doubly Linked List

**Cấu trúc:**
```
null ← [A] ↔ [B] ↔ [C] ↔ [D] → null
       head                tail
```
Mỗi node có `prev`, `next` pointer. Java's `LinkedList` implements cả `List` và `Deque`.

**Trade-offs vs ArrayList:**
- `add(0, x)` → O(1): chỉ thay đổi head pointer (ArrayList: O(n) shift)
- `get(i)` → O(n): phải traverse từ head (ArrayList: O(1))
- Memory: mỗi node = data + 2 pointers (~24 bytes overhead vs ~4 bytes in array)
- Cache locality kém: nodes scattered trong heap → cache miss nhiều

**Thực tế:** `ArrayDeque` thường tốt hơn `LinkedList` ngay cả cho FIFO queue — array memory locality win.

---

### 1.000 TreeMap & TreeSet — Red-Black Tree

**Red-Black Tree là gì:**
Self-balancing Binary Search Tree. Mọi insertion/deletion tự rebalance để đảm bảo tree height = O(log n). Java dùng RBT cho `TreeMap`, `TreeSet`, và HashMap buckets khi > 8 entries.

**Properties:**
- Keys được sorted (natural order hoặc custom Comparator)
- `O(log n)` cho get, put, containsKey, remove
- In-order traversal → sorted output

**Khi nào dùng TreeMap thay HashMap:**
```java
TreeMap<LocalDate, Double> dailyRevenue = new TreeMap<>();
dailyRevenue.put(LocalDate.of(2024, 1, 15), 5000.0);
dailyRevenue.put(LocalDate.of(2024, 1, 10), 3000.0);

// Range queries — chỉ có ở TreeMap (không có ở HashMap)
SortedMap<LocalDate, Double> week = dailyRevenue.subMap(
    LocalDate.of(2024, 1, 10),
    LocalDate.of(2024, 1, 17)
);

// First/last key
LocalDate oldest = dailyRevenue.firstKey();
LocalDate newest = dailyRevenue.lastKey();

// Ceiling/floor
LocalDate nextDate = dailyRevenue.ceilingKey(LocalDate.now());
```

---

### 1.0000 PriorityQueue — Min/Max Heap

**Heap là gì:**
Complete binary tree thỏa mãn heap property:
- **Min-heap:** parent ≤ children → `peek()` trả min element — Java's default
- **Max-heap:** parent ≥ children → dùng `Comparator.reverseOrder()`

**Heap operations:**
- `offer(x)` → O(log n): add vào cuối, bubble-up để restore heap property
- `poll()` → O(log n): remove root, bubble-down
- `peek()` → O(1): xem root (min/max) mà không remove

**Use cases:**
```java
// Top K elements (K-th largest problem)
PriorityQueue<Integer> minHeap = new PriorityQueue<>(k); // min-heap of size k
for (int num : nums) {
    minHeap.offer(num);
    if (minHeap.size() > k) minHeap.poll(); // remove smallest
}
// minHeap.peek() = k-th largest

// Task scheduling by priority
PriorityQueue<Task> tasks = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority).reversed() // max-priority first
);
tasks.offer(new Task("low", 1));
tasks.offer(new Task("high", 10));
tasks.poll(); // "high" — priority 10 first
```

**Dijkstra's algorithm dùng PriorityQueue:**
```java
// Shortest path: always process nearest unvisited node first
PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[1]));
// [node, distance]
pq.offer(new int[]{source, 0});
while (!pq.isEmpty()) {
    int[] curr = pq.poll(); // node with smallest distance
    // process neighbors...
}
```

---

### 1.00000 Binary Search — Patterns & Variants

**Standard binary search:**
```java
int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2; // avoid integer overflow (vs (left+right)/2)
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1; // not found
}
```

**Find first occurrence (lower bound):**
```java
int lowerBound(int[] arr, int target) {
    int left = 0, right = arr.length;
    while (left < right) { // note: < not <=
        int mid = left + (right - left) / 2;
        if (arr[mid] < target) left = mid + 1;
        else right = mid; // don't exclude mid, might be first occurrence
    }
    return left; // insertion point / first position >= target
}
```

**Binary search on answer:**
Khi không search trên mảng mà search trên "answer space":
```java
// "Minimum capacity to ship packages within D days"
// Binary search: try capacity X → check if feasible → narrow range
int left = maxWeight, right = totalWeight;
while (left < right) {
    int mid = left + (right - left) / 2;
    if (canShip(packages, mid, D)) right = mid; // mid feasible, try smaller
    else left = mid + 1; // mid too small, need more
}
```

---

### 1.000000 Sliding Window & Two Pointers

**Two Pointers — O(n) thay O(n²):**
```java
// Pair sum = target (sorted array)
int left = 0, right = arr.length - 1;
while (left < right) {
    int sum = arr[left] + arr[right];
    if (sum == target) { found(); left++; right--; }
    else if (sum < target) left++;  // need bigger sum
    else right--;                    // need smaller sum
}
// O(n) vs brute force O(n²)
```

**Sliding Window — for subarray/substring problems:**
```java
// Longest substring without repeating characters
Map<Character, Integer> lastSeen = new HashMap<>();
int maxLen = 0, left = 0;
for (int right = 0; right < s.length(); right++) {
    char c = s.charAt(right);
    if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
        left = lastSeen.get(c) + 1; // shrink window
    }
    lastSeen.put(c, right);
    maxLen = Math.max(maxLen, right - left + 1);
}
```

---

### 1.0000000 DP Patterns — Tư duy Quy hoạch động

**Khi nào dùng DP:**
Bài toán có: (1) optimal substructure — optimal solution bao gồm optimal sub-solutions, và (2) overlapping subproblems — cùng sub-problem được giải nhiều lần.

**Memoization (Top-down) vs Tabulation (Bottom-up):**
```java
// Fibonacci — Memoization: natural recursive + cache
Map<Integer, Long> memo = new HashMap<>();
long fib(int n) {
    if (n <= 1) return n;
    if (memo.containsKey(n)) return memo.get(n); // cache hit
    long result = fib(n-1) + fib(n-2);
    memo.put(n, result);
    return result;
}

// Fibonacci — Tabulation: fill table bottom-up (better: no stack overflow)
long[] dp = new long[n + 1];
dp[0] = 0; dp[1] = 1;
for (int i = 2; i <= n; i++) dp[i] = dp[i-1] + dp[i-2];

// Space optimization: only need last 2 values
long a = 0, b = 1;
for (int i = 2; i <= n; i++) { long tmp = a + b; a = b; b = tmp; }
```

**Coin Change — classic DP:**
```java
// Minimum coins to make amount
int[] dp = new int[amount + 1];
Arrays.fill(dp, amount + 1); // initialize with "infinity"
dp[0] = 0;
for (int coin : coins) {
    for (int i = coin; i <= amount; i++) {
        dp[i] = Math.min(dp[i], dp[i - coin] + 1);
    }
}
return dp[amount] > amount ? -1 : dp[amount];
```

---

### 1.1 ArrayList — Dynamic Array

**Cấu trúc bên trong:** `ArrayList` là một **mảng `Object[]` có thể tự mở rộng**. Khi đầy, nó:
1. Cấp phát mảng mới lớn hơn (thường gấp 1.5x)
2. Copy toàn bộ phần tử vào mảng mới (`Arrays.copyOf`)
3. Trỏ reference sang mảng mới

**Trade-offs:**
- `get(i)` → O(1) — truy cập random bằng index
- `add(end)` → O(1) amortized — thỉnh thoảng O(n) khi resize
- `add(i)` ở giữa → O(n) — phải shift phần tử
- `remove(i)` ở giữa → O(n) — phải shift

**Khi nào dùng vs LinkedList:**
- ArrayList tốt cho: random access, ít insert/delete ở giữa
- LinkedList tốt cho: nhiều insert/delete ở đầu/cuối, không cần random access
- Thực tế: ArrayList thường win vì **cache locality** — phần tử liền kề trong memory, CPU prefetch hiệu quả

**Growth factor 1.5x vs 2x:**
- 2x (HashMap, Python list): tốn memory hơn, ít copy hơn
- 1.5x (ArrayList): cân bằng hơn, tổng số copy vẫn O(n) amortized về mặt lý thuyết

---

### 1.2 HashMap — Hash Table với Separate Chaining

**Cách HashMap lưu dữ liệu:**
1. Tính `hash = key.hashCode() ^ (hash >>> 16)` — xor với high bits để phân phối đều hơn
2. `index = hash & (capacity - 1)` — modulo với capacity (phải là power of 2)
3. Tại bucket index đó: LinkedList (hoặc TreeNode khi dài > 8)

**Tại sao xor với `(h >>> 16)`:**
Nếu chỉ lấy `h & (capacity-1)`, khi capacity nhỏ (16), chỉ dùng 4 bits thấp nhất của hash. Nếu nhiều keys có cùng 4 bits thấp → nhiều collision. XOR với high bits trộn thêm entropy → ít collision hơn.

**Load factor 0.75:**
Khi `size / capacity > 0.75`, HashMap resize (capacity × 2). 
- 0.75 cân bằng giữa memory usage và collision rate
- 0.5 → ít collision nhưng tốn memory
- 0.9 → tiết kiệm memory nhưng nhiều collision, lookup chậm

**Tại sao key cần implement `hashCode()` VÀ `equals()` đúng:**
- `hashCode()` quyết định bucket nào
- `equals()` tìm đúng key trong bucket
- Nếu `equals()` trả `true` nhưng `hashCode()` khác → hai objects "bằng nhau" nằm ở hai bucket khác nhau → get() không tìm thấy!
- Contract: `a.equals(b) == true` → `a.hashCode() == b.hashCode()` (bắt buộc)

**Java 8+: Treeify khi bucket quá dài**
Khi một bucket có > 8 entries (linked list), Java chuyển sang Red-Black Tree → O(log n) thay vì O(n) trong trường hợp xấu.

---

### 1.3 Sorting Algorithms

**MergeSort:**
- Divide: chia mảng thành 2 nửa
- Conquer: sort đệ quy từng nửa
- Merge: gộp 2 nửa đã sort thành 1

**Tại sao MergeSort ổn định (stable):**
Khi merge, nếu hai phần tử bằng nhau, luôn lấy từ nửa trái trước → thứ tự tương đối được giữ nguyên. Java's `Arrays.sort()` cho Object dùng TimSort (cải tiến từ MergeSort) vì lý do này.

**QuickSort vs MergeSort:**
| | QuickSort | MergeSort |
|---|---|---|
| Average | O(n log n) | O(n log n) |
| Worst | O(n²) | O(n log n) |
| Space | O(log n) stack | O(n) merge buffer |
| Stable | Không | Có |
| Cache friendly | Có (in-place) | Ít hơn |

**Median-of-three pivot** — tránh O(n²) worst case:
Thay vì chọn pivot = phần tử cuối (bad với sorted input), chọn median của `arr[left]`, `arr[mid]`, `arr[right]`.

---

### 1.4 Graph Algorithms

**BFS (Breadth-First Search):**
Duyệt theo lớp (level by level) bằng Queue. Đảm bảo tìm đường ngắn nhất (số cạnh) trong unweighted graph. Tại sao? Vì mở rộng đều từ source, nodes gần source được thăm trước.

**DFS (Depth-First Search):**
Đi sâu nhất có thể trước khi backtrack. Dùng Stack (hoặc đệ quy). Không đảm bảo đường ngắn nhất. Thích hợp cho: tìm cycle, topological sort, connected components.

**UnionFind (Disjoint Set Union):**
Data structure theo dõi **nhóm** (connected components). Hai operations:
- `find(x)`: tìm representative (root) của nhóm chứa x
- `union(x, y)`: gộp nhóm chứa x và nhóm chứa y

**Path compression:** Khi `find(x)`, trỏ thẳng mọi node trên đường đi về root → tất cả future finds đều O(1). Tại sao hiệu quả? Lần đầu tốn O(depth), nhưng làm phẳng tree → lần sau O(1). Amortized nearly O(1) (technically O(α(n)) — inverse Ackermann).

---

### 1.5 Dynamic Programming — Knapsack

**Knapsack là gì:**
Có W kg tải trọng, n items mỗi item có `weight[i]` và `value[i]`. Chọn items để maximize value mà không vượt W.

**Tại sao cần DP (không dùng brute force):**
Brute force: 2^n tập con → O(2^n). Với n=30: 10^9 operations.
DP: O(n×W). Với n=30, W=100: 3000 operations.

**Recurrence relation:**
`dp[i][w]` = max value dùng i items đầu tiên với tải trọng tối đa w.
```
dp[i][w] = max(
    dp[i-1][w],                     // không lấy item i
    dp[i-1][w-weight[i]] + value[i] // lấy item i (nếu w >= weight[i])
)
```

**Space optimization (1D):**
Duyệt từ `W` xuống `weight[i]` — tại sao? Vì `dp[w]` phụ thuộc `dp[w-weight[i]]` (chưa update trong iteration này). Nếu duyệt từ trái sang → `dp[w-weight[i]]` đã bị update → item có thể được lấy nhiều lần (unbounded knapsack).

---

## 2. Vấn đề thường gặp & Cách fix

### Issue 1: ConcurrentModificationException

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));

// ❌ Ném ConcurrentModificationException
for (String s : list) {
    if (s.equals("b")) list.remove(s); // modify during iteration!
}

// ✅ Fix 1: Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove(); // safe
}

// ✅ Fix 2: removeIf (Java 8+) — đẹp nhất
list.removeIf(s -> s.equals("b"));
```

**Tại sao xảy ra:** ArrayList có `modCount` counter tăng mỗi lần cấu trúc thay đổi. Iterator kiểm tra `modCount` mỗi lần `next()` — nếu thay đổi → ném exception. Design để phát hiện lỗi sớm (fail-fast).

### Issue 2: HashMap KeySet với mutable keys

```java
// ❌ Rất nguy hiểm
class MutableKey {
    int value;
    public int hashCode() { return value; }
    public boolean equals(Object o) { ... }
}

MutableKey key = new MutableKey();
key.value = 1;
map.put(key, "hello");

key.value = 2; // hashCode thay đổi!
map.get(key);  // → null! key ở wrong bucket
```

**Rule:** HashMap keys phải là immutable hoặc hashCode() không dựa trên mutable fields. String, Integer, record đều safe.

### Issue 3: Stack Overflow trong DFS đệ quy

```java
// ❌ Với graph lớn → StackOverflowError (default stack ~512KB)
void dfs(int node) {
    visited[node] = true;
    for (int neighbor : graph.get(node)) {
        if (!visited[neighbor]) dfs(neighbor); // recursive
    }
}

// ✅ Iterative DFS với explicit Stack
void dfsIterative(int start) {
    Deque<Integer> stack = new ArrayDeque<>();
    stack.push(start);
    while (!stack.isEmpty()) {
        int node = stack.pop();
        if (visited[node]) continue;
        visited[node] = true;
        for (int neighbor : graph.get(node)) {
            if (!visited[neighbor]) stack.push(neighbor);
        }
    }
}
```

---

## 3. Code mẫu

```java
// MyArrayList.java
public class MyArrayList<T> implements Iterable<T> {
    private Object[] data;
    private int size;

    @SuppressWarnings("unchecked")
    public MyArrayList(int initialCapacity) {
        this.data = new Object[initialCapacity];
        this.size = 0;
    }

    public void add(T item) {
        ensureCapacity();
        data[size++] = item;
    }

    @SuppressWarnings("unchecked")
    public T get(int index) {
        if (index < 0 || index >= size)
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        return (T) data[index];
    }

    // 1.5x growth — amortized O(1) add
    private void ensureCapacity() {
        if (size == data.length) {
            int newCapacity = data.length + (data.length >> 1); // × 1.5
            data = Arrays.copyOf(data, newCapacity);
        }
    }

    public int size() { return size; }

    @Override
    public Iterator<T> iterator() {
        return new Iterator<T>() {
            int cursor = 0;
            @Override public boolean hasNext() { return cursor < size; }
            @Override public T next() {
                if (!hasNext()) throw new NoSuchElementException();
                return get(cursor++);
            }
        };
    }
}
```

```java
// MyHashMap.java
public class MyHashMap<K, V> {
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;

    private Node<K,V>[] buckets;
    private int size;

    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }

    @SuppressWarnings("unchecked")
    public MyHashMap() {
        this.buckets = new Node[DEFAULT_CAPACITY];
    }

    // Spread high bits of hash — reduce collision when capacity is small
    private int hash(Object key) {
        int h = key.hashCode();
        return h ^ (h >>> 16);
    }

    private int indexFor(int hash) {
        return hash & (buckets.length - 1); // capacity must be power of 2
    }

    public void put(K key, V value) {
        if (key == null) throw new IllegalArgumentException("null key not supported");
        int hash = hash(key);
        int idx = indexFor(hash);

        for (Node<K,V> node = buckets[idx]; node != null; node = node.next) {
            if (node.hash == hash && key.equals(node.key)) {
                node.value = value; // update existing
                return;
            }
        }

        // Prepend to chain (O(1))
        buckets[idx] = new Node<>(hash, key, value, buckets[idx]);
        size++;

        if ((float) size / buckets.length > LOAD_FACTOR) {
            resize();
        }
    }

    public V get(K key) {
        int hash = hash(key);
        int idx = indexFor(hash);
        for (Node<K,V> node = buckets[idx]; node != null; node = node.next) {
            if (node.hash == hash && key.equals(node.key)) return node.value;
        }
        return null;
    }

    @SuppressWarnings("unchecked")
    private void resize() {
        Node<K,V>[] old = buckets;
        buckets = new Node[old.length * 2];
        size = 0;
        for (Node<K,V> head : old) {
            for (Node<K,V> node = head; node != null; node = node.next) {
                put(node.key, node.value);
            }
        }
    }
}
```

```java
// BFS với shortest path tracking
public class BFS {
    public static List<Integer> shortestPath(Map<Integer, List<Integer>> graph, int src, int dst) {
        if (!graph.containsKey(src)) return Collections.emptyList();

        Map<Integer, Integer> parent = new HashMap<>();
        Queue<Integer> queue = new LinkedList<>();
        Set<Integer> visited = new HashSet<>();

        queue.offer(src);
        visited.add(src);
        parent.put(src, -1);

        while (!queue.isEmpty()) {
            int curr = queue.poll();
            if (curr == dst) break;

            for (int neighbor : graph.getOrDefault(curr, List.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    parent.put(neighbor, curr);
                    queue.offer(neighbor);
                }
            }
        }

        if (!parent.containsKey(dst)) return Collections.emptyList();

        // Reconstruct path
        List<Integer> path = new ArrayList<>();
        for (int node = dst; node != -1; node = parent.get(node)) {
            path.add(0, node); // prepend
        }
        return path;
    }
}
```

```java
// UnionFind với path compression + union by rank
public class UnionFind {
    private final int[] parent;
    private final int[] rank;

    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    // Path compression: make every node point directly to root
    public int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    // Union by rank: attach smaller tree under larger
    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false; // already connected
        if (rank[px] < rank[py]) { int tmp = px; px = py; py = tmp; }
        parent[py] = px;
        if (rank[px] == rank[py]) rank[px]++;
        return true;
    }

    public boolean connected(int x, int y) {
        return find(x) == find(y);
    }
}
```

```java
// Knapsack 0/1 với space optimization
public class Knapsack {
    public static int solve(int capacity, int[] weights, int[] values) {
        int n = weights.length;
        int[] dp = new int[capacity + 1];

        for (int i = 0; i < n; i++) {
            // Duyệt từ phải sang trái để tránh dùng item i nhiều lần
            // dp[w] phụ thuộc dp[w - weights[i]] từ iteration trước
            for (int w = capacity; w >= weights[i]; w--) {
                dp[w] = Math.max(dp[w], dp[w - weights[i]] + values[i]);
            }
        }
        return dp[capacity];
    }
}
```

---

## 4. Lab Steps

1. **ArrayList resize:** Tạo `MyArrayList(2)`, add 4 phần tử, in ra `size` và capacity (add field `getCapacity()`)
2. **HashMap collision:** Tạo class với `hashCode() { return 1; }` — mọi key vào cùng bucket → quan sát O(n) get
3. **BFS:** Build graph `0→1→2→3, 0→3` → `shortestPath(0, 3)` phải trả `[0, 3]` (không phải `[0,1,2,3]`)
4. **UnionFind:** Bài toán đếm số islands: iterate qua grid, union neighbors → `find` đếm roots
5. **Knapsack:** Trace qua dp array với input nhỏ để hiểu state transition

---

## 5. Checklist tự kiểm tra

- [ ] Sau 2 adds vào `MyArrayList(2)`, add thứ 3 → capacity tăng lên 3 (×1.5 = 3)
- [ ] HashMap với 2 keys cùng hashCode vẫn get() được đúng (vì equals check)
- [ ] BFS luôn trả đường ngắn nhất (số cạnh), DFS không đảm bảo
- [ ] UnionFind path compression: sau `find()`, `parent[x]` trỏ thẳng về root
- [ ] Knapsack 1D: loop `w` từ `capacity` xuống `weights[i]` — không phải từ 0 lên
- [ ] `List.of()` vs `new ArrayList<>()`: `List.of()` immutable — không thể add/remove
