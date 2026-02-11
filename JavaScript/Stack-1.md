# 🧠 JavaScript Deep Dive — Interview Master Guide

> Complete reference for JS internals, closures, scope, async, OOP, and V8 internals.  
> Every topic: **What it is → Where it applies → Types/Methods → Why it exists → Mental Model → Examples → Edge Cases → Interview Questions**

---

## Table of Contents

1. [Temporal Dead Zone (TDZ)](#1-temporal-dead-zone-tdz)
2. [var vs let vs const](#2-var-vs-let-vs-const)
3. [Scope](#3-scope)
4. [Hoisting](#4-hoisting)
5. [Scope Chain](#5-scope-chain)
6. [Block Scope](#6-block-scope)
7. [Shadowing & Illegal Shadowing](#7-shadowing--illegal-shadowing)
8. [Closures (Deep)](#8-closures-deep-explanation)
9. [OOP in JavaScript](#9-oop-in-javascript)
10. [Factory Functions](#10-factory-functions)
11. [Memoization / Caching](#11-memoizationcaching)
12. [Event Handlers](#12-event-handlers)
13. [Async Callbacks](#13-async-callbacks)
14. [Module Patterns](#14-module-patterns)
15. [setTimeout + Closures](#15-settimeout--closures)
16. [IIFE & IIFE + Closures](#16-iife--iife--closures)
17. [First-Class Functions](#17-first-class-functions)
18. [Timers in JS](#18-timers-in-js)
19. [Higher-Order Functions (HOF)](#19-higher-order-functions-hof)
20. [map, filter, reduce (Deep)](#20-map-filter-reduce-deep)
21. [this Keyword](#21-this-keyword)
22. [Arrow Functions](#22-arrow-functions)
23. [V8 Internals: Lexical Scope, Scope Tree, Deopt Killers](#23-v8-internals)
24. [Garbage Collection (GC)](#24-garbage-collection-gc)

---

## 1. Temporal Dead Zone (TDZ)

### What It Is

The **Temporal Dead Zone** is the period between the *start of a block scope* and the point where a `let` or `const` variable is declared and initialized. During this period, accessing the variable throws a `ReferenceError`.

### Where It Applies

- `let` declarations
- `const` declarations
- `class` declarations (yes, classes have TDZ too!)
- Default function parameter expressions (partially)

### Where It Does NOT Apply

- `var` declarations (hoisted and initialized to `undefined`)
- Function declarations (fully hoisted with their body)

### Why TDZ Exists

TDZ was designed intentionally by TC39 to **catch programmer errors**. In pre-ES6 JavaScript, accessing a `var` before its declaration silently returned `undefined`, which caused subtle, hard-to-debug bugs. TDZ makes such errors loud and explicit.

### Mental Model

Think of a block like a room. When you enter the room (start the block), `let`/`const` variables are **reserved but roped off**. You know they exist, but you can't touch them until the declaration line is reached. Crossing the rope throws an error.

```
Block start ─────────────────────────────────────► Declaration line
   │                                                      │
   │◄──────────────── TDZ Zone ──────────────────────────►│
   │  Accessing variable here → ReferenceError            │
                                                          ▼
                                              Variable is now accessible
```

### Examples

```javascript
// ✅ var — no TDZ, silently undefined
console.log(x); // undefined (NOT an error)
var x = 5;

// ❌ let — TDZ throws ReferenceError
console.log(y); // ReferenceError: Cannot access 'y' before initialization
let y = 10;

// ❌ const — same as let
console.log(z); // ReferenceError
const z = 20;

// ❌ class — TDZ applies
const obj = new MyClass(); // ReferenceError
class MyClass {}
```

```javascript
// TDZ in block scope
{
  // TDZ for 'a' starts here
  console.log(a); // ❌ ReferenceError
  let a = 42;     // TDZ ends here
  console.log(a); // ✅ 42
}
```

```javascript
// TDZ in function scope
function test() {
  console.log(typeof val); // ❌ ReferenceError (NOT "undefined"!)
  let val = 5;
}
test();
```

> ⚠️ **Tricky:** `typeof` usually never throws — EXCEPT during TDZ for `let`/`const`.

```javascript
// TDZ with closures
let x = 1;
function foo() {
  console.log(x); // ❌ ReferenceError — inner x is in TDZ even though outer x=1
  let x = 2;
}
foo();
```

### Use Cases and Best Practices

- Always declare `let`/`const` at the **top of their block** to avoid TDZ surprises
- TDZ is a feature, not a bug — it forces disciplined variable usage
- Helps identify circular dependencies in modules (module loading TDZ)

### Real Interview Questions

**Q1:** What is the output?
```javascript
console.log(typeof a); // ?
let a = 5;
```
**Answer:** `ReferenceError` — `typeof` does NOT bypass TDZ for `let`/`const`.

**Q2:** What is the output?
```javascript
let a = 10;
{
  console.log(a); // ?
  let a = 20;
}
```
**Answer:** `ReferenceError` — the inner `let a` creates a new TDZ in the block, shadowing the outer `a`.

**Q3:** Does TDZ apply to function parameters?
```javascript
function foo(a = b, b = 2) {
  return a + b;
}
foo(); // ?
```
**Answer:** `ReferenceError` — `b` is in TDZ when `a`'s default is evaluated left-to-right.

**Q4:** Does this throw?
```javascript
class Foo {}
const f = new Foo(); // ?
```
**Answer:** No — the `class` declaration comes first. But reversing the order throws `ReferenceError`.

---

## 2. `var` vs `let` vs `const`

### What They Are

Three ways to declare variables in JavaScript with different scoping, hoisting, and mutability behaviors.

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function/Global | Block | Block |
| Hoisting | Yes (→ `undefined`) | Yes (→ TDZ) | Yes (→ TDZ) |
| Re-declaration | ✅ Allowed | ❌ Same scope | ❌ Same scope |
| Re-assignment | ✅ Allowed | ✅ Allowed | ❌ Not allowed |
| Global property | Yes (`window.x`) | No | No |
| TDZ | No | Yes | Yes |

### `var` — Function-Scoped

```javascript
function example() {
  if (true) {
    var x = 10; // scoped to function, NOT block
  }
  console.log(x); // ✅ 10 — leaks out of the if-block
}

// var in loops — classic bug
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3 — all share the same 'i'
```

```javascript
// var can be re-declared
var a = 1;
var a = 2; // ✅ No error
```

### `let` — Block-Scoped

```javascript
function example() {
  if (true) {
    let y = 20; // scoped to if-block
  }
  console.log(y); // ❌ ReferenceError — y not accessible here
}

// let in loops — fixes the classic bug
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2 — each iteration has its own 'i'
```

### `const` — Block-Scoped + Immutable Binding

```javascript
const PI = 3.14159;
PI = 3; // ❌ TypeError: Assignment to constant variable

// ⚠️ const does NOT freeze objects — only the binding is constant
const obj = { name: "Alice" };
obj.name = "Bob"; // ✅ Works — mutating the object, not the binding
obj = {};         // ❌ TypeError — reassigning the binding

const arr = [1, 2, 3];
arr.push(4);      // ✅ Works
arr = [1, 2];     // ❌ TypeError
```

### Mental Model for Choosing

- Use `const` by default
- Use `let` only when you need to reassign
- Avoid `var` in modern JavaScript (it leaks scope)

### Tricky Edge Cases

```javascript
// var hoisting across blocks
console.log(x); // undefined (not ReferenceError)
{
  var x = 5;
}

// Temporal behavior difference in switch
switch (val) {
  case 1:
    let x = 10; // ⚠️ Shared block scope — all cases share it
    break;
  case 2:
    let x = 20; // ❌ SyntaxError: already declared
}

// Fix: use explicit blocks
switch (val) {
  case 1: {
    let x = 10;
    break;
  }
  case 2: {
    let x = 20; // ✅ Own block scope
    break;
  }
}
```

### Real Interview Questions

**Q1:** What is the output?
```javascript
var x = 1;
function foo() {
  console.log(x);
  var x = 2;
}
foo();
```
**Answer:** `undefined` — `var x` inside `foo` is hoisted to top of `foo` as `undefined`, shadowing the outer `x`.

**Q2:** Is `const` truly immutable?
**Answer:** No — `const` prevents *re-binding* but the value itself can be mutated if it's an object/array.

**Q3:** What's the difference between TDZ for `let` vs the hoisting of `var`?
**Answer:** Both are "hoisted" in the sense that the engine knows about them before execution. But `var` is initialized to `undefined` immediately, while `let`/`const` remain in TDZ (uninitialized) until the declaration line.

---

## 3. Scope

### What It Is

**Scope** defines the region of code where a variable is accessible. JavaScript uses **lexical (static) scoping** — scope is determined by where code is *written*, not where it is *called*.

### Types of Scope

#### 1. Global Scope
Variables declared outside any function or block. Accessible everywhere.

```javascript
var globalVar = "I'm global";
let globalLet = "Also global but not on window";

function foo() {
  console.log(globalVar); // ✅ accessible
}
```

> `var` at global level becomes a property of `window` (browsers). `let`/`const` do not.

#### 2. Function Scope
Variables declared inside a function. Only accessible within that function.

```javascript
function greet() {
  var message = "Hello";
  console.log(message); // ✅
}
console.log(message); // ❌ ReferenceError
```

#### 3. Block Scope (ES6+)
Variables declared with `let`/`const` inside `{}`. Only accessible within that block.

```javascript
{
  let blockVar = 42;
  console.log(blockVar); // ✅
}
console.log(blockVar); // ❌ ReferenceError
```

#### 4. Module Scope
In ES modules (`.mjs` or `type="module"`), top-level code has module scope, not global scope.

```javascript
// module.js
const secret = "private"; // module-scoped, not on window
export const public = "visible";
```

#### 5. Lexical Scope
Inner functions can access variables of outer functions.

```javascript
function outer() {
  let x = 10;
  function inner() {
    console.log(x); // ✅ — lexically sees outer's x
  }
  inner();
}
```

### Where Scope Applies / Doesn't Apply

| Construct | Creates Scope? |
|---|---|
| `function` | ✅ Yes (function scope) |
| `if`, `for`, `while` blocks `{}` | ✅ Yes for `let`/`const`, ❌ No for `var` |
| `try/catch` blocks | ✅ Yes for `let`/`const` |
| Arrow functions | ✅ Yes (own scope but no `this`) |
| `class` body | ✅ Yes |
| Plain `{}` | ✅ Yes for `let`/`const` |
| Object literals `{}` | ❌ No scope created |

### Mental Model

Think of scope as **nested boxes**. Inner boxes can see everything in outer boxes, but outer boxes cannot see inside inner boxes.

```
┌────────────────── Global Scope ──────────────────┐
│  var a = 1                                        │
│                                                   │
│  ┌──────────── outer() scope ──────────────────┐  │
│  │  let b = 2                                   │  │
│  │                                              │  │
│  │  ┌──────── inner() scope ─────────────────┐ │  │
│  │  │  const c = 3                            │ │  │
│  │  │  // can see: a, b, c ✅                 │ │  │
│  │  └────────────────────────────────────────┘ │  │
│  │  // can see: a, b ✅  cannot see: c ❌       │  │
│  └─────────────────────────────────────────────┘  │
│  // can see: a ✅  cannot see: b, c ❌             │
└───────────────────────────────────────────────────┘
```

### Real Interview Questions

**Q1:** Is JavaScript lexically or dynamically scoped?
**Answer:** **Lexically scoped** — scope is determined at write time (by nesting in source code), not at runtime.

**Q2:** What is the difference between scope and context (`this`)?
**Answer:** Scope refers to variable accessibility. Context (`this`) refers to the object the function is invoked on. They are completely different concepts.

---

## 4. Hoisting

### What It Is

**Hoisting** is JavaScript's behavior of moving declarations to the top of their containing scope during the compile phase, before any code executes. Only **declarations** are hoisted — not initializations.

### What Gets Hoisted

| Declaration Type | Hoisted? | Initialized To |
|---|---|---|
| `var` declaration | ✅ Yes | `undefined` |
| `var` initialization | ❌ No | — |
| `function` declaration | ✅ Yes | Full function body |
| `function` expression (`var f = function()`) | ✅ var only | `undefined` |
| Arrow function (`const f = () => {}`) | ✅ const only | TDZ |
| `let` / `const` | ✅ Yes (declared) | TDZ (not initialized) |
| `class` declaration | ✅ Yes (declared) | TDZ |

### How JavaScript Compiles Code (2 Phases)

1. **Compilation Phase:** Engine scans code, registers all declarations in their scope.
2. **Execution Phase:** Code runs line by line.

```javascript
// What you write:
console.log(x); // undefined
var x = 5;
console.log(x); // 5

// What JS engine sees (conceptually):
var x;           // hoisted declaration
console.log(x);  // undefined
x = 5;           // initialization stays in place
console.log(x);  // 5
```

### Function Declaration Hoisting (Full Hoist)

```javascript
greet(); // ✅ "Hello" — works before declaration

function greet() {
  console.log("Hello");
}
```

### Function Expression Hoisting (Partial)

```javascript
greet(); // ❌ TypeError: greet is not a function

var greet = function() {
  console.log("Hello");
};
// Only 'var greet' is hoisted → undefined
// Calling undefined() throws TypeError
```

### Hoisting in Practice

```javascript
// Hoisting priority: function declarations > var declarations
console.log(typeof foo); // "function" (not "undefined")

var foo = "variable";
function foo() {}

// After hoisting (conceptual):
// function foo() {}  ← takes priority
// var foo;           ← ignored (already declared)
// console.log(typeof foo); → "function"
// foo = "variable";
```

### Tricky Edge Cases

```javascript
// 1. var in if-block — still hoisted to function
function test() {
  if (false) {
    var x = 10; // Never runs, but var x is hoisted!
  }
  console.log(x); // undefined (not ReferenceError)
}

// 2. Named function expression — name only scoped inside
var f = function myFunc() {
  console.log(myFunc); // ✅ accessible inside
};
console.log(myFunc); // ❌ ReferenceError — not accessible outside

// 3. Block-scoped function declarations (behavior varies per engine)
if (true) {
  function blockFn() { return 1; }
}
// In strict mode: blockFn is block-scoped
// In sloppy mode: blockFn may leak to enclosing scope (avoid this!)
```

### Real Interview Questions

**Q1:** What is the output?
```javascript
foo();
var foo = function() { console.log("A"); };
function foo() { console.log("B"); }
foo();
```
**Answer:** `"B"` then `"A"`.
- After hoisting: `function foo(){"B"}` takes priority over `var foo`
- First `foo()` → `"B"` (function declaration)
- `foo = function(){"A"}` reassigns
- Second `foo()` → `"A"` (function expression)

**Q2:** Are `let`/`const` hoisted?
**Answer:** Yes — they are hoisted (registered in the scope), but NOT initialized, so they sit in TDZ until the declaration line.

**Q3:** What does "hoisting" actually mean under the hood?
**Answer:** During the parsing/compilation phase, V8 does a "pre-scan" of the code, registers all `var` declarations and function declarations in the relevant scope's environment record. This happens before any execution.

---

## 5. Scope Chain

### What It Is

The **scope chain** is the mechanism JavaScript uses to look up variable values. When a variable is referenced, the engine looks in the current scope first, then in each successive outer scope, all the way up to the global scope. If not found anywhere, it throws a `ReferenceError`.

### How It Works

```javascript
const global = "global";

function outer() {
  const outerVar = "outer";

  function middle() {
    const middleVar = "middle";

    function inner() {
      const innerVar = "inner";
      // Scope chain lookup order for 'global':
      // 1. inner scope → not found
      // 2. middle scope → not found
      // 3. outer scope → not found
      // 4. global scope → FOUND ✅
      console.log(global);
    }
    inner();
  }
  middle();
}
outer();
```

### The [[Scope]] Internal Property

Every function in JS has an internal `[[Scope]]` property that references the **scope chain** at the point the function was defined (not called). This is what makes closures work.

```javascript
function makeAdder(x) {
  // makeAdder's [[Scope]] → [local: {x}, outer: global]
  return function(y) {
    // inner's [[Scope]] → [local: {y}, outer: {x}, outer: global]
    return x + y; // looks up 'x' via scope chain
  };
}

const add5 = makeAdder(5);
add5(3); // 8 — 'x' found in outer scope via chain
```

### Scope Chain vs Prototype Chain

| Feature | Scope Chain | Prototype Chain |
|---|---|---|
| Used for | Variable lookup | Property lookup on objects |
| Traversal | Lexical (static) | Dynamic (runtime `__proto__`) |
| End | Global scope | `null` |

### Real Interview Questions

**Q1:** What is the scope chain?
**Answer:** An ordered list of environments (scopes) that JavaScript searches through to resolve variable references, from innermost to outermost (global).

**Q2:** Is scope chain determined at definition time or call time?
**Answer:** **Definition time** (lexical scoping). The chain is captured when the function is *created*, not when it is *invoked*.

**Q3:** What happens when a variable is not found in the scope chain?
**Answer:** In strict mode → `ReferenceError`. In sloppy mode for assignments → the variable is accidentally created as a global (a major bug source).

---

## 6. Block Scope

### What It Is

**Block scope** confines a variable to the `{}` block it's declared in. Only `let` and `const` are block-scoped. `var` is NOT block-scoped (it's function-scoped).

### Where It Applies

```javascript
// if block
if (true) {
  let x = 1;
  const y = 2;
  var z = 3;
}
console.log(x); // ❌ ReferenceError
console.log(y); // ❌ ReferenceError
console.log(z); // ✅ 3 — var leaked out

// for loop block
for (let i = 0; i < 3; i++) {
  // each iteration gets its OWN i
}
console.log(i); // ❌ ReferenceError

// while block
while (condition) {
  let temp = calculate();
  // temp only lives here
}

// try/catch
try {
  let err = new Error();
} catch (e) {
  // e is block-scoped to catch
}

// Standalone blocks (valid JS!)
{
  let secret = 42;
}
console.log(secret); // ❌ ReferenceError
```

### Block Scope in for Loops (Key Interview Topic)

```javascript
// var — ONE shared variable for all iterations
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 3, 3, 3

// let — NEW binding per iteration (V8 creates a new scope each iteration)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
// Output: 0, 1, 2

// IIFE workaround for var (pre-ES6)
for (var i = 0; i < 3; i++) {
  (function(j) {
    setTimeout(() => console.log(j), 0);
  })(i);
}
// Output: 0, 1, 2
```

### Mental Model

Each `{}` with `let`/`const` creates a new **lexical environment** (a new entry in the scope chain).

---

## 7. Shadowing & Illegal Shadowing

### What Shadowing Is

**Shadowing** occurs when a variable in an inner scope has the same name as a variable in an outer scope. The inner variable "shadows" (hides) the outer one within its scope.

```javascript
let x = "global";

function foo() {
  let x = "local"; // shadows outer x
  console.log(x);  // "local"
}

foo();
console.log(x); // "global" — outer x unchanged
```

### Types of Shadowing

```javascript
// var shadowing var ✅
var a = 1;
function f() {
  var a = 2; // ✅ valid
  console.log(a); // 2
}

// let shadowing let ✅
let b = 1;
{
  let b = 2; // ✅ valid — different scope
  console.log(b); // 2
}
console.log(b); // 1

// let shadowing var ✅
var c = 1;
function g() {
  let c = 2; // ✅ valid
}

// const shadowing anything ✅ (in a new block)
const d = 1;
{
  const d = 2; // ✅ valid
}
```

### Illegal Shadowing — What It Is

**Illegal shadowing** occurs when a `var` inside a block tries to shadow a `let`/`const` from an outer scope. This is a `SyntaxError` because `var` leaks out of the block (function-scoped), and it would effectively be in the SAME scope as the `let`, creating a conflict.

```javascript
// ❌ ILLEGAL: var cannot shadow let in same scope
let x = 10;
{
  var x = 20; // SyntaxError: Cannot redeclare block-scoped variable 'x'
}
// Why? 'var x' hoists to function/global scope where 'let x' already exists

// ✅ LEGAL: var CAN shadow let if in a function (creates new scope)
let x = 10;
function foo() {
  var x = 20; // ✅ valid — foo() creates a new scope boundary
  console.log(x); // 20
}
```

### Mental Model for Illegal Shadowing

The rule is: **`var` cannot shadow `let`/`const` if they end up in the same scope**. Since `var` is function-scoped (not block-scoped), a `var` in a plain `{}` block hoists to the enclosing function/global scope — where the `let` already lives. This creates a direct conflict.

### Tricky Edge Cases

```javascript
// ✅ This is fine — let in block, var in function creates real scope boundary
let x = 1;
function test() {
  if (true) {
    var x = 2; // ✅ var hoists to test() — no conflict with outer let
  }
  console.log(x); // 2
}

// ❌ Same function — var hoists and conflicts
function test2() {
  let x = 1;
  {
    var x = 2; // ❌ SyntaxError
  }
}
```

### Real Interview Questions

**Q1:** Is this valid?
```javascript
var a = 1;
let a = 2;
```
**Answer:** No — `SyntaxError`. Both are in the same (global) scope.

**Q2:** Why is it illegal for `var` to shadow `let`?
**Answer:** Because `var` is function/global scoped — it would hoist to the same scope as the `let`, creating a re-declaration of a block-scoped variable, which is forbidden.

---

## 8. Closures (Deep Explanation)

### What It Is

A **closure** is a function that **remembers** the variables from its outer lexical scope even after that outer scope has finished executing. The function "closes over" its surrounding environment.

**Formal definition:** A closure is the combination of a function and the lexical environment in which it was declared.

### Where It Applies

Closures happen **every time** a function is created in JavaScript. Every function is technically a closure. However, closures are most visible when an inner function outlives its outer function.

### Where It Does NOT Apply

- Pure standalone functions with no reference to outer variables (technically still closures, just trivial ones)
- Variables that are not referenced by the inner function are NOT captured (V8 optimizes this)

### Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│  Outer Function — executes and "dies"                        │
│                                                              │
│  Variables: x = 10, y = 20                                   │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐   │
│  │  Inner Function                                        │   │
│  │  Has a "backpack" (closure) attached to it:           │   │
│  │  { x: 10 }  ← only captures what it references       │   │
│  │                                                        │   │
│  │  function() { return x + 1; }                         │   │
│  └───────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                          │
              Outer function returns inner function
              Outer function's stack frame is gone
              BUT inner function's "backpack" persists
                          │
                          ▼
           Inner function still accesses x = 10 ✅
```

### Basic Closure Example

```javascript
function makeCounter() {
  let count = 0; // This variable is "closed over"

  return {
    increment() { count++; },
    decrement() { count--; },
    getCount() { return count; }
  };
}

const counter = makeCounter();
counter.increment();
counter.increment();
counter.increment();
console.log(counter.getCount()); // 3

// count is NOT accessible from outside
console.log(count); // ❌ ReferenceError
```

### Closures Capture Variables, NOT Values

This is the most critical thing to understand.

```javascript
// ❌ Common mistake — var shares one binding
function buggy() {
  const funcs = [];
  for (var i = 0; i < 3; i++) {
    funcs.push(() => console.log(i));
  }
  funcs.forEach(f => f()); // 3, 3, 3 — all see final i
}

// ✅ Fix 1: Use let (creates new binding per iteration)
function fixed1() {
  const funcs = [];
  for (let i = 0; i < 3; i++) {
    funcs.push(() => console.log(i));
  }
  funcs.forEach(f => f()); // 0, 1, 2
}

// ✅ Fix 2: Use IIFE to capture value
function fixed2() {
  const funcs = [];
  for (var i = 0; i < 3; i++) {
    funcs.push((function(j) {
      return () => console.log(j);
    })(i));
  }
  funcs.forEach(f => f()); // 0, 1, 2
}
```

### Closures and Private State

```javascript
function createBankAccount(initialBalance) {
  let balance = initialBalance; // Private! No external access

  return {
    deposit(amount) {
      if (amount > 0) balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (amount > balance) throw new Error("Insufficient funds");
      balance -= amount;
      return balance;
    },
    getBalance() { return balance; }
  };
}

const account = createBankAccount(100);
account.deposit(50);  // 150
account.withdraw(30); // 120
console.log(account.balance); // undefined — truly private!
```

### Closures Over Mutable Variables

```javascript
function makeMultiplier(multiplier) {
  return function(x) {
    return x * multiplier;
  };
}

const double = makeMultiplier(2);
const triple = makeMultiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
// Each closure has its own 'multiplier' binding
```

### Closures in Async Code

```javascript
function fetchUser(id) {
  // 'id' is captured in the closure
  return fetch(`/api/users/${id}`)
    .then(response => response.json())
    .then(user => {
      console.log(`Fetched user ${id}:`, user.name); // 'id' is captured ✅
    });
}
```

### Memory Considerations

```javascript
function outer() {
  const heavyData = new Array(1000000).fill("data");
  const small = 42;

  return function inner() {
    return small; // Only captures 'small'
    // V8 is smart: 'heavyData' is NOT captured, so it can be GC'd
  };
}
```

> ⚠️ **Memory Leak Pattern:** If you capture a large object in a closure that lives forever (like a global), the large object can never be GC'd.

```javascript
// ⚠️ Potential memory leak
let leakyRef;
function bad() {
  const HUGE = new Array(1e6).fill(0);
  leakyRef = function() {
    return HUGE[0]; // HUGE stays in memory as long as leakyRef exists
  };
}
```

### Tricky Closure Edge Cases

```javascript
// 1. Closure in a loop with setTimeout
for (var i = 1; i <= 3; i++) {
  setTimeout(function() {
    console.log(i); // 4, 4, 4
  }, i * 1000);
}

// 2. Closures share the SAME variable binding
function sharedBinding() {
  let x = 0;
  const inc = () => ++x;
  const dec = () => --x;
  const get = () => x;
  return { inc, dec, get };
}
// All three functions share the exact same 'x' binding

// 3. Closure in returned class method
function createClass() {
  let privateState = 0;
  class MyClass {
    doSomething() {
      return ++privateState; // closure over privateState
    }
  }
  return MyClass;
}
```

### Real Interview Questions

**Q1:** What is a closure? Give an example.
**Answer:** A closure is a function that retains access to variables from its outer lexical scope even after that scope has finished executing.

**Q2:** What is the output?
```javascript
function outer() {
  let x = 10;
  function inner() {
    x++;
    console.log(x);
  }
  return inner;
}
const fn = outer();
fn(); // ?
fn(); // ?
```
**Answer:** `11` then `12` — both calls share the same `x` binding.

**Q3:** How do closures cause memory leaks?
**Answer:** When a closure captures a reference to a large object, and that closure lives longer than expected (stored in a global, event listener not removed, etc.), the large object cannot be garbage collected.

**Q4:** What's the difference between a closure and a function?
**Answer:** Every function is a closure, but closures become significant when the inner function outlives the outer scope, retaining access to the outer environment's variables.

---

## 9. OOP in JavaScript

### What It Is

JavaScript supports **prototype-based OOP**, not class-based like Java. ES6 `class` syntax is syntactic sugar over the prototype system.

### Prototype Chain

Every JS object has an internal `[[Prototype]]` property (accessible via `__proto__` or `Object.getPrototypeOf()`). When a property is not found on an object, JS looks up the prototype chain.

```javascript
const animal = {
  breathe() { return "breathing"; }
};

const dog = Object.create(animal);
dog.bark = function() { return "woof"; };

console.log(dog.bark());    // "woof" — own property
console.log(dog.breathe()); // "breathing" — from prototype chain
console.log(dog.hasOwnProperty("bark"));    // true
console.log(dog.hasOwnProperty("breathe")); // false
```

### Constructor Functions (Pre-ES6)

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

Person.prototype.greet = function() {
  return `Hi, I'm ${this.name}`;
};

Person.prototype.birthday = function() {
  this.age++;
};

const alice = new Person("Alice", 30);
console.log(alice.greet()); // "Hi, I'm Alice"
```

**What `new` does (step by step):**
1. Creates a new empty object `{}`
2. Sets `__proto__` to `Constructor.prototype`
3. Calls the constructor with `this` = new object
4. Returns the new object (unless constructor explicitly returns an object)

```javascript
// Manual implementation of 'new'
function myNew(Constructor, ...args) {
  const obj = Object.create(Constructor.prototype);
  const result = Constructor.apply(obj, args);
  return result instanceof Object ? result : obj;
}
```

### ES6 Classes (Syntactic Sugar)

```javascript
class Animal {
  #sound; // private field (ES2022)

  constructor(name, sound) {
    this.name = name;
    this.#sound = sound;
  }

  speak() {
    return `${this.name} says ${this.#sound}`;
  }

  static create(name, sound) { // static method
    return new Animal(name, sound);
  }

  get info() { // getter
    return `${this.name} (${this.#sound})`;
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name, "woof"); // Must call super() before using 'this'
    this.tricks = [];
  }

  learn(trick) {
    this.tricks.push(trick);
    return this;
  }

  speak() {
    return super.speak() + "!"; // Call parent method
  }
}

const rex = new Dog("Rex");
rex.learn("sit").learn("shake"); // method chaining
console.log(rex.speak()); // "Rex says woof!"
console.log(rex.info);    // "Rex (woof)"
```

### Inheritance Patterns

```javascript
// 1. Prototypal inheritance with Object.create
const vehicleProto = {
  start() { return `${this.brand} started`; }
};

const car = Object.create(vehicleProto);
car.brand = "Toyota";
car.start(); // "Toyota started"

// 2. Mixin pattern (multiple inheritance-like)
const Serializable = {
  serialize() { return JSON.stringify(this); },
  deserialize(json) { return JSON.parse(json); }
};

const Printable = {
  print() { console.log(this.toString()); }
};

class MyClass {
  constructor(data) { this.data = data; }
}

Object.assign(MyClass.prototype, Serializable, Printable);
```

### Real Interview Questions

**Q1:** What does the `new` keyword do?
**Answer:** Creates a new object, sets its `[[Prototype]]` to the constructor's `prototype` property, invokes the constructor with `this` bound to the new object, and returns the object.

**Q2:** What is the difference between `__proto__` and `prototype`?
**Answer:** `prototype` is a property on **functions** that becomes the `[[Prototype]]` of instances created by that function. `__proto__` is an accessor on **objects** that exposes the internal `[[Prototype]]`.

**Q3:** Are ES6 classes truly "classes" like in Java?
**Answer:** No — they are syntactic sugar over JavaScript's prototype-based system. The class syntax makes it look class-based but under the hood it's still prototype chains.

---

## 10. Factory Functions

### What They Are

A **factory function** is any function that returns an object — without using `new` or classes. It's an alternative pattern to constructor functions and classes.

### Where They Apply

- When you want private state (closures)
- When you want to avoid `new` keyword issues
- When you want to return different object types conditionally
- Composition over inheritance scenarios

### Basic Factory Function

```javascript
function createUser(name, email) {
  // private state via closure
  let loginCount = 0;

  return {
    name,
    email,
    login() {
      loginCount++;
      return `${name} logged in (${loginCount} times)`;
    },
    getLoginCount() {
      return loginCount;
    }
  };
}

const user1 = createUser("Alice", "alice@example.com");
const user2 = createUser("Bob", "bob@example.com");

user1.login(); // "Alice logged in (1 times)"
user1.login(); // "Alice logged in (2 times)"
console.log(user1.getLoginCount()); // 2
console.log(user2.getLoginCount()); // 0 — independent state
```

### Factory vs Constructor vs Class

```javascript
// Constructor (requires 'new', 'this' confusion possible)
function PersonConstructor(name) {
  this.name = name;
  this.greet = function() { return `Hi ${this.name}`; };
}
const p1 = new PersonConstructor("Alice"); // must use new

// Factory (no 'new', cleaner)
function createPerson(name) {
  return {
    name,
    greet() { return `Hi ${name}`; }
  };
}
const p2 = createPerson("Alice"); // no 'new' needed

// Class (syntactic sugar)
class PersonClass {
  constructor(name) { this.name = name; }
  greet() { return `Hi ${this.name}`; }
}
const p3 = new PersonClass("Alice");
```

### Composable Factories (Composition over Inheritance)

```javascript
// Behaviors as mixins
const canSwim = (state) => ({
  swim: () => `${state.name} is swimming`
});

const canFly = (state) => ({
  fly: () => `${state.name} is flying`
});

const canQuack = (state) => ({
  quack: () => `${state.name} says quack`
});

// Compose a Duck (has all behaviors)
function createDuck(name) {
  const state = { name };
  return Object.assign(
    {},
    canSwim(state),
    canFly(state),
    canQuack(state)
  );
}

const duck = createDuck("Donald");
duck.swim();  // "Donald is swimming"
duck.fly();   // "Donald is flying"
duck.quack(); // "Donald says quack"
```

### When to Use Factory Functions

- Prefer when you need **private variables** (closures provide true privacy)
- Prefer when you want **composition over inheritance**
- Prefer when the `this` binding would be confusing
- Avoid when you need `instanceof` checks or need to extend with `class`

---

## 11. Memoization/Caching

### What It Is

**Memoization** is an optimization technique where the results of expensive function calls are **cached** based on their input arguments. If the same arguments are passed again, the cached result is returned instead of recomputing.

### Where It Applies

- Pure functions (same inputs always produce same outputs)
- Expensive computations (recursion, API calls, complex calculations)
- Functions called repeatedly with the same arguments

### Basic Memoization

```javascript
function memoize(fn) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);

    if (cache.has(key)) {
      console.log("Cache hit!");
      return cache.get(key);
    }

    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// Usage
const slowSquare = (n) => {
  // Simulate expensive operation
  let result = 0;
  for (let i = 0; i < 1e6; i++) result += n;
  return result;
};

const fastSquare = memoize(slowSquare);
fastSquare(5);   // Computed (slow)
fastSquare(5);   // Cache hit! (instant)
fastSquare(10);  // Computed (slow)
fastSquare(10);  // Cache hit! (instant)
```

### Memoizing Recursive Functions

```javascript
// Fibonacci without memoization: O(2^n)
function fib(n) {
  if (n <= 1) return n;
  return fib(n - 1) + fib(n - 2);
}

// With memoization: O(n)
function memoFib() {
  const cache = {};
  return function fib(n) {
    if (n in cache) return cache[n];
    if (n <= 1) return n;
    cache[n] = fib(n - 1) + fib(n - 2);
    return cache[n];
  };
}

const fibonacci = memoFib();
fibonacci(50); // Instant! Without memoization: 2^50 calls
```

### Advanced Memoization with WeakMap (Avoids Memory Leaks)

```javascript
// For object arguments — use WeakMap to allow GC
function memoizeWeak(fn) {
  const cache = new WeakMap();

  return function(obj) {
    if (cache.has(obj)) return cache.get(obj);
    const result = fn(obj);
    cache.set(obj, result);
    return result;
  };
}
```

### Cache with TTL (Time-to-Live)

```javascript
function memoizeWithTTL(fn, ttlMs) {
  const cache = new Map();

  return function(...args) {
    const key = JSON.stringify(args);
    const now = Date.now();

    if (cache.has(key)) {
      const { value, expiry } = cache.get(key);
      if (now < expiry) return value;
    }

    const result = fn.apply(this, args);
    cache.set(key, { value: result, expiry: now + ttlMs });
    return result;
  };
}

const getUser = memoizeWithTTL(fetchUser, 5 * 60 * 1000); // 5 min TTL
```

### Real Interview Questions

**Q1:** What is memoization and when would you use it?
**Answer:** Caching the return value of a function based on its inputs. Use it for pure functions that are called repeatedly with the same arguments, where recalculation is expensive (e.g., Fibonacci, API calls, complex DOM calculations).

**Q2:** What are the downsides of memoization?
**Answer:** Increased memory usage (storing results indefinitely), stale cache (for non-pure functions), and complexity from serializing cache keys. Use TTL or LRU cache for real-world applications.

---

## 12. Event Handlers

### What They Are

**Event handlers** are functions that respond to events (user interactions, browser events, etc.) in the DOM or Node.js. Closures make event handlers powerful.

### Types

```javascript
// 1. HTML attribute (avoid this)
<button onclick="handleClick()">Click</button>

// 2. DOM property (one handler per event)
button.onclick = function() { console.log("clicked"); };

// 3. addEventListener (multiple handlers, preferred)
button.addEventListener("click", function() { console.log("clicked"); });

// 4. Arrow function handler
button.addEventListener("click", (e) => console.log(e.target));
```

### Event Handler + Closure Pattern

```javascript
function createClickCounter(buttonId) {
  let count = 0; // private, captured by closure
  const button = document.getElementById(buttonId);

  button.addEventListener("click", function() {
    count++;
    button.textContent = `Clicked ${count} times`;
  });

  return {
    getCount: () => count,
    reset: () => { count = 0; button.textContent = "Click me"; }
  };
}

const counter = createClickCounter("myBtn");
```

### Removing Event Listeners (Important for Memory)

```javascript
// ❌ Arrow functions are anonymous — can't remove!
element.addEventListener("click", () => doSomething());
// element.removeEventListener("click", ???) — impossible

// ✅ Named function reference
function handler(e) { doSomething(e); }
element.addEventListener("click", handler);
element.removeEventListener("click", handler); // ✅ can remove

// ✅ AbortController (modern approach)
const controller = new AbortController();
element.addEventListener("click", handler, { signal: controller.signal });
controller.abort(); // Removes ALL listeners registered with this controller
```

### Event Delegation (Closure-Powered)

```javascript
// Instead of adding listeners to each item, add ONE to the parent
function createList(items) {
  const ul = document.querySelector("ul");

  ul.addEventListener("click", function(e) {
    const li = e.target.closest("li");
    if (!li) return;
    const index = [...ul.children].indexOf(li);
    console.log(`Clicked item: ${items[index]}`); // closure over 'items'
  });
}

createList(["Apple", "Banana", "Cherry"]);
```

---

## 13. Async Callbacks

### What They Are

**Callbacks** are functions passed as arguments to other functions and invoked later (asynchronously). They were the original way to handle async operations in JS before Promises.

### The Event Loop and Async

```javascript
console.log("1 - Start");

setTimeout(() => {
  console.log("3 - Timeout callback");
}, 0);

Promise.resolve().then(() => {
  console.log("2.5 - Microtask (Promise)");
});

console.log("2 - End");

// Output:
// 1 - Start
// 2 - End
// 2.5 - Microtask (Promise)  ← microtask queue first!
// 3 - Timeout callback       ← macrotask queue second
```

### Callback Hell (Pyramid of Doom)

```javascript
// ❌ Hard to read and maintain
getUser(userId, function(user) {
  getPosts(user.id, function(posts) {
    getComments(posts[0].id, function(comments) {
      getAuthor(comments[0].userId, function(author) {
        console.log(author.name); // deeply nested!
      });
    });
  });
});
```

### Solving Callback Hell with Promises

```javascript
// ✅ Flat chain
getUser(userId)
  .then(user => getPosts(user.id))
  .then(posts => getComments(posts[0].id))
  .then(comments => getAuthor(comments[0].userId))
  .then(author => console.log(author.name))
  .catch(err => console.error(err));
```

### Error Handling in Async Callbacks

```javascript
// Callback convention (Node.js style): error-first
function readFile(path, callback) {
  fs.readFile(path, "utf8", (err, data) => {
    if (err) return callback(err, null);
    callback(null, data);
  });
}

readFile("./data.txt", (err, data) => {
  if (err) {
    console.error("Error:", err);
    return;
  }
  console.log(data);
});
```

---

## 14. Module Patterns

### What They Are

**Module patterns** are ways to encapsulate code and expose only what's needed, creating a public API while hiding implementation details.

### 1. IIFE Module Pattern (Classic)

```javascript
const Calculator = (function() {
  // Private
  let history = [];

  function logOperation(op) {
    history.push(op);
  }

  // Public API
  return {
    add(a, b) {
      const result = a + b;
      logOperation(`${a} + ${b} = ${result}`);
      return result;
    },
    getHistory() {
      return [...history]; // return copy, not reference
    }
  };
})();

Calculator.add(2, 3);  // 5
Calculator.getHistory(); // ["2 + 3 = 5"]
```

### 2. Revealing Module Pattern

```javascript
const ThemeManager = (function() {
  let currentTheme = "light";
  const validThemes = ["light", "dark", "auto"];

  function setTheme(theme) {
    if (!validThemes.includes(theme)) throw new Error("Invalid theme");
    currentTheme = theme;
    document.body.className = theme;
  }

  function getTheme() {
    return currentTheme;
  }

  // Reveal only selected functions
  return { setTheme, getTheme };
})();
```

### 3. ES6 Modules (Modern Standard)

```javascript
// math.js
const PI = Math.PI;

function add(a, b) { return a + b; }
function multiply(a, b) { return a * b; }

export { add, multiply };
export default PI;
export const E = Math.E;

// main.js
import PI, { add, multiply } from "./math.js";
import * as Math from "./math.js";
```

### 4. CommonJS (Node.js)

```javascript
// utils.js
const _ = require("lodash");
let count = 0;

function increment() { return ++count; }

module.exports = { increment };
// OR: module.exports.increment = increment;

// main.js
const { increment } = require("./utils");
```

---

## 15. `setTimeout` + Closures

### The Classic Problem

```javascript
// ❌ Classic bug
for (var i = 1; i <= 5; i++) {
  setTimeout(function() {
    console.log(i); // logs 6 five times!
  }, i * 1000);
}
// Why? All closures share the SAME 'i' binding
// By the time any timeout fires, loop is done and i = 6
```

### Solutions

```javascript
// ✅ Solution 1: let (creates new binding per iteration)
for (let i = 1; i <= 5; i++) {
  setTimeout(() => console.log(i), i * 1000);
} // 1, 2, 3, 4, 5

// ✅ Solution 2: IIFE to capture value
for (var i = 1; i <= 5; i++) {
  (function(j) {
    setTimeout(() => console.log(j), j * 1000);
  })(i);
}

// ✅ Solution 3: bind
for (var i = 1; i <= 5; i++) {
  setTimeout(console.log.bind(null, i), i * 1000);
}

// ✅ Solution 4: array forEach (creates new scope per iteration)
[1, 2, 3, 4, 5].forEach(i => {
  setTimeout(() => console.log(i), i * 1000);
});
```

### Tricky setTimeout Edge Cases

```javascript
// setTimeout(fn, 0) is NOT immediate — it goes to macrotask queue
console.log("start");
setTimeout(() => console.log("timeout"), 0);
console.log("end");
// Output: "start" → "end" → "timeout"

// Nested timeouts — minimum delay accumulates
setTimeout(() => {
  setTimeout(() => {
    setTimeout(() => {
      // After 5+ nesting levels, browsers enforce minimum 4ms delay
    }, 0);
  }, 0);
}, 0);

// setTimeout returns an ID — can clear it
const id = setTimeout(() => console.log("won't fire"), 5000);
clearTimeout(id); // Cancels the timeout
```

---

## 16. IIFE & IIFE + Closures

### What IIFE Is

**IIFE (Immediately Invoked Function Expression)** is a function that is defined and executed at the same time.

```javascript
(function() {
  // code here
})();

// Arrow function IIFE
(() => {
  // code here
})();

// Named IIFE (name is local to the function)
(function init() {
  console.log("initialized");
})();
```

### Why IIFE Was Needed

Before ES6 modules, JavaScript had no block scope. IIFE was the only way to create isolated scope.

```javascript
// Without IIFE — pollutes global scope
var data = [];
var process = function() {};

// With IIFE — nothing leaks out
(function() {
  var data = [];       // private
  var process = function() {};
  // only expose what's needed
  window.myLib = { /* public API */ };
})();
```

### IIFE + Closures

IIFE and closures together are extremely powerful for the **module pattern**:

```javascript
const counter = (function() {
  let count = 0; // private, via closure
  
  function increment() { return ++count; }
  function decrement() { return --count; }
  function reset() { count = 0; return count; }
  function getCount() { return count; }
  
  return { increment, decrement, reset, getCount };
})();

counter.increment(); // 1
counter.increment(); // 2
counter.decrement(); // 1
console.log(counter.count); // undefined — truly private
```

### IIFE with Parameters

```javascript
(function(global, factory) {
  factory(global); // Used in UMD module pattern
})(typeof window !== "undefined" ? window : this, function(root) {
  root.myLib = {};
});
```

### Modern Equivalent

IIFE is less needed now with ES6 modules and `let`/`const`, but still useful for:
- One-time initialization code
- Avoiding variable name collisions in older environments
- Creating scope in scripts without modules

---

## 17. First-Class Functions

### What It Means

Functions in JavaScript are **first-class citizens** — they can be:
1. Assigned to variables
2. Passed as arguments to other functions
3. Returned from functions
4. Stored in data structures

```javascript
// 1. Assign to variable
const greet = function(name) { return `Hello, ${name}`; };
const sayHi = greet; // copy the reference
sayHi("Alice"); // "Hello, Alice"

// 2. Pass as argument
function applyTwice(fn, value) {
  return fn(fn(value));
}
const double = x => x * 2;
applyTwice(double, 3); // 12

// 3. Return from function
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}
const triple = multiplier(3);
triple(5); // 15

// 4. Store in data structure
const operations = {
  add: (a, b) => a + b,
  subtract: (a, b) => a - b,
  multiply: (a, b) => a * b
};
const actions = [
  (x) => x + 1,
  (x) => x * 2,
  (x) => x ** 2
];
```

### Why This Matters

First-class functions are the foundation of:
- **Callbacks** (passing functions)
- **Higher-order functions** (`map`, `filter`, `reduce`)
- **Closures** (returning functions)
- **Currying** (returning partially applied functions)
- **Functional programming** patterns

---

## 18. Timers in JS

### Types of Timers

| Timer | Runs | Cleared By |
|---|---|---|
| `setTimeout(fn, ms)` | Once, after delay | `clearTimeout(id)` |
| `setInterval(fn, ms)` | Repeatedly | `clearInterval(id)` |
| `requestAnimationFrame(fn)` | Before next repaint | `cancelAnimationFrame(id)` |
| `queueMicrotask(fn)` | End of current task (microtask) | — |

### setTimeout

```javascript
const id = setTimeout(function() {
  console.log("Fired after ~1 second");
}, 1000);

clearTimeout(id); // Cancel before it fires

// setTimeout(fn, 0) — deferred, not immediate
setTimeout(() => console.log("deferred"), 0);
console.log("sync");
// Output: "sync" then "deferred"
```

### setInterval

```javascript
let ticks = 0;
const id = setInterval(function() {
  ticks++;
  console.log(`Tick: ${ticks}`);
  if (ticks >= 5) clearInterval(id); // Stop after 5 ticks
}, 1000);

// ⚠️ setInterval does NOT account for execution time
// If callback takes 200ms and interval is 1000ms,
// actual gaps are NOT exactly 1000ms

// ✅ Better: use recursive setTimeout for precise intervals
function preciseInterval(fn, ms) {
  function tick() {
    fn();
    setTimeout(tick, ms);
  }
  setTimeout(tick, ms);
}
```

### requestAnimationFrame

```javascript
let position = 0;
let animationId;

function animate() {
  position += 2;
  element.style.left = position + "px";

  if (position < 500) {
    animationId = requestAnimationFrame(animate);
  }
}

requestAnimationFrame(animate);
// cancelAnimationFrame(animationId); // to stop

// rAF: ~60fps (16.67ms), synced with display refresh rate
// Better than setTimeout for animations (no frame skipping)
```

### Timer Accuracy and Event Loop

```javascript
// Timers are NOT real-time — they're "at least this long"
console.time("test");
setTimeout(() => {
  console.timeEnd("test"); // Could be > 1000ms if event loop is busy
}, 1000);

// Simulating busy event loop delays timer
const start = Date.now();
while (Date.now() - start < 2000) {} // Blocks for 2 seconds
// Timer would fire 2+ seconds late because event loop was blocked
```

### Real Interview Questions

**Q1:** What is the minimum delay for `setTimeout`?
**Answer:** Technically 0ms, but browsers enforce a minimum of ~4ms after 5+ nested calls. The actual delay is "at least" the specified time.

**Q2:** How do you implement setInterval that accounts for execution time?
**Answer:** Use recursive `setTimeout` — after each execution completes, schedule the next one.

---

## 19. Higher-Order Functions (HOF)

### What They Are

A **Higher-Order Function** is a function that either:
1. Takes one or more functions as arguments, OR
2. Returns a function as its result

### Built-in HOFs

```javascript
// Array methods: forEach, map, filter, reduce, find, every, some, sort
[1, 2, 3].map(x => x * 2);       // [2, 4, 6]
[1, 2, 3, 4].filter(x => x % 2 === 0); // [2, 4]

// Function composition
const compose = (...fns) => x => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => x => fns.reduce((acc, fn) => fn(acc), x);

const add1 = x => x + 1;
const double = x => x * 2;
const square = x => x ** 2;

const transform = compose(square, double, add1); // square(double(add1(x)))
transform(3); // square(double(4)) = square(8) = 64
```

### Custom HOFs

```javascript
// Currying — converts multi-arg function to chain of single-arg functions
const curry = (fn) => {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    }
    return function(...moreArgs) {
      return curried.apply(this, args.concat(moreArgs));
    };
  };
};

const add = curry((a, b, c) => a + b + c);
add(1)(2)(3);    // 6
add(1, 2)(3);    // 6
add(1)(2, 3);    // 6
add(1, 2, 3);    // 6

// Partial application
const multiply = (a, b) => a * b;
const double = multiply.bind(null, 2); // partially apply first arg
double(5); // 10

// Function that times execution
function withTiming(fn) {
  return function(...args) {
    const start = performance.now();
    const result = fn.apply(this, args);
    console.log(`${fn.name} took ${performance.now() - start}ms`);
    return result;
  };
}

const timedSort = withTiming(Array.prototype.sort);
```

---

## 20. `map`, `filter`, `reduce` (Deep)

### `map` — Transform

**What:** Creates a NEW array by transforming each element via a callback.  
**Returns:** New array of same length.  
**Does NOT:** Mutate the original array.

```javascript
// Signature: array.map(callback(currentValue, index, array), thisArg)

const nums = [1, 2, 3, 4, 5];

// Basic
nums.map(x => x * 2); // [2, 4, 6, 8, 10]

// With index
nums.map((x, i) => `${i}: ${x}`); // ["0: 1", "1: 2", ...]

// Objects
const users = [{name: "Alice", age: 30}, {name: "Bob", age: 25}];
users.map(u => u.name); // ["Alice", "Bob"]
users.map(u => ({...u, isAdult: u.age >= 18})); // adds property

// Chaining
[1, 2, 3]
  .map(x => x * 2)
  .map(x => x + 1); // [3, 5, 7]
```

**Gotchas:**

```javascript
// parseInt with map — classic gotcha!
["1", "2", "3"].map(parseInt); // [1, NaN, NaN]
// Why? map passes (value, index, array) to callback
// parseInt("1", 0) = 1, parseInt("2", 1) = NaN, parseInt("3", 2) = NaN
// (second arg to parseInt is the RADIX)

// Fix:
["1", "2", "3"].map(Number);          // [1, 2, 3]
["1", "2", "3"].map(x => parseInt(x)); // [1, 2, 3]

// map skips holes in sparse arrays
[1, , 3].map(x => x * 2); // [2, empty, 6]

// map on array-like objects
Array.from({length: 5}, (_, i) => i); // [0, 1, 2, 3, 4]
```

### `filter` — Select

**What:** Creates a NEW array with only elements that pass a test.  
**Returns:** New array of 0 to original length.

```javascript
// Signature: array.filter(callback(currentValue, index, array), thisArg)

const nums = [1, 2, 3, 4, 5, 6];

// Basic
nums.filter(x => x % 2 === 0); // [2, 4, 6]
nums.filter(x => x > 3);       // [4, 5, 6]

// Remove falsy values
[0, 1, "", "a", null, undefined, false, true].filter(Boolean);
// [1, "a", true]

// Filter objects
const products = [
  { name: "Apple", price: 1.5, inStock: true },
  { name: "Mango", price: 3.0, inStock: false },
  { name: "Banana", price: 0.5, inStock: true }
];
products.filter(p => p.inStock && p.price < 2);
// [{name: "Apple", ...}, {name: "Banana", ...}]

// Remove duplicates
const arr = [1, 2, 2, 3, 3, 4];
arr.filter((val, idx, self) => self.indexOf(val) === idx); // [1, 2, 3, 4]
// Better: [...new Set(arr)]
```

### `reduce` — Accumulate

**What:** Reduces an array to a SINGLE value by applying a function against an accumulator.  
**Returns:** Single value (any type).

```javascript
// Signature: array.reduce(callback(accumulator, currentValue, index, array), initialValue)

const nums = [1, 2, 3, 4, 5];

// Sum
nums.reduce((acc, val) => acc + val, 0); // 15

// Product
nums.reduce((acc, val) => acc * val, 1); // 120

// Max
nums.reduce((max, val) => val > max ? val : max, -Infinity); // 5

// Flatten
[[1, 2], [3, 4], [5]].reduce((acc, arr) => acc.concat(arr), []);
// [1, 2, 3, 4, 5]
// Better: [[1, 2], [3, 4], [5]].flat()

// Group by
const people = [
  { name: "Alice", dept: "Engineering" },
  { name: "Bob",   dept: "Marketing" },
  { name: "Carol", dept: "Engineering" }
];
people.reduce((groups, person) => {
  const dept = person.dept;
  if (!groups[dept]) groups[dept] = [];
  groups[dept].push(person);
  return groups;
}, {});
// { Engineering: [{Alice}, {Carol}], Marketing: [{Bob}] }

// Count occurrences
["a", "b", "a", "c", "b", "a"].reduce((count, char) => {
  count[char] = (count[char] || 0) + 1;
  return count;
}, {});
// { a: 3, b: 2, c: 1 }

// Build a pipeline
const pipeline = [
  x => x + 1,
  x => x * 2,
  x => x - 3
];
pipeline.reduce((val, fn) => fn(val), 5); // ((5+1)*2)-3 = 9
```

**Critical: No initial value = dangerous!**

```javascript
// Without initial value: first element becomes accumulator
[1, 2, 3].reduce((acc, val) => acc + val); // 6 (starts with acc=1, val=2)

// ❌ Dangerous: empty array with no initial value
[].reduce((acc, val) => acc + val); // TypeError!

// ✅ Always provide initial value
[].reduce((acc, val) => acc + val, 0); // 0 — safe
```

### Real Interview Questions

**Q1:** What does `["1","2","3"].map(parseInt)` return?
**Answer:** `[1, NaN, NaN]` because `map` passes `(value, index)` to `parseInt`, so it becomes `parseInt("2", 1)` and `parseInt("3", 2)`, which are `NaN`.

**Q2:** Implement `map` using `reduce`:
```javascript
Array.prototype.myMap = function(fn) {
  return this.reduce((acc, val, idx, arr) => {
    acc.push(fn(val, idx, arr));
    return acc;
  }, []);
};
```

**Q3:** Implement `filter` using `reduce`:
```javascript
Array.prototype.myFilter = function(fn) {
  return this.reduce((acc, val, idx, arr) => {
    if (fn(val, idx, arr)) acc.push(val);
    return acc;
  }, []);
};
```

---

## 21. `this` Keyword

### What It Is

`this` is a **dynamic context binding** — it refers to the object that a function is being invoked on. The value of `this` depends on **how** the function is called, NOT where it's defined (except for arrow functions).

### The 5 Rules of `this` (Binding Rules)

#### Rule 1: Default Binding (Standalone Call)

```javascript
function foo() {
  console.log(this);
}

foo(); // In non-strict: window/global. In strict mode: undefined
```

#### Rule 2: Implicit Binding (Method Call)

```javascript
const obj = {
  name: "Alice",
  greet() {
    console.log(this.name); // 'this' = obj
  }
};
obj.greet(); // "Alice"

// ⚠️ Losing implicit binding
const fn = obj.greet;
fn(); // undefined (or error in strict) — 'this' is no longer obj!

// ⚠️ Callback context loss
setTimeout(obj.greet, 1000); // 'this' is window, not obj
```

#### Rule 3: Explicit Binding (call/apply/bind)

```javascript
function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

const user = { name: "Bob" };

// call — args individually
greet.call(user, "Hello", "!");    // "Hello, Bob!"

// apply — args as array
greet.apply(user, ["Hello", "!"]); // "Hello, Bob!"

// bind — returns NEW function with 'this' permanently bound
const greetBob = greet.bind(user, "Hello");
greetBob("!");     // "Hello, Bob!"
greetBob("?");     // "Hello, Bob?"
```

#### Rule 4: `new` Binding

```javascript
function Person(name) {
  this.name = name; // 'this' is the newly created object
}
const alice = new Person("Alice");
console.log(alice.name); // "Alice"
```

#### Rule 5: Arrow Function (Lexical `this`)

```javascript
const obj = {
  name: "Alice",
  greet: function() {
    // 'this' here is obj ✅
    const inner = () => {
      console.log(this.name); // Arrow inherits 'this' from greet ✅
    };
    inner();
  },
  arrowMethod: () => {
    console.log(this.name); // ❌ Arrow at object level — 'this' is window/undefined!
  }
};
```

### Binding Priority (Highest to Lowest)

1. `new` binding
2. Explicit binding (`call`/`apply`/`bind`)
3. Implicit binding (method call)
4. Default binding (global or `undefined` in strict)

### `this` in Classes

```javascript
class Timer {
  constructor() {
    this.seconds = 0;
  }

  start() {
    // ❌ 'this' is lost in callback
    setInterval(function() {
      this.seconds++; // TypeError: Cannot set property 'seconds' of undefined
    }, 1000);

    // ✅ Fix 1: Arrow function (lexical this)
    setInterval(() => {
      this.seconds++; // ✅ 'this' is Timer instance
    }, 1000);

    // ✅ Fix 2: bind
    setInterval(function() {
      this.seconds++;
    }.bind(this), 1000);

    // ✅ Fix 3: self/that pattern
    const self = this;
    setInterval(function() {
      self.seconds++;
    }, 1000);
  }
}
```

### Real Interview Questions

**Q1:** What is the output?
```javascript
const obj = {
  x: 10,
  getX: function() {
    return this.x;
  }
};
const { getX } = obj;
console.log(getX()); // ?
```
**Answer:** `undefined` (in non-strict) or `TypeError` (in strict). Destructuring loses the `this` binding.

**Q2:** What does `bind` return?
**Answer:** A new function with `this` permanently bound to the first argument. The original function is unchanged.

**Q3:** Can you override a `bind`-ed `this` with `call`?
```javascript
function foo() { console.log(this.x); }
const bound = foo.bind({ x: 1 });
bound.call({ x: 2 }); // ?
```
**Answer:** `1` — `bind` takes priority over `call`. The bound `this` cannot be overridden.

**Q4:** What is `this` in an arrow function?
**Answer:** Arrow functions do NOT have their own `this`. They inherit `this` from the enclosing lexical scope at the time of their creation.

---

## 22. Arrow Functions

### What They Are

Arrow functions are a concise function syntax introduced in ES6. Beyond syntax, they have important behavioral differences from regular functions.

### Syntax Variants

```javascript
// Full form
const add = (a, b) => { return a + b; };

// Implicit return (no braces, single expression)
const add = (a, b) => a + b;

// Single param (no parentheses needed)
const double = x => x * 2;

// No params
const greet = () => "Hello!";

// Returning an object literal — MUST wrap in ()
const makeObj = (name) => ({ name, type: "user" });
// Without () would be parsed as a block!
```

### Key Differences from Regular Functions

| Feature | Regular Function | Arrow Function |
|---|---|---|
| `this` | Dynamic (call-site) | Lexical (definition-site) |
| `arguments` object | ✅ Has it | ❌ No (use `...args`) |
| Constructor (`new`) | ✅ Can use | ❌ Cannot use |
| `prototype` | ✅ Has it | ❌ No |
| Generator (`function*`) | ✅ Can be | ❌ Cannot be |
| Hoisting | Declaration: full | No (variable hoisting only) |

### Arrow Functions and `this` — The Most Important Difference

```javascript
function Timer() {
  this.seconds = 0;

  // ✅ Arrow function: 'this' is lexically bound to Timer instance
  setInterval(() => {
    this.seconds++; // 'this' = Timer instance (correct!)
    console.log(this.seconds);
  }, 1000);
}

const t = new Timer(); // Works correctly

// Contrast with regular function:
function BadTimer() {
  this.seconds = 0;
  setInterval(function() {
    this.seconds++; // 'this' = global/undefined — WRONG!
  }, 1000);
}
```

### Arrow Functions DO NOT Have `arguments`

```javascript
function regular() {
  console.log(arguments); // [Arguments] { '0': 1, '1': 2 }
}
regular(1, 2);

const arrow = () => {
  console.log(arguments); // ReferenceError OR outer function's arguments
};
arrow(1, 2);

// Use rest params instead:
const arrow2 = (...args) => {
  console.log(args); // [1, 2]
};
arrow2(1, 2);
```

### When NOT to Use Arrow Functions

```javascript
// ❌ Object methods (loses 'this' context)
const obj = {
  name: "Alice",
  greet: () => {
    console.log(this.name); // ❌ 'this' is window/undefined
  }
};

// ❌ Event handlers where you need 'this' to be the element
button.addEventListener("click", () => {
  console.log(this); // ❌ 'this' is NOT the button
});

// ❌ Constructor functions
const Person = (name) => { this.name = name; }; // ❌ TypeError
new Person("Alice"); // Arrow functions cannot be constructors

// ❌ Prototype methods
Animal.prototype.speak = () => {
  console.log(this.name); // ❌ 'this' is not the instance
};
```

---

## 23. V8 Internals

### How V8 Stores Lexical Scopes

V8 (Chrome/Node.js JS engine) stores lexical scopes in **Lexical Environments** (spec) mapped to **Context objects** (V8 implementation).

```
Function Execution Context
├── VariableEnvironment   (for var declarations)
├── LexicalEnvironment    (for let/const)
│   ├── Environment Record (stores bindings: { x: 10, y: 20 })
│   └── Outer Reference → Parent LexicalEnvironment
└── ThisBinding
```

### The Scope Tree

When V8 parses code, it builds a **scope tree** — a tree of scope objects representing nested lexical scopes. This is built at **parse time**, before execution.

```
GlobalScope
├── x = undefined (var, hoisted)
└── foo: FunctionScope
    ├── a = undefined (var, hoisted)
    └── bar: FunctionScope
        ├── b (let, TDZ until declaration)
        └── innerFn: FunctionScope
            └── c (const, TDZ)
```

### Context Chaining in V8

```javascript
function outer(x) {
  function inner(y) {
    return x + y; // Accesses outer's x via context chain
  }
  return inner;
}

// V8 Internal representation:
// inner's Context → { y: <arg>, outer: outerContext }
// outerContext → { x: <arg>, outer: globalContext }
// When inner accesses x:
//   1. Check inner's Context → not found
//   2. Follow outer pointer to outerContext → found x ✅
```

### Deopt Killers (V8 Optimization Breakers)

V8 uses **JIT compilation** — it compiles hot code to optimized machine code. Certain patterns cause V8 to "deoptimize" (throw away optimized code and fall back to interpreter). These are called **deopt killers**.

#### 1. `arguments` Object Manipulation

```javascript
// ❌ Deopt: leaking arguments object
function bad(a, b) {
  return arguments; // Leaking args prevents optimization
}

// ❌ Deopt: arguments object aliasing
function bad2(x) {
  arguments[0] = 10; // Aliased mutation
  return x; // x and arguments[0] are aliased — not optimizable
}

// ✅ Use rest params instead
function good(...args) { return args; }
```

#### 2. `eval` and `with`

```javascript
// ❌ eval can introduce new variables at runtime — prevents scope analysis
function withEval(code) {
  eval(code); // V8 can't predict what variables this creates!
  return x;   // Is 'x' from eval? Unknown at compile time → deopt
}

// ❌ with changes scope chain at runtime
function withStatement(obj) {
  with (obj) {
    return x; // Is 'x' from obj or outer scope? Unknown → deopt
  }
}
```

#### 3. Deleting Local Variables

```javascript
// ❌ delete forces V8 to allocate variable in heap
function bad() {
  let x = 10;
  delete x; // (has no effect in strict mode, but still causes deopt)
  return x;
}
```

#### 4. Mixing Types (Hidden Class Violation)

```javascript
// V8 creates "hidden classes" (internal type descriptors) for objects
// ❌ Adding properties in different orders breaks hidden class sharing
function bad(flag) {
  const obj = {};
  if (flag) {
    obj.x = 1;
    obj.y = 2;
  } else {
    obj.y = 2; // Different order!
    obj.x = 1;
  }
  return obj; // Two different hidden classes — can't be optimized uniformly
}

// ✅ Consistent property order
function good(flag) {
  const obj = { x: flag ? 1 : 0, y: 2 }; // Same shape always
  return obj;
}

// ❌ Adding properties after the fact
const obj = {};
obj.a = 1; // Each addition can change hidden class
obj.b = 2;
obj.c = 3;

// ✅ Initialize all properties at once
const obj = { a: 1, b: 2, c: 3 }; // One hidden class
```

#### 5. Polymorphic Function Calls

```javascript
// ❌ V8 can't optimize if function receives different argument shapes
function add(a, b) { return a + b; }
add(1, 2);          // int + int
add("a", "b");      // string + string
add(1, "b");        // mixed — polymorphic site
// After seeing too many types, V8 marks the call as "megamorphic" → slower
```

### Lexical Scope in V8 (Technical Detail)

At parse time, V8 creates a `ScopeInfo` for each function containing:
- List of variables
- Whether each is `var`, `let`, `const`
- Whether variables are captured by inner functions (determines heap vs stack)

At runtime, variables captured by closures are allocated in a **Context** (on the heap), not on the stack. This is why closures survive after the outer function returns.

```javascript
function outer() {
  let x = 10; // ← If captured by inner, allocated on HEAP (Context)
  let y = 20; // ← If NOT captured, can be on STACK (optimized away)
  
  return function inner() {
    return x; // captures x → x goes to heap
    // y is not captured → y stays on stack (or gets optimized away)
  };
}
```

---

## 24. Garbage Collection (GC)

### What It Is

**Garbage Collection** is the automatic process by which the JavaScript engine reclaims memory from objects that are no longer reachable/accessible. JavaScript uses automatic GC — you don't manually free memory.

### Where It Applies

- Objects on the heap (closures, arrays, plain objects, DOM nodes, etc.)
- Closed-over variables
- Event listeners
- Timers

### Where It Does NOT Apply

- Stack-allocated primitives (automatically freed when the stack frame pops)
- Values you explicitly keep referenced (they won't be GC'd as long as reference exists)

### Core Algorithm: Mark-and-Sweep

V8 uses a **generational garbage collector** based on mark-and-sweep:

1. **Mark Phase:** Starting from "GC roots" (global variables, currently executing function variables, native stack), mark all reachable objects.
2. **Sweep Phase:** Collect (free) all unmarked (unreachable) objects.

```
GC Roots
├── Global variables
├── Local variables in active call stack
├── CPU registers
└── These objects are "alive"
    └── Their references are also "alive"
        └── And so on...
        
Everything NOT reachable from roots → GC'd ✅
```

### V8's Generational GC

V8 splits the heap into two generations:

```
V8 Heap
├── Young Generation (Nursery) — small, collected frequently (~1-2ms)
│   ├── New Space (semi-space x2)
│   └── Most objects start here — quickly collected (Minor GC)
│
└── Old Generation — large, collected infrequently
    ├── Old Space (long-lived objects, promoted from young gen)
    ├── Large Object Space (objects > 512KB)
    ├── Code Space (compiled code)
    └── Major GC (Full GC) — uses Orinoco (incremental/parallel)
```

**Promotion:** Objects that survive multiple Young GC cycles are "promoted" to Old Generation.

### Common Memory Leak Patterns

#### 1. Accidental Global Variables

```javascript
function leak() {
  // ❌ In sloppy mode, this creates a global variable
  oopsGlobal = "This lives forever";
}
// Fix: always use 'use strict'
```

#### 2. Forgotten Event Listeners

```javascript
// ❌ Memory leak — handler closes over 'data', both stay alive
function setup() {
  const data = new Array(1e6).fill("large");
  
  document.addEventListener("click", function handler() {
    console.log(data.length); // holds reference to 'data'
  });
  // Event listener never removed → 'data' never GC'd
}

// ✅ Remove listener when done
function setup() {
  const data = new Array(1e6).fill("large");
  
  function handler() { console.log(data.length); }
  document.addEventListener("click", handler);
  
  return () => document.removeEventListener("click", handler); // cleanup
}
```

#### 3. Forgotten Timers

```javascript
// ❌ Interval closure holds reference forever
function startBadTimer() {
  const data = getLargeData();
  
  setInterval(() => {
    process(data); // 'data' never GC'd as long as interval runs
  }, 1000);
  // No way to stop it!
}

// ✅ Keep the ID and clear it when done
function startGoodTimer() {
  const data = getLargeData();
  const id = setInterval(() => process(data), 1000);
  return () => clearInterval(id); // cleanup function
}
```

#### 4. Closures Holding Large Objects

```javascript
// ❌ 'element' holds a reference to the entire DOM subtree
function bad() {
  const element = document.getElementById("bigElement");
  
  return function() {
    console.log(element.textContent); // element never freed
  };
}

// ✅ Capture only what you need
function good() {
  const text = document.getElementById("bigElement").textContent;
  
  return function() {
    console.log(text); // only the string is captured
  };
}
```

#### 5. Detached DOM Nodes

```javascript
// ❌ Removed from DOM but still referenced in JS
const cache = {};

function bad() {
  const el = document.getElementById("btn");
  cache.button = el; // Still referenced!
  el.remove(); // Removed from DOM, but NOT GC'd — still in cache
}

// ✅ Use WeakRef or WeakMap for DOM references
const cache = new WeakMap();

function good() {
  const el = document.getElementById("btn");
  cache.set(el, { created: Date.now() });
  el.remove(); // Once el has no other references, GC can collect
}
```

### WeakMap and WeakSet — GC-Friendly Containers

```javascript
// WeakMap: keys are weakly held — don't prevent GC of key objects
const metadata = new WeakMap();

let obj = { name: "Alice" };
metadata.set(obj, { created: Date.now() });

// When obj is set to null, BOTH obj AND the metadata entry get GC'd
obj = null; // ✅ GC can now collect the object and its metadata

// WeakSet: weakly holds objects
const visited = new WeakSet();
function visit(obj) {
  if (visited.has(obj)) return;
  visited.add(obj);
  // When obj becomes unreachable, it's removed from WeakSet automatically
}
```

### Real Interview Questions

**Q1:** What is the difference between `WeakMap` and `Map` regarding GC?
**Answer:** `Map` holds strong references to its keys — they won't be GC'd while they're in the Map. `WeakMap` holds weak references — if the key object has no other references, it can be GC'd even if still in the WeakMap.

**Q2:** How can closures cause memory leaks?
**Answer:** If a closure captures a reference to a large object and the closure itself lives longer than expected (stored globally, or in a never-removed event listener), the large object cannot be GC'd.

**Q3:** What is the generational hypothesis in GC?
**Answer:** Most objects die young — they're created, used briefly, and become garbage quickly. V8 exploits this by having a small, frequently-collected Young Generation for new objects.

**Q4:** Can you force GC in JavaScript?
**Answer:** No — you cannot force GC in production JavaScript. `gc()` is only available in V8 with `--expose-gc` flag for debugging. The best approach is to write GC-friendly code (remove references, clean up listeners/timers).

---

## Quick Reference Cheat Sheet

### Variable Declarations at a Glance

| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function | Block | Block |
| Hoisted | `undefined` | TDZ | TDZ |
| Re-declare | ✅ | ❌ | ❌ |
| Re-assign | ✅ | ✅ | ❌ |
| Global `window` prop | ✅ | ❌ | ❌ |

### `this` Binding Rules

| Call Type | `this` Value |
|---|---|
| Standalone `foo()` | `undefined` (strict) / `window` |
| Method `obj.foo()` | `obj` |
| `foo.call(ctx)` | `ctx` |
| `new Foo()` | New object |
| Arrow `() => {}` | Lexically inherited |

### Closure Capture Summary

| Variable type | Captured? | Location |
|---|---|---|
| Referenced local | ✅ Yes | Heap (Context object) |
| Unreferenced local | ❌ No | Stack (GC'd with frame) |
| Parameters | ✅ If referenced | Heap |

### Memory Leak Checklist

- [ ] Remove event listeners when component unmounts
- [ ] Clear intervals/timeouts when no longer needed
- [ ] Avoid capturing large objects in long-lived closures
- [ ] Use `WeakMap`/`WeakSet` for DOM → data associations
- [ ] Use `'use strict'` to prevent accidental globals
- [ ] Null out large object references when done

---

*Last updated: 2026 | Covers ES2022+ features including private class fields, optional chaining, and nullish coalescing*