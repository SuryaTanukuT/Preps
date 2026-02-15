
# 🔥 PART 1 — CORE DSA PATTERNS (Interview + Template)

---

# 1️⃣ Sliding Window

## ✅ Definition

Technique for solving **subarray/substring problems** in O(n) using two pointers.

## 📌 When to Use

* Contiguous elements
* “At most K”
* “Longest/Shortest substring”
* Fraud velocity detection

---

## 🔹 Template (Variable Window)

```ts
function slidingWindow(nums: number[], k: number): number {
  let left = 0;
  let sum = 0;
  let result = 0;

  for (let right = 0; right < nums.length; right++) {
    sum += nums[right];

    while (sum > k) {
      sum -= nums[left++];
    }

    result = Math.max(result, right - left + 1);
  }

  return result;
}
```

---

## 🏦 Fraud Detection Usage

> “More than 5 transactions in 10 seconds?”

Maintain timestamps:

* Add current
* Remove expired
* If size > limit → block

Time: O(n)

---

# 2️⃣ Two Pointers

## ✅ Definition

Two indices moving toward each other or same direction.

## 📌 When to Use

* Sorted arrays
* Pair sum
* Palindrome
* Merge operations

---

```ts
function isPalindrome(s: string): boolean {
  let left = 0, right = s.length - 1;

  while (left < right) {
    if (s[left++] !== s[right--]) return false;
  }

  return true;
}
```

---

# 3️⃣ Fast / Slow Pointer

## ✅ Definition

Two pointers at different speeds.

## 📌 Used For

* Cycle detection
* Middle of linked list

---

```ts
function hasCycle(head: any): boolean {
  let slow = head, fast = head;

  while (fast && fast.next) {
    slow = slow.next;
    fast = fast.next.next;
    if (slow === fast) return true;
  }

  return false;
}
```

---

# 4️⃣ Monotonic Stack

## ✅ Definition

Stack that maintains increasing or decreasing order.

## 📌 Used For

* Next greater element
* Stock span
* Histogram area

---

## 🔹 Stock Span (BFSI trading system)

```ts
function stockSpan(prices: number[]): number[] {
  const stack: number[] = [];
  const result: number[] = [];

  for (let i = 0; i < prices.length; i++) {
    while (stack.length && prices[stack[stack.length - 1]] <= prices[i]) {
      stack.pop();
    }

    result[i] = stack.length ? i - stack[stack.length - 1] : i + 1;
    stack.push(i);
  }

  return result;
}
```

Used in:

* Market analytics
* Risk exposure

---

# 5️⃣ Backtracking

## ✅ Definition

Try → Recurse → Undo

## 📌 Used For

* Permutations
* Combinations
* Sudoku
* Access control rule matching

---

```ts
function permute(nums: number[]): number[][] {
  const result: number[][] = [];

  function backtrack(path: number[]) {
    if (path.length === nums.length) {
      result.push([...path]);
      return;
    }

    for (const n of nums) {
      if (path.includes(n)) continue;
      path.push(n);
      backtrack(path);
      path.pop();
    }
  }

  backtrack([]);
  return result;
}
```

---

# 6️⃣ Topological Sort

## ✅ Definition

Ordering of DAG nodes.

## 📌 Used For

* Course schedule
* Dependency resolution
* Microservice boot order

---

```ts
function topoSort(graph: Map<number, number[]>): number[] {
  const inDegree = new Map<number, number>();
  const result: number[] = [];
  const queue: number[] = [];

  for (const [u, neighbors] of graph) {
    if (!inDegree.has(u)) inDegree.set(u, 0);
    for (const v of neighbors) {
      inDegree.set(v, (inDegree.get(v) || 0) + 1);
    }
  }

  for (const [node, degree] of inDegree) {
    if (degree === 0) queue.push(node);
  }

  while (queue.length) {
    const node = queue.shift()!;
    result.push(node);

    for (const neighbor of graph.get(node) || []) {
      inDegree.set(neighbor, inDegree.get(neighbor)! - 1);
      if (inDegree.get(neighbor) === 0) queue.push(neighbor);
    }
  }

  return result;
}
```

---

# 7️⃣ Union-Find (Disjoint Set)

## ✅ Definition

Efficient structure to detect connected components.

## 📌 BFSI Use: Identity Resolution

```ts
class UnionFind {
  parent: number[];

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
  }

  find(x: number): number {
    if (this.parent[x] !== x)
      this.parent[x] = this.find(this.parent[x]);
    return this.parent[x];
  }

  union(x: number, y: number) {
    this.parent[this.find(x)] = this.find(y);
  }
}
```

Used in:

* KYC identity merge
* Account linking

---

# 8️⃣ Prefix Sum

## ✅ Definition

Cumulative sum array.

## 📌 Used For

* Balance calculation
* Range sum query

```ts
function prefixSum(nums: number[]): number[] {
  const prefix = [0];

  for (const n of nums) {
    prefix.push(prefix[prefix.length - 1] + n);
  }

  return prefix;
}
```

---

# 9️⃣ Binary Search on Answer

## ✅ Definition

Binary search over solution space.

Used in:

* Minimum capacity problems
* Loan EMI threshold
* Allocation problems

---

# 🔟 Segment Tree (Advanced)

## ✅ Definition

Tree for range queries in O(log n).

Used in:

* Real-time trading systems
* Risk analytics

---

# 1️⃣1️⃣ Bit Manipulation

Used in:

* Permission flags
* Fraud bitmask scoring
* Feature toggles

Example:

```ts
const READ = 1 << 0;
const WRITE = 1 << 1;

let perm = READ | WRITE;
```

---

# 1️⃣2️⃣ Dynamic Programming Patterns

Must know:

* 0/1 Knapsack
* Longest Increasing Subsequence
* Coin Change
* Matrix DP
* Interval DP

Senior view:

> DP = Overlapping subproblems + memoization.

---

# 🔥 PART 2 — DISTRIBUTED SYSTEM PATTERNS

---

# 1️⃣ Consistent Hashing

## ✅ Definition

Distributes keys across servers minimizing rehashing.

Used in:

* Redis cluster
* Distributed cache
* Microservices sharding

---

# 2️⃣ Bloom Filter

## ✅ Definition

Probabilistic structure to test membership.

Used in:

* Prevent DB hits
* API key validation
* Blacklist checks

---

# 3️⃣ Circuit Breaker

States:

* Closed
* Open
* Half-open

Used in:

* Payment gateway integration
* External credit bureau APIs

---

# 4️⃣ Idempotency Keys

Used in:

* Payment APIs
* Retry-safe transaction systems

Flow:

1. Client sends unique key
2. Server checks Redis
3. If exists → return stored response
4. Else process + store

---

# 5️⃣ Event-Driven Caching

Instead of TTL-only:

* DB update → Publish event
* Consumer invalidates cache

Used in:

* Account balance updates
* Customer profile updates

---

# 6️⃣ CQRS (Command Query Responsibility Segregation)

Separate:

* Write DB
* Read DB (optimized)

Used in:

* High transaction banking systems
* Reporting dashboards

---

# 🔥 Senior-Level View

In interviews, combine:

* Sliding Window → Fraud detection
* Monotonic Stack → Market analytics
* Union-Find → Identity graph
* Bloom Filter → Prevent DB overload
* Consistent Hashing → Scale horizontally
* CQRS → Optimize read-heavy workloads

---

