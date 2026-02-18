````md
---
# Node.js Internals — Event Loop, Microtasks, Thread Pool, Lifecycle (Interview-Grade Markdown)
---

## What is Node.js?

Node.js is a JavaScript runtime built on **Chrome’s V8 engine**. It uses an **event-driven, non-blocking I/O** model (via **libuv**) to build highly scalable network applications.

Key idea:

- **JavaScript execution is single-threaded** (one main call stack)
- **Async I/O is handled via the event loop + OS + libuv thread pool**

---

# Task Queues: Microtasks vs Macrotasks

## Queue Types

| Queue Type | Priority | Purpose |
| --- | --- | --- |
| **Microtask Queue** | Highest | Runs immediately after current execution |
| **Macrotask Queue** | Normal | Runs in event loop phases |

### Critical Rule (Node.js)
After every phase **and** after every callback, Node.js **empties the Microtask Queue** before moving forward.

That’s why Promises often “feel faster” than timers.

---

## Microtasks

Microtasks are **high-priority async jobs** that run:

1. After the current JS stack finishes
2. Before the event loop moves forward to the next phase

### Microtask APIs

| API | Type |
| --- | --- |
| `Promise.then()` | Microtask |
| `Promise.catch()` | Microtask |
| `queueMicrotask()` | Microtask |
| `process.nextTick()` | **Special microtask (Node-only, even higher priority)** |

---

## Macrotasks (Event Loop Phases)

Macrotasks are processed in **event loop phases**.

| API | Phase |
| --- | --- |
| `setTimeout()` | Timers |
| `setInterval()` | Timers |
| `setImmediate()` | Check |
| `fs.readFile()` | Poll |
| `http request` | Poll |

---

# Node.js Event Loop Phases (Macrotask Cycle)

| Phase | What Runs |
| --- | --- |
| **Timers** | `setTimeout`, `setInterval` |
| **Poll** | I/O callbacks, network, file system |
| **Check** | `setImmediate()` |
| **Close** | Cleanup callbacks |

> Between **every phase**, Node runs microtasks.

---

# Real Priority Order (Execution Order)

1. **Call Stack** (sync code)
2. **`process.nextTick()` queue**
3. **Microtasks** (`Promise`, `queueMicrotask`)
4. **Macrotasks** (Timers → Poll → Check → Close)
5. Repeat…

---

## Example: Execution Order

```js
console.log("Start");
setTimeout(() => console.log("Timeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
process.nextTick(() => console.log("NextTick"));
console.log("End");
````

### Output

```text
Start
End
NextTick
Promise
Timeout
```

---

# Node.js Special Case: `process.nextTick()`

Unlike browsers, Node.js has a **nextTick queue** that runs **before promise microtasks**.

This can cause **event loop starvation** if abused.

## Dangerous Example (Starvation)

```js
function blockLoop() {
  process.nextTick(blockLoop);
}
blockLoop();
```

This can freeze Node.js because macrotasks never run.

---

# Mental Flow Diagram

```text
Call Stack
  ↓
process.nextTick Queue
  ↓
Microtask Queue (Promises)
  ↓
Timers Phase (setTimeout)
  ↓
Poll Phase (I/O)
  ↓
Check Phase (setImmediate)
  ↓
Close Phase
  ↓
Repeat
```

---

# Interview One-Liner (Perfect Summary)

> “In Node.js, microtasks like Promises run immediately after the current call stack, but `process.nextTick()` runs even before them. Macrotasks are processed in event loop phases like timers, poll, and check. If `nextTick` is abused, it can starve the event loop and block I/O.”

---

# Node.js Architecture (Under the Hood)

```text
┌────────────┐
│ Your JS App│
└─────┬──────┘
      ↓
┌────────────┐
│   V8 Engine│  → Executes JavaScript
└─────┬──────┘
      ↓
┌────────────┐
│ Node APIs  │  → fs, http, crypto, timers
└─────┬──────┘
      ↓
┌────────────┐
│   libuv    │  → Event Loop + Thread Pool
└─────┬──────┘
      ↓
┌────────────┐
│ Operating  │  → OS, Network, Disk, DNS
│ System     │
└────────────┘
```

---

# Node.js Lifecycle — Big Picture

## Phase 1 — Process Startup

When you run:

```bash
node app.js
```

Internals:

* OS creates the process
* Node initializes runtime
* V8 boots
* Heap + Call Stack created
* Event Loop initialized
* Your JS file starts executing

---

## Phase 2 — Application Bootstrap

Top-level code runs first:

```js
const express = require("express");
const app = express();

app.listen(3000);
```

Happens here:

| Action            | Description                              |
| ----------------- | ---------------------------------------- |
| Module loading    | `require` / `import`                     |
| Memory allocation | Heap objects created                     |
| Sync code         | Runs on Call Stack                       |
| Async setup       | Timers/servers/DB connections registered |

> Event loop does **NOT** start until sync code finishes.

---

# Event Loop — Heart of the System

Phases (full view):

```text
┌──────────────┐
│ Timers       │ → setTimeout, setInterval
├──────────────┤
│ Pending I/O   │ → OS callbacks
├──────────────┤
│ Poll          │ → I/O (HTTP, FS, DB)
├──────────────┤
│ Check         │ → setImmediate
├──────────────┤
│ Close         │ → Cleanup
└──────────────┘
```

Between every phase:

* `process.nextTick()`
* `Promise.then()`

---

# Thread Pool — Hidden Multithreading (libuv)

Node is single-threaded for JS, but libuv maintains a **worker thread pool**.

Used for:

| Operation         | Where it runs                     |
| ----------------- | --------------------------------- |
| `fs`              | libuv thread pool                 |
| `crypto`          | libuv thread pool                 |
| `dns.lookup`      | libuv thread pool                 |
| `zlib`            | libuv thread pool                 |
| `http` networking | Mostly OS async (not thread pool) |

Default size:

```text
UV_THREADPOOL_SIZE = 4
```

---

# Memory System (V8)

## Stack

* Function calls
* Execution contexts
* Small and fast

## Heap

* Objects
* Closures
* Buffers
* Garbage collected by V8

GC model (high-level):

* Generational GC
* Mark & sweep (plus other optimizations)

---

# Final Interview Summary (End-to-End)

> “Node.js runs JavaScript on V8 and uses libuv to manage an event loop and a worker thread pool. Requests enter through the OS, are queued in the poll phase, executed on the JS call stack, and async work is offloaded to either the OS or the thread pool. Promises and `nextTick` run as microtasks between event loop phases, and the system achieves scalability by never blocking the main thread.”

```
```
