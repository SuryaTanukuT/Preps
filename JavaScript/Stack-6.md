JavaScript (JS) is a high-level, interpreted, dynamic programming language used to make applications interactive, asynchronous, and event-driven — on the web, servers, mobile apps, and even desktops.

JS is read and executed by a JavaScript engine (like V8) — no manual compilation needed.

Dynamic & Loosely Typed

You don’t need to declare variable types:
let x = 10;
x = "Hello"; // valid

JavaScript runs code on a call stack, manages async tasks using an event loop, and executes programs inside an engine like V8.

"JavaScript is a high-level, single-threaded, event-driven programming language that runs in browsers and servers, enabling interactive web applications and scalable backend systems using asynchronous, non-blocking execution."

The Big Picture

JavaScript runs inside a JavaScript Engine (like V8 in Chrome & Node.js).

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


A JavaScript Engine reads, compiles, optimizes, and executes your code.

| Part              | Job                                  |
| ----------------- | ------------------------------------ |
| Parser            | Converts code → AST                  |
| Interpreter       | Runs code line by line               |
| JIT Compiler      | Optimizes hot code into machine code |
| Garbage Collector | Frees unused memory                  |


Execution Context — How Code Starts

Every time JS runs:

It creates an Execution Context

| Type     | When Created            |
| -------- | ----------------------- |
| Global   | When program starts     |
| Function | When function is called |
| Eval     | Rare                    |


Two Phases per Context
1. Creation Phase

Memory allocated

Variables → undefined

Functions → full definition

this set

2. Execution Phase

Code runs line by line

Values assigned

Functions invoked

Call Stack — Where Code Executes
Definition

The Call Stack is a LIFO stack that tracks which function is currently running.

function a() {
  b();
}
function b() {
  console.log("Hi");
}
a();


Heap — Where Memory Lives
Definition

The Heap is a large memory area where objects, arrays, closures, and functions are stored.

What Goes Here

Objects {}

Arrays []

Functions

Closures

DOM references

Synchronous vs Asynchronous
Sync Code

Runs directly on the call stack

console.log("A");
console.log("B");

Async Code

Delegated to:

Browser Web APIs

Node.js APIs / libuv

Example:

setTimeout(() => console.log("Later"), 1000);


This runs outside the call stack.

Event Loop — The Traffic Controller
Definition

The Event Loop constantly checks:

Is the call stack empty?

Are there completed async tasks waiting?

If yes:

It moves callbacks into the call stack.

8️⃣ Task Queues
🟡 Microtask Queue (High Priority)

Runs first

Promise.then()

queueMicrotask()

process.nextTick() (Node.js — even higher)

Macrotask Queue (Phases)

Runs after microtasks

setTimeout

I/O

setImmediate

| Feature     | Browser           | Node.js       |
| ----------- | ----------------- | ------------- |
| Engine      | V8 / SpiderMonkey | V8            |
| Async APIs  | Web APIs          | libuv         |
| Event Loop  | Browser-managed   | libuv-managed |
| Thread Pool | ❌                 | ✅             |


Memory Management — Garbage Collection

V8 uses:

Mark & Sweep

Generational GC

Unreachable objects are cleaned from the heap automatically.

"JavaScript runs inside an engine like V8, which parses and JIT-compiles code, executes it on a call stack, and stores objects in the heap. Asynchronous operations are delegated to Web APIs or libuv, and once completed, their callbacks are queued and pushed back onto the call stack by the event loop, with microtasks like Promises running before macrotasks."


Got it — let’s build a **clear mental picture** of how JavaScript runs, where **hoisting happens**, how **functions work**, and what **`undefined` vs `not defined`** really mean — *step by step, like a system diagram in words*.

---

# 🧠 JavaScript Execution — The Big Picture

Think of JS execution as **2 phases per file/function**:

```text
1) Creation Phase (Memory Setup / Hoisting)
2) Execution Phase (Run Code Line by Line)
```

---

## 🔁 Overall Flow

```text
JS Code
  ↓
JavaScript Engine (V8)
  ↓
Create Global Execution Context
  ↓
Creation Phase (Hoisting)
  ↓
Execution Phase (Run Code)
  ↓
Call Stack manages function calls
```

---

# 1️⃣ Execution Context — What It Is

> An **Execution Context (EC)** is an environment where JavaScript code is evaluated and executed.

Each EC contains:

* **Memory Space (Variable Environment)**
* **Scope Chain**
* **`this` value**

---

## Types of Execution Contexts

| Type            | Created When                |
| --------------- | --------------------------- |
| **Global EC**   | JS file starts              |
| **Function EC** | A function is called        |
| **Eval EC**     | `eval()` runs (rare, avoid) |

---

# 2️⃣ Creation Phase (Hoisting Happens Here)

Before a single line runs, JS **scans your code and allocates memory**

### What Happens:

| Code Type               | Stored As                  |
| ----------------------- | -------------------------- |
| `var`                   | `undefined`                |
| `let` / `const`         | Uninitialized (TDZ)        |
| `function declarations` | Full function              |
| `this`                  | Set (window/global/object) |

---

## Visual Model

```text
Code:
  var x = 10;
  function foo() {}
  let y = 20;

Memory After Creation Phase:
  x → undefined
  foo → function foo() {}
  y → <uninitialized> (TDZ)
```

---

# 3️⃣ Execution Phase

Now JS **runs code line by line** and assigns values.

```js
var x = 10;  // x becomes 10
let y = 20;  // y becomes 20
foo();       // function runs
```

---

# 4️⃣ Hoisting — Clear Definition

> **Hoisting is JavaScript’s behavior of moving declarations (not values) to the top of their scope during the creation phase.**

---

# Types of Hoisting (Important for Interviews)

---

## 1️⃣ `var` Hoisting

### Behavior:

* Hoisted
* Initialized as `undefined`

```js
console.log(a); // undefined
var a = 10;
```

### Memory Phase:

```text
a → undefined
```

---

## 2️⃣ `let` Hoisting (TDZ)

### Behavior:

* Hoisted
* NOT initialized
* Exists in **Temporal Dead Zone**

```js
console.log(b); // ReferenceError
let b = 20;
```

---

## 3️⃣ `const` Hoisting (TDZ)

Same as `let`, but:

* Must be initialized at declaration

```js
const c; // ❌ SyntaxError
```

---

## 4️⃣ Function Declaration Hoisting

### Fully Hoisted

```js
hello(); // Works

function hello() {
  console.log("Hi");
}
```

Memory:

```text
hello → function
```

---

## 5️⃣ Function Expression Hoisting

```js
sayHi(); // ❌ TypeError
var sayHi = function() {
  console.log("Hi");
};
```

Memory:

```text
sayHi → undefined
```

---

# 5️⃣ Temporal Dead Zone (TDZ) — Simple Meaning

> The **TDZ** is the time between hoisting and initialization where `let` and `const` **exist but cannot be accessed**

---

# 6️⃣ How Functions Work in JavaScript

---

## Function Execution Model

When a function is called:

```text
New Execution Context is created
  ↓
Memory Phase (params + local vars hoisted)
  ↓
Execution Phase (code runs)
  ↓
Return value sent back
  ↓
Context destroyed
```

---

## Example

```js
function add(a, b) {
  var sum = a + b;
  return sum;
}
add(2, 3);
```

### Memory Phase

```text
a → 2
b → 3
sum → undefined
```

---

# 7️⃣ Types of Functions in JS

---

## 1️⃣ Function Declaration

```js
function foo() {}
```

* Hoisted
* Has name
* Can be called before declaration

---

## 2️⃣ Function Expression

```js
const bar = function() {};
```

* Not hoisted as function
* Stored as variable

---

## 3️⃣ Arrow Function

```js
const baz = () => {};
```

* No `this`
* No `arguments`
* Not hoisted

---

## 4️⃣ IIFE

```js
(function() {
  console.log("Run once");
})();
```

---

## 5️⃣ Constructor Function

```js
function User(name) {
  this.name = name;
}
```

---

# 8️⃣ Functions Are Objects (Important Concept)

In JS:

> **Functions are first-class objects**

This means:

* Can be stored in variables
* Passed as arguments
* Returned from functions
* Have properties & methods

---

# 9️⃣ Methods on Functions

---

## 🔹 `.call()`

Invoke with custom `this`

```js
fn.call(obj, arg1, arg2);
```

---

## 🔹 `.apply()`

Same, but args as array

```js
fn.apply(obj, [arg1, arg2]);
```

---

## 🔹 `.bind()`

Returns new function with bound `this`

```js
const newFn = fn.bind(obj);
```

---

## 🔹 `.toString()`

Returns function source

---

## 🔹 `.length`

Number of parameters

---

## 🔹 `.name`

Function name

---

# 🔟 `undefined` vs `not defined` (Very Important)

---

## 🔹 `undefined`

> Variable exists in memory but has no value yet

```js
var x;
console.log(x); // undefined
```

---

## 🔹 `not defined` (ReferenceError)

> Variable does NOT exist in memory

```js
console.log(y); // ReferenceError: y is not defined
```

---

## Mental Model

| Term        | Memory Exists? | Value Assigned? |
| ----------- | -------------- | --------------- |
| undefined   | ✅              | ❌               |
| not defined | ❌              | ❌               |

---

# 🔁 Full Execution Example (All Concepts Together)

```js
console.log(a); // undefined
foo();          // works
bar();          // error

var a = 10;

function foo() {
  console.log("Hello");
}

var bar = function() {
  console.log("Hi");
};
```

---

## Creation Phase Memory

```text
a → undefined
foo → function
bar → undefined
```

---

## Execution Phase

```text
console.log(a) → undefined
foo() → runs
bar() → TypeError (bar is undefined)
```

---

# 🏆 Senior Interview One-Liner

> "JavaScript runs in two phases: a creation phase where execution contexts are created and declarations are hoisted into memory, and an execution phase where code runs line by line on the call stack. `var` is hoisted and initialized as undefined, `let` and `const` are hoisted into the Temporal Dead Zone, and function declarations are fully hoisted. Functions create their own execution contexts, and `undefined` means a variable exists in memory but has no value, while `not defined` means it doesn’t exist at all."

---

# 🧠 Memory Trick

> **Hoisted ≠ Initialized**
> **Undefined = Exists**
> **Not Defined = Doesn’t Exist**

---
