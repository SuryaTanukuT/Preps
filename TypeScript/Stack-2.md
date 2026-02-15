# 📘 Data Structures in TypeScript

*(With Definitions + When & Why to Use — Interview Ready)*

TypeScript uses JavaScript data structures but adds **type safety, generics, and better modeling** for scalable systems.

Below is each data structure with:

* ✅ Definition
* 📌 Context / When to Use
* ⚡ Complexity (important for interviews)
* 🧠 Real-world usage

---

# 1️⃣ Primitive Types

## ✅ Definition

Basic single-value data types: `number`, `string`, `boolean`, `null`, `undefined`, `symbol`, `bigint`.

```ts
let age: number = 30;
let name: string = "Surya";
```

## 📌 Context

Used as building blocks for all other structures.

## 🧠 Real-world

* API request parameters
* Config flags
* IDs and counters

---

# 2️⃣ Array

## ✅ Definition

An ordered, index-based collection of elements.

```ts
let nums: number[] = [1, 2, 3];
```

## 📌 Context

Use when:

* Order matters
* You need fast index access

## ⚡ Complexity

* Access by index → O(1)
* Insert/Delete middle → O(n)

## 🧠 Real-world

* Product listings
* Pagination results
* Event logs

---

# 3️⃣ Tuple

## ✅ Definition

A fixed-length array with fixed types at each index.

```ts
let user: [number, string] = [1, "Surya"];
```

## 📌 Context

Use when:

* Returning multiple structured values
* Position has meaning

## 🧠 Real-world

* Returning `[data, error]`
* Database row result

---

# 4️⃣ Enum

## ✅ Definition

A set of named constant values.

```ts
enum Status {
  SUCCESS = "success",
  FAILED = "failed"
}
```

## 📌 Context

Use when:

* You need fixed allowed values
* Business states

## 🧠 Real-world

* Order status
* Payment status
* User roles

---

# 5️⃣ Object (Typed Object)

## ✅ Definition

A key-value structure with defined shape.

```ts
type User = {
  id: number;
  name: string;
};
```

## 📌 Context

Use when:

* Representing structured entities

## ⚡ Complexity

* Property access → O(1)

## 🧠 Real-world

* DTOs
* API responses
* Database models

---

# 6️⃣ Interface

## ✅ Definition

A contract that defines object structure.

```ts
interface Product {
  id: number;
  name: string;
}
```

## 📌 Context

Use when:

* Designing scalable systems
* Enforcing contracts across layers

## 🧠 Real-world

* Microservice contracts
* Clean Architecture domain models

---

# 7️⃣ Map

## ✅ Definition

An ordered key-value data structure that allows any type as key.

```ts
const map = new Map<string, number>();
```

## 📌 Context

Use when:

* Keys are dynamic
* Frequent insert/delete
* Key type is not string

## ⚡ Complexity

* get/set → O(1) average

## 🧠 Real-world

* Caching
* Frequency counting
* Graph adjacency list

---

# 8️⃣ Set

## ✅ Definition

A collection of unique values.

```ts
const set = new Set<number>();
```

## 📌 Context

Use when:

* Removing duplicates
* Fast membership check

## ⚡ Complexity

* has → O(1)

## 🧠 Real-world

* Unique user sessions
* Tag filtering
* Prevent duplicate processing

---

# 9️⃣ Stack (LIFO)

## ✅ Definition

Last-In-First-Out structure.

```ts
class Stack<T> {
  private items: T[] = [];
  push(item: T) { this.items.push(item); }
  pop() { return this.items.pop(); }
}
```

## 📌 Context

Use when:

* Backtracking
* Undo/Redo
* DFS

## ⚡ Complexity

* push/pop → O(1)

## 🧠 Real-world

* Browser history
* Expression evaluation

---

# 🔟 Queue (FIFO)

## ✅ Definition

First-In-First-Out structure.

```ts
class Queue<T> {
  private items: T[] = [];
  enqueue(item: T) { this.items.push(item); }
  dequeue() { return this.items.shift(); }
}
```

## 📌 Context

Use when:

* Scheduling
* Order processing
* BFS

## ⚡ Complexity

* enqueue → O(1)
* dequeue (array shift) → O(n)

⚠️ For high performance → use linked list

## 🧠 Real-world

* Job queues
* Notification systems
* Message brokers

---

# 1️⃣1️⃣ Linked List

## ✅ Definition

A sequence of nodes where each node points to the next.

```ts
class Node<T> {
  constructor(public value: T, public next: Node<T> | null = null) {}
}
```

## 📌 Context

Use when:

* Frequent insert/delete
* Dynamic size

## ⚡ Complexity

* Insert/Delete at head → O(1)
* Search → O(n)

## 🧠 Real-world

* Low-level memory systems
* Implementing queues efficiently

---

# 1️⃣2️⃣ Hash Table

## ✅ Definition

A structure mapping keys to values using hashing.

In TS, implemented via:

* Object
* Map

## 📌 Context

Use when:

* Fast lookups needed
* Key-value relationship

## ⚡ Complexity

* Average O(1)

## 🧠 Real-world

* Caching
* Authentication token storage
* Frequency counters

---

# 1️⃣3️⃣ Tree

## ✅ Definition

A hierarchical structure with parent-child relationships.

```ts
class TreeNode<T> {
  constructor(
    public value: T,
    public left: TreeNode<T> | null = null,
    public right: TreeNode<T> | null = null
  ) {}
}
```

## 📌 Context

Use when:

* Hierarchical data
* Sorted structures

## ⚡ Complexity

* Balanced BST search → O(log n)

## 🧠 Real-world

* File systems
* DOM tree
* Category hierarchies

---

# 1️⃣4️⃣ Graph

## ✅ Definition

A collection of nodes connected by edges.

```ts
class Graph<T> {
  private adj = new Map<T, T[]>();
}
```

## 📌 Context

Use when:

* Many-to-many relationships
* Network modeling

## 🧠 Real-world

* Social networks
* Routing systems
* Dependency graphs

---

# 🔥 Advanced TypeScript Utility Structures

## Record

### ✅ Definition

Creates a typed object with fixed key type.

```ts
const roles: Record<string, string> = {};
```

## Partial

### ✅ Definition

Makes all properties optional.

```ts
type UpdateUser = Partial<User>;
```

## Pick / Omit

### ✅ Definition

Select or remove specific fields.

```ts
type UserName = Pick<User, "name">;
type WithoutId = Omit<User, "id">;
```

---

# 🎯 Senior Interview Perspective

Be ready to answer:

| Question                     | Expected Depth                                     |
| ---------------------------- | -------------------------------------------------- |
| Map vs Object?               | Key type flexibility, iteration order, performance |
| Set vs Array?                | Membership O(1) vs O(n)                            |
| Array vs LinkedList?         | Cache locality vs dynamic insertion                |
| When to use Tree?            | Sorted retrieval                                   |
| Why generics?                | Reusable type-safe abstractions                    |
| How would you implement LRU? | Map + Doubly LinkedList                            |

---

# 🧠 Real Engineering Context (Your Level)

In scalable backend systems (Node/NestJS):

* `Map` → in-memory caching
* `Set` → deduplication of events
* `Queue` → job processing
* `Graph` → dependency resolution
* `Tree` → hierarchical RBAC
* `Record` → configuration mapping
* `Partial` → PATCH APIs

---
---

# 🔥 1️⃣ LRU Cache Implementation (O(1))

### ✅ Definition

LRU (Least Recently Used) evicts the least recently accessed item when capacity is exceeded.

### 🧠 Why?

Used in:

* API caching
* DB query caching
* Token/session caching

### 💡 Approach

Use:

* `Map` → O(1) lookup
* Doubly Linked List → O(1) move/remove

---

## TypeScript Implementation

```ts
class Node<K, V> {
  constructor(
    public key: K,
    public value: V,
    public prev: Node<K, V> | null = null,
    public next: Node<K, V> | null = null
  ) {}
}

class LRUCache<K, V> {
  private capacity: number;
  private map = new Map<K, Node<K, V>>();
  private head: Node<K, V> | null = null;
  private tail: Node<K, V> | null = null;

  constructor(capacity: number) {
    this.capacity = capacity;
  }

  get(key: K): V | undefined {
    const node = this.map.get(key);
    if (!node) return undefined;

    this.moveToHead(node);
    return node.value;
  }

  put(key: K, value: V): void {
    let node = this.map.get(key);

    if (node) {
      node.value = value;
      this.moveToHead(node);
    } else {
      node = new Node(key, value);
      this.map.set(key, node);
      this.addToHead(node);

      if (this.map.size > this.capacity) {
        this.removeTail();
      }
    }
  }

  private addToHead(node: Node<K, V>) {
    node.next = this.head;
    if (this.head) this.head.prev = node;
    this.head = node;
    if (!this.tail) this.tail = node;
  }

  private moveToHead(node: Node<K, V>) {
    this.removeNode(node);
    this.addToHead(node);
  }

  private removeNode(node: Node<K, V>) {
    if (node.prev) node.prev.next = node.next;
    if (node.next) node.next.prev = node.prev;
    if (node === this.head) this.head = node.next;
    if (node === this.tail) this.tail = node.prev;
  }

  private removeTail() {
    if (!this.tail) return;
    this.map.delete(this.tail.key);
    this.removeNode(this.tail);
  }
}
```

### ⏱ Complexity

* get → O(1)
* put → O(1)

---

# 🔥 2️⃣ Trie Implementation

### ✅ Definition

Tree structure used for prefix-based searching.

Used in:

* Autocomplete
* Search engines
* Spell check

---

```ts
class TrieNode {
  children: Map<string, TrieNode> = new Map();
  isEnd: boolean = false;
}

class Trie {
  root = new TrieNode();

  insert(word: string) {
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) {
        node.children.set(char, new TrieNode());
      }
      node = node.children.get(char)!;
    }
    node.isEnd = true;
  }

  search(word: string): boolean {
    let node = this.root;
    for (const char of word) {
      if (!node.children.has(char)) return false;
      node = node.children.get(char)!;
    }
    return node.isEnd;
  }
}
```

---

# 🔥 3️⃣ Heap / Priority Queue

### ✅ Definition

Heap is a binary tree where:

* Min-heap → smallest on top
* Max-heap → largest on top

Used in:

* Task scheduling
* Dijkstra
* Rate limiting systems

---

```ts
class MinHeap {
  private heap: number[] = [];

  insert(value: number) {
    this.heap.push(value);
    this.bubbleUp();
  }

  extractMin(): number | undefined {
    if (this.heap.length === 0) return undefined;
    const min = this.heap[0];
    const end = this.heap.pop()!;
    if (this.heap.length > 0) {
      this.heap[0] = end;
      this.bubbleDown();
    }
    return min;
  }

  private bubbleUp() {
    let index = this.heap.length - 1;
    while (index > 0) {
      let parent = Math.floor((index - 1) / 2);
      if (this.heap[parent] <= this.heap[index]) break;
      [this.heap[parent], this.heap[index]] =
        [this.heap[index], this.heap[parent]];
      index = parent;
    }
  }

  private bubbleDown() {
    let index = 0;
    const length = this.heap.length;

    while (true) {
      let left = 2 * index + 1;
      let right = 2 * index + 2;
      let smallest = index;

      if (left < length && this.heap[left] < this.heap[smallest])
        smallest = left;
      if (right < length && this.heap[right] < this.heap[smallest])
        smallest = right;

      if (smallest === index) break;

      [this.heap[index], this.heap[smallest]] =
        [this.heap[smallest], this.heap[index]];

      index = smallest;
    }
  }
}
```

### ⏱ Complexity

* Insert → O(log n)
* Extract → O(log n)

---

# 🔥 4️⃣ Why Map over Object?

| Map                              | Object                      |
| -------------------------------- | --------------------------- |
| Any key type                     | String/symbol only          |
| Ordered                          | Not guaranteed historically |
| Built for frequent insert/delete | Not optimized               |
| Better iteration                 | Needs Object.keys           |

**Senior answer:**

> Use Map when keys are dynamic, not just strings, and when you need predictable iteration + better performance for frequent mutations.

---

# 🔥 5️⃣ When to use Set?

Use when:

* Need uniqueness
* Membership check O(1)
* Removing duplicates

---

# 🔥 6️⃣ Array vs LinkedList

| Feature         | Array | LinkedList |
| --------------- | ----- | ---------- |
| Index access    | O(1)  | O(n)       |
| Insert middle   | O(n)  | O(1)*      |
| Memory locality | High  | Low        |
| Cache friendly  | Yes   | No         |

Senior answer:

> Arrays are better for CPU cache performance. Linked lists are useful for frequent insert/delete but rarely used in high-level backend code.

---

# 🔥 7️⃣ Time Complexity Cheat

| DS    | Access   | Insert   | Delete   |
| ----- | -------- | -------- | -------- |
| Array | O(1)     | O(n)     | O(n)     |
| Map   | O(1)     | O(1)     | O(1)     |
| Set   | O(1)     | O(1)     | O(1)     |
| Heap  | O(log n) | O(log n) | O(log n) |
| BST   | O(log n) | O(log n) | O(log n) |

---

# 🔥 8️⃣ BFS / DFS Template

### BFS

```ts
function bfs(graph: Map<number, number[]>, start: number) {
  const visited = new Set<number>();
  const queue: number[] = [start];

  while (queue.length) {
    const node = queue.shift()!;
    if (visited.has(node)) continue;

    visited.add(node);
    console.log(node);

    for (const neighbor of graph.get(node) || []) {
      queue.push(neighbor);
    }
  }
}
```

### DFS

```ts
function dfs(graph: Map<number, number[]>, node: number, visited = new Set<number>()) {
  if (visited.has(node)) return;

  visited.add(node);
  console.log(node);

  for (const neighbor of graph.get(node) || []) {
    dfs(graph, neighbor, visited);
  }
}
```

---

# 🔥 9️⃣ Generics Improve Reusability

Without generics:

```ts
class Stack {
  items: any[];
}
```

With generics:

```ts
class Stack<T> {
  items: T[];
}
```

✔ Type-safe
✔ Reusable
✔ Compile-time errors

---

# 🔥 1️⃣0️⃣ Immutability Improves Predictability

Why?

* No side effects
* Easier debugging
* Better concurrency handling
* Works well with Redux / Event sourcing

Example:

```ts
const newArr = [...oldArr, 5];
```

Instead of mutating:

```ts
oldArr.push(5);
```

---

# 🔥 1️⃣1️⃣ Real Interview Problems

* Two Sum (Map)
* LRU Cache
* Detect cycle in graph
* Merge intervals
* Top K elements (Heap)
* Implement autocomplete (Trie)
* Sliding window maximum
* Design rate limiter

---

# 🔥 1️⃣2️⃣ System Design Perspective

| DS    | System Usage            |
| ----- | ----------------------- |
| Map   | In-memory cache         |
| Set   | Dedup event processing  |
| Heap  | Task scheduler          |
| Trie  | Search suggestion       |
| Graph | Microservice dependency |
| Queue | Async processing        |
| LRU   | Redis local cache       |

---

# 🔥 1️⃣3️⃣ Advanced DSA Patterns (Senior Must Know)

* Sliding Window
* Two Pointers
* Fast/Slow Pointer
* Monotonic Stack
* Backtracking
* Topological Sort
* Union-Find
* Prefix Sum
* Binary Search on Answer
* Segment Tree (advanced)
* Bit Manipulation
* Dynamic Programming patterns

---