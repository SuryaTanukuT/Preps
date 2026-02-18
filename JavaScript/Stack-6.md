````md
---
# JavaScript Runtime & Execution Model — Interview-Grade Guide (Markdown)
---

## What is JavaScript (JS)?

JavaScript (JS) is a **high-level, dynamic, loosely typed** programming language used to build **interactive, asynchronous, event-driven** applications across:

- Web (browsers)
- Servers (Node.js)
- Mobile apps
- Desktop apps

JS runs inside a **JavaScript Engine** (e.g., **V8**) which **reads, compiles, optimizes, and executes** your code.

> JS is typically *JIT-compiled under the hood* (engine compiles hot code to machine code), but from a developer point-of-view it feels “interpreted” because you don’t do manual compilation steps.

---

## Dynamic & Loosely Typed

You don’t declare types explicitly:

```js
let x = 10;
x = "Hello"; // valid
````

---

# Big Picture: How JS Runs

JavaScript runs inside a **JavaScript Engine** (like **V8** in Chrome and Node.js).

```text
Your Code
   ↓
JavaScript Engine (V8)
   ↓
Call Stack  ←→  Heap (Memory)
   ↓
Web APIs / Node APIs
   ↓
Event Loop
   ↓
Back to Call Stack
```

---

# What a JavaScript Engine Does

A JavaScript engine reads your source code and turns it into executable work.

| Part              | Job                                        |
| ----------------- | ------------------------------------------ |
| Parser            | Converts code → AST (Abstract Syntax Tree) |
| Interpreter       | Executes bytecode / baseline execution     |
| JIT Compiler      | Optimizes hot code into machine code       |
| Garbage Collector | Frees unused memory                        |

---

# Execution Context (EC) — How Code Starts

> An **Execution Context** is the environment where JavaScript code is prepared and executed.

### Types of Execution Context

| Type        | When Created              |
| ----------- | ------------------------- |
| Global EC   | When program starts       |
| Function EC | When a function is called |
| Eval EC     | Rare (avoid `eval`)       |

---

## Two Phases in Each Execution Context

### 1) Creation Phase (Memory Setup / Hoisting)

What happens:

* Memory is allocated
* Declarations are registered
* `this` is set
* Scope chain is prepared

**Hoisting happens here.**

| Declaration type     | What gets created in memory (creation phase)       |
| -------------------- | -------------------------------------------------- |
| `var`                | Created and initialized to `undefined`             |
| `let` / `const`      | Created but **uninitialized** (Temporal Dead Zone) |
| Function declaration | Full function definition stored                    |
| `this`               | Bound (global or function-specific rules)          |

---

### 2) Execution Phase (Run Line by Line)

Now the engine executes statements:

* Values are assigned
* Functions are called
* Expressions are evaluated

---

# Call Stack — Where Code Executes

> The **Call Stack** is a LIFO stack that tracks which function is currently executing.

```js
function a() {
  b();
}
function b() {
  console.log("Hi");
}
a();
```

---

# Heap — Where Memory Lives

> The **Heap** is a large memory region where objects and reference types live.

Typically stored in heap:

* Objects `{ }`
* Arrays `[ ]`
* Functions (as objects)
* Closures
* DOM references (browser)

---

# Synchronous vs Asynchronous

## Synchronous (Sync) Code

Runs directly on the call stack, in order:

```js
console.log("A");
console.log("B");
```

## Asynchronous (Async) Code

Delegated to:

* **Web APIs** (browser)
* **Node APIs / libuv** (Node.js)

Example:

```js
setTimeout(() => console.log("Later"), 1000);
```

That callback runs **outside** the call stack first, then comes back later via queues.

---

# Event Loop — The Traffic Controller

> The **Event Loop** coordinates when async callbacks are pushed back onto the call stack.

It constantly checks:

1. Is the call stack empty?
2. Are completed async tasks waiting in queues?

If yes → it moves callbacks into the call stack.

---

# Task Queues (Microtasks vs Macrotasks)

## Microtask Queue (High Priority)

Runs **before** macrotasks:

* `Promise.then()`
* `queueMicrotask()`
* `process.nextTick()` *(Node-only, even higher priority)*

## Macrotask Queue (Event Loop Phases)

Runs **after** microtasks:

* `setTimeout` / `setInterval`
* I/O callbacks
* `setImmediate` (Node)

---

# Browser vs Node.js (Key Differences)

| Feature     | Browser                            | Node.js               |
| ----------- | ---------------------------------- | --------------------- |
| Engine      | V8 / SpiderMonkey / JavaScriptCore | V8                    |
| Async APIs  | Web APIs                           | libuv + Node APIs     |
| Event Loop  | Browser-managed                    | libuv-managed         |
| Thread Pool | ❌                                  | ✅ (libuv thread pool) |

---

# Memory Management — Garbage Collection

V8 automatically frees heap memory using GC strategies such as:

* **Mark & Sweep**
* **Generational GC**

> Unreachable objects (no references) are eligible for cleanup.

---

# Interview-Ready Summary

> “JavaScript runs inside an engine like V8, which parses and JIT-compiles code, executes it on a call stack, and stores objects in the heap. Asynchronous operations are delegated to Web APIs or libuv, and once completed, their callbacks are queued and pushed back onto the call stack by the event loop—where microtasks like Promises run before macrotasks.”

---

# JavaScript Execution: Hoisting, Functions, TDZ, `undefined` vs `not defined`

## The Big Picture

Every file/function runs in **two phases**:

```text
1) Creation Phase (Memory Setup / Hoisting)
2) Execution Phase (Run Code Line by Line)
```

---

## Execution Context Types

| Type        | Created When         |
| ----------- | -------------------- |
| Global EC   | Program starts       |
| Function EC | Function is called   |
| Eval EC     | `eval()` runs (rare) |

---

## Hoisting — Clear Definition

> **Hoisting** means declarations are registered in memory during the **creation phase** (values are not “moved”; memory is prepared).

### What Hoists How?

| Code Type            | Hoisted?                 | Initialized during creation? |
| -------------------- | ------------------------ | ---------------------------- |
| `var`                | ✅                        | ✅ `undefined`                |
| `let`                | ✅                        | ❌ (TDZ)                      |
| `const`              | ✅                        | ❌ (TDZ + must initialize)    |
| Function declaration | ✅                        | ✅ full function              |
| Function expression  | depends on var/let/const | depends                      |

---

## `var` Hoisting

```js
console.log(a); // undefined
var a = 10;
```

---

## `let` / `const` Hoisting (TDZ)

```js
console.log(b); // ReferenceError
let b = 20;
```

> **TDZ (Temporal Dead Zone)** = time between hoisting and initialization where the variable exists but cannot be accessed.

---

## Function Declaration Hoisting

```js
hello(); // works

function hello() {
  console.log("Hi");
}
```

---

## Function Expression Hoisting (Common Trap)

```js
sayHi(); // TypeError: sayHi is not a function (because sayHi is undefined)
var sayHi = function () {
  console.log("Hi");
};
```

---

# Functions: How They Execute

When a function is called:

```text
New Function Execution Context created
  ↓
Creation Phase (params + locals set up)
  ↓
Execution Phase (runs)
  ↓
Return
  ↓
Context destroyed
```

Example:

```js
function add(a, b) {
  var sum = a + b;
  return sum;
}

add(2, 3);
```

---

# `undefined` vs `not defined`

## `undefined`

> Variable exists in memory but has no value assigned yet.

```js
var x;
console.log(x); // undefined
```

## `not defined` (ReferenceError)

> Variable does not exist in the current scope/memory.

```js
console.log(y); // ReferenceError: y is not defined
```

### Mental Model

| Term          | Memory Exists? | Value Assigned? |
| ------------- | -------------- | --------------- |
| `undefined`   | ✅              | ❌               |
| `not defined` | ❌              | ❌               |

---

# Full Example (All Concepts Together)

```js
console.log(a); // undefined
foo();          // works
bar();          // error

var a = 10;

function foo() {
  console.log("Hello");
}

var bar = function () {
  console.log("Hi");
};
```

## Creation Phase Memory

```text
a   → undefined
foo → function
bar → undefined
```

## Execution Phase Outcome

```text
console.log(a) → undefined
foo() → runs
bar() → TypeError (bar is undefined at that moment)
```

---

# Senior Interview One-Liner

> “JavaScript runs in two phases: a creation phase where execution contexts are created and declarations are hoisted into memory, and an execution phase where code runs line by line on the call stack. `var` is hoisted and initialized as undefined, `let`/`const` are hoisted into the TDZ, and function declarations are fully hoisted. `undefined` means the variable exists but has no value, while `not defined` means it doesn’t exist in scope.”

---

# Memory Trick

> **Hoisted ≠ Initialized**
> **Undefined = Exists**
> **Not defined = Doesn’t Exist**

```
```
