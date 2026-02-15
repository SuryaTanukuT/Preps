
---

## 1) Prototype & Prototype Inheritance

### ✅ What it is

JS uses **prototype-based inheritance**: objects inherit from other objects via an internal link called `[[Prototype]]`.

### ✅ Where it applies

* Sharing methods across many objects (memory efficient)
* Inheritance for “instances” created via `new` / `class`
* Built-ins: Array, Date, Function all use prototypes

### ❌ Where it does NOT

* Not copy-based inheritance (it’s a *link*, not cloning)
* Not “classical inheritance” semantics like Java/C# (JS classes are syntax sugar)

### 🧠 Core principles

* Property access: **own property → prototype → prototype of prototype → … → null**
* Writes go to the object itself (unless setter exists)
* Deleting own property reveals inherited one

### 🧩 Mental model

Think of a hidden pointer:
`obj → protoObj → protoProtoObj → null`

### 🧪 Examples

```js
const animal = { eats: true };
const dog = Object.create(animal);
dog.barks = true;

console.log(dog.eats); // true (inherited)
console.log(dog.hasOwnProperty("eats")); // false
```

### ✅ Use cases & Best practices

* Put shared functions on prototype (not recreated per instance)
* Prefer composition if inheritance chain becomes deep/complex
* Use `Object.create(null)` for “dictionary objects”

### ⚠️ Tricky edge cases

```js
const base = { x: 1 };
const obj = Object.create(base);

obj.x = 2;        // shadows base.x
delete obj.x;     // reveals base.x again
console.log(obj.x); // 1
```

### 🎯 Interview questions

* How does JS find `dog.eats`?
* What’s shadowing in prototypes?
* Inheritance vs composition in real apps?

---

## 2) `__proto__` vs `.prototype` vs `[[Prototype]]`

### ✅ What it is

* `[[Prototype]]` = internal hidden slot (real concept)
* `__proto__` = legacy getter/setter exposing `[[Prototype]]` (avoid)
* `.prototype` = property on **functions**, used when calling with `new`

### ✅ Where it applies

* Understanding `new`, `class`, inheritance
* Debugging prototype chain

### ❌ Where it does NOT

* `.prototype` on non-functions doesn’t affect inheritance
* `__proto__` shouldn’t be used in production

### 🧠 Core principles

* Instances created by `new F()` get `[[Prototype]] = F.prototype`
* `Object.getPrototypeOf(obj)` is the standard way to read prototype

### 🧩 Mental model

* `F.prototype` = “prototype assigned to future instances”
* `obj.__proto__ / [[Prototype]]` = “prototype obj currently inherits from”

### 🧪 Examples

```js
function A() {}
const a = new A();

console.log(Object.getPrototypeOf(a) === A.prototype); // true
console.log(a.__proto__ === A.prototype); // true (legacy)
```

### ✅ Use cases & Best practices

* Prefer:

  * `Object.getPrototypeOf(obj)`
  * `Object.create(proto)`
* Avoid `Object.setPrototypeOf` on hot objects (slow)

### ⚠️ Tricky edge cases

```js
const o = Object.create(null);
console.log(o.__proto__); // undefined (no Object.prototype in chain)
```

### 🎯 Interview questions

* Explain `.prototype` vs `__proto__` with an example.
* Why is changing prototype at runtime slow?

---

## 3) Constructor Functions

### ✅ What it is

A function used with `new` to create objects and set their prototype.

### ✅ Where it applies

* Legacy OOP patterns
* Libraries relying on prototype methods
* Understanding ES6 classes internally

### ❌ Where it does NOT

* Calling constructor without `new` (strict mode breaks)
* Not needed if you use factory functions / composition

### 🧠 Core principles

`new Foo()` does:

1. create empty object
2. set prototype: `obj.[[Prototype]] = Foo.prototype`
3. call `Foo` with `this = obj`
4. return obj (unless Foo returns an object explicitly)

### 🧩 Mental model

`new` = “allocate + link prototype + initialize”

### 🧪 Examples

```js
function User(name) {
  this.name = name;
}
User.prototype.sayHi = function () {
  return `Hi ${this.name}`;
};

const u = new User("Surya");
console.log(u.sayHi());
```

### ✅ Use cases & Best practices

* Put methods on `User.prototype`
* New-safety:

```js
function SafeUser(name) {
  if (!new.target) return new SafeUser(name);
  this.name = name;
}
```

### ⚠️ Tricky edge cases

```js
function X() { this.a = 1; return { a: 99 }; }
console.log(new X().a); // 99 (returned object wins)
```

### 🎯 Interview questions

* What does `new.target` mean?
* What happens if constructor returns a primitive?

---

## 4) Prototype Chain

### ✅ What it is

The linked chain used for property/method resolution.

### ✅ Where it applies

* `instanceof`, method lookup, built-in methods
* Polyfills, monkey-patching (careful)

### ❌ Where it does NOT

* Does not “copy methods” into the object
* Not used for private fields (`#x`) in classes

### 🧠 Core principles

* Lookup climbs the chain until found or reaches `null`
* `hasOwnProperty` checks only the object, not prototypes

### 🧩 Mental model

`obj.key` = “search here, else ask prototype, repeat”

### 🧪 Examples

```js
const arr = [];
// arr → Array.prototype → Object.prototype → null
console.log(arr.toString); // from Array.prototype / Object.prototype
```

### ✅ Use cases & Best practices

* Prefer `Object.hasOwn(obj, key)` (modern) or `hasOwnProperty.call`
* Avoid deep inheritance chains

### ⚠️ Tricky edge cases

```js
const dict = Object.create(null);
console.log(dict.hasOwnProperty); // undefined
// Safe check:
console.log(Object.prototype.hasOwnProperty.call(dict, "x"));
```

### 🎯 Interview questions

* How does `instanceof` work internally?
* Why does `[] instanceof Object` return true?

---

## 5) Prototype Edge Cases & V8 De-optimizations

### ✅ What it is

V8 optimizes objects with **hidden classes** and **inline caches**. Prototype mutations and unpredictable object shapes can deoptimize hot code.

### ✅ Where it applies

* Performance-sensitive code
* Large React apps (object shapes, megamorphic callsites)
* Node.js services under load

### ❌ Where it does NOT

* Small scripts where performance doesn’t matter

### 🧠 Core principles

* Consistent property addition order → stable hidden classes
* Changing prototypes dynamically → slows property access
* `delete` on hot objects can degrade performance

### 🧩 Mental model

V8 likes predictable “shapes”. If you keep reshaping objects, it can’t optimize.

### 🧪 Examples

```js
function A(){ this.x = 1; this.y = 2; } // order: x then y
function B(){ this.y = 2; this.x = 1; } // order: y then x (different shape)
```

### ✅ Use cases & Best practices

* Initialize properties in same order for all instances
* Avoid `delete` in hot paths; use `obj.key = null`
* Don’t mutate prototypes after objects are created

### ⚠️ Tricky edge cases

* Too many different object shapes passed into the same function → **megamorphic** → slower

### 🎯 Interview questions

* What are hidden classes?
* Why is `Object.setPrototypeOf` slow?
* Why is `delete` slow?

---

## 6) Currying

### ✅ What it is

Transform `f(a,b,c)` into `f(a)(b)(c)`.

### ✅ Where it applies

* Reusable utilities, functional pipelines
* Partial application patterns

### ❌ Where it does NOT

* When it reduces readability in business logic

### 🧠 Core principles

* Returns functions until enough args are collected
* Often uses closures

### 🧩 Mental model

“Fix some inputs now, supply rest later.”

### 🧪 Examples

```js
const curry = (fn) => function curried(...args) {
  return args.length >= fn.length
    ? fn(...args)
    : (...rest) => curried(...args, ...rest);
};

const add3 = (a,b,c) => a+b+c;
const cadd3 = curry(add3);

console.log(cadd3(1)(2)(3)); // 6
console.log(cadd3(1,2)(3));  // 6
```

### ✅ Use cases & Best practices

* Best for validators/formatters/loggers
* Don’t over-curry everything; keep team readability

### ⚠️ Tricky edge cases

* `fn.length` breaks when default params exist
* Currying with variadic functions needs custom logic

### 🎯 Interview questions

* Currying vs partial application?
* Implement curry supporting mixed calls

---

## 7) Functional Programming (FP) Concepts in JS

### ✅ What it is

A style focusing on:

* **pure functions**
* **immutability**
* **higher-order functions**
* **composition**

### ✅ Where it applies

* React state, reducers
* data transformations
* predictable code and testability

### ❌ Where it does NOT

* When it adds complexity for simple flows
* Heavy FP without team alignment

### 🧠 Core principles

* Pure function: same input → same output, no side effects
* Higher-order: functions that take/return functions
* Composition: combine small functions to build bigger ones

### 🧩 Mental model

Treat data as flowing through a pipeline.

### 🧪 Examples

```js
const pipe = (...fns) => (x) => fns.reduce((v, fn) => fn(v), x);

const inc = x => x + 1;
const double = x => x * 2;

console.log(pipe(inc, double)(3)); // 8
```

### ✅ Use cases & Best practices

* Use `map/filter/reduce` for transformation
* Avoid mutating shared state in async workflows

### ⚠️ Tricky edge cases

* Overusing `reduce` can become unreadable
* Purity breaks with hidden dependencies (globals)

### 🎯 Interview questions

* What makes a function pure?
* Explain function composition with code

---

## 8) `call`, `apply`, `bind`

### ✅ What it is

Ways to control `this` and arguments.

* `call(this, a, b)`
* `apply(this, [a, b])`
* `bind(this, a)` returns new function

### ✅ Where it applies

* Borrowing methods
* Setting `this` in callbacks
* Partial application

### ❌ Where it does NOT

* Arrow functions ignore `bind/call/apply` for `this`

### 🧠 Core principles

* `bind` creates a new function with fixed `this`
* `call/apply` invoke immediately

### 🧩 Mental model

“Choose what `this` points to.”

### 🧪 Examples

```js
function greet(msg) { return `${msg}, ${this.name}`; }
const user = { name: "Surya" };

console.log(greet.call(user, "Hi"));
console.log(greet.apply(user, ["Hi"]));
const g = greet.bind(user, "Hi");
console.log(g());
```

### ✅ Use cases & Best practices

* Use arrow functions in callbacks to avoid losing `this`
* In classes, bind in constructor only if needed

### ⚠️ Tricky edge cases

* `bind` can’t be overridden:

```js
const bound = greet.bind(user);
bound.call({name:"X"}); // still Surya
```

### 🎯 Interview questions

* Implement bind polyfill
* Why arrows don’t have their own `this`?

---

## 9) Debouncing

### ✅ What it is

Delay execution until events stop firing for a period.

### ✅ Where it applies

* search input
* resize
* autosave

### ❌ Where it does NOT

* If you need steady updates (use throttle)

### 🧠 Core principles

Clear previous timer, set a new one.

### 🧩 Mental model

“Run only after the user pauses.”

### 🧪 Example

```js
const debounce = (fn, delay) => {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), delay);
  };
};
```

### ✅ Best practices

* Add `leading/trailing` options if needed
* Cancel on unmount (React)

### ⚠️ Tricky edge cases

* Debounced function needs `.cancel()` sometimes
* `this` binding inside debounce wrapper

### 🎯 Interview questions

* Debounce with `immediate` option?
* Debounce vs throttle differences?

---

## 10) Throttling

### ✅ What it is

Run at most once per interval.

### ✅ Where it applies

* scroll
* mousemove
* analytics (rate limiting)

### ❌ Where it does NOT

* If last update must always happen (need trailing support)

### 🧠 Core principles

Allow execution only after `wait` time.

### 🧩 Mental model

“Limit rate.”

### 🧪 Example

```js
const throttle = (fn, wait) => {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= wait) {
      last = now;
      fn(...args);
    }
  };
};
```

### ✅ Best practices

* Prefer requestAnimationFrame throttle for UI updates

### ⚠️ Tricky edge cases

* Need “trailing call” guarantee → more advanced throttle

### 🎯 Interview questions

* Throttle scroll handler best approach?
* Build throttle with leading/trailing options

---

## 11) `async` vs `defer` (Script Loading)

### ✅ What it is

* `defer`: downloads in parallel, executes after HTML parsing, **in order**
* `async`: downloads in parallel, executes ASAP, **order not guaranteed**

### ✅ Where it applies

* Performance optimization for script loading

### ❌ Where it does NOT

* `async` for dependency scripts (can break order)

### 🧠 Core principles

* `defer` waits for DOM parse
* `async` doesn’t wait, can execute mid-parse

### 🧩 Mental model

* `defer` = “safe parallel load”
* `async` = “fast but unpredictable execution order”

### 🧪 Example

```html
<script defer src="vendor.js"></script>
<script defer src="app.js"></script>
```

### ✅ Best practices

* Use `defer` for app scripts
* Use `async` for analytics/ads that don’t depend on other scripts

### ⚠️ Tricky edge cases

* `defer` scripts run before `DOMContentLoaded`
* `async` may run before DOM exists

### 🎯 Interview questions

* Why does defer preserve order but async doesn’t?
* Which one is better for React app bundle?

---

## 12) Web APIs (Browser Provided APIs)

### ✅ What it is

APIs provided by the browser environment (not the JS engine):

* DOM, timers, fetch/XHR, storage, media, geolocation, etc.

### ✅ Where it applies

* Frontend apps
* Async IO (fetch, timers)

### ❌ Where it does NOT

* JS language itself (ECMAScript) doesn’t include fetch/timers

### 🧠 Core principles

* JS runs on one main thread
* Web APIs do async work, then enqueue callbacks via event loop queues

### 🧩 Mental model

JS = “brain”, Web APIs = “browser assistants”, event loop = “scheduler”.

### 🧪 Example

```js
setTimeout(() => console.log("timer"), 0);
```

`setTimeout` is Web API; callback runs later via event loop.

### ✅ Best practices

* Avoid heavy work on main thread (use Web Workers for CPU tasks)

### ⚠️ Tricky edge cases

* Timers are not precise (clamping, throttling in background tabs)

### 🎯 Interview questions

* Is setTimeout part of JS?
* Why fetch doesn’t block execution?

---

## 13) DOM Lifecycle (Render Pipeline)

### ✅ What it is

How browser turns HTML/CSS into pixels.

### ✅ Where it applies

* Performance (reflow/repaint)
* DOM manipulation impacts

### ❌ Where it does NOT

* Node.js (no DOM pipeline)

### 🧠 Core principles

* HTML → DOM
* CSS → CSSOM
* DOM + CSSOM → Render tree
* Layout → Paint → Composite

### 🧩 Mental model

“DOM changes can trigger layout and repaint.”

### 🧪 Example (layout thrash risk)

```js
el.style.width = "100px";
const h = el.offsetHeight; // forces layout
el.style.height = (h + 10) + "px";
```

### ✅ Best practices

* Batch DOM reads then writes
* Use CSS transforms for animations (compositor-friendly)

### ⚠️ Tricky edge cases

* Measuring layout (`offsetHeight`) forces sync layout

### 🎯 Interview questions

* Difference between reflow vs repaint?
* Why transforms are faster than top/left animations?

---

## 14) DOM Events + Bubbling/Capturing/Delegation

### ✅ What it is

Event propagation phases:

1. Capture
2. Target
3. Bubble

### ✅ Where it applies

* UI handling
* Delegation patterns

### ❌ Where it does NOT

* Some events don’t bubble (e.g., `focus`/`blur`), use `focusin/focusout`

### 🧠 Core principles

* Bubbling: child → parent
* Capturing: parent → child
* Delegation uses bubbling to handle many items with one listener

### 🧩 Mental model

Event “travels” through ancestors.

### 🧪 Examples

```js
parent.addEventListener("click", () => console.log("bubble"));
parent.addEventListener("click", () => console.log("capture"), true);
```

Delegation:

```js
list.addEventListener("click", (e) => {
  const li = e.target.closest("li");
  if (!li) return;
  console.log(li.dataset.id);
});
```

### ✅ Best practices

* Delegate for large lists
* Use `{ passive: true }` for scroll/touch performance where safe

### ⚠️ Tricky edge cases

* `stopPropagation` vs `stopImmediatePropagation`
* Shadow DOM retargeting (advanced)

### 🎯 Interview questions

* Why delegation improves performance?
* Capturing use cases?

---

## 15) BOM vs DOM

### ✅ What it is

* DOM: `document`, elements
* BOM: `window`, `location`, `history`, `navigator`

### ✅ Where it applies

* Routing without reload (`history.pushState`)
* Reading URL params (`location.search`)
* Browser capabilities (`navigator`)

### ❌ Where it does NOT

* Node.js environment doesn’t have BOM

### 🧪 Example

```js
history.pushState({}, "", "/new-page");
```

### 🎯 Interview questions

* What’s difference between BOM and DOM?
* How SPA routing works using history API?

---

## 16) Network / Communication (Fetch)

### ✅ What it is

Browser API to make network requests (Promise-based).

### ✅ Where it applies

* REST/GraphQL calls
* Streaming responses (advanced)

### ❌ Where it does NOT

* Doesn’t reject on HTTP 404/500 (only rejects on network failure)

### 🧠 Core principles

* Fetch resolves on HTTP response; check `res.ok`
* CORS enforced by browser

### 🧪 Example

```js
const res = await fetch("/api");
if (!res.ok) throw new Error("HTTP error");
const data = await res.json();
```

### 🎯 Interview questions

* Why fetch doesn’t throw on 404?
* Fetch vs XHR?

---

## 17) CORS + Preflight (OPTIONS)

### ✅ What it is

CORS is a browser security policy controlling cross-origin reads.

### ✅ Where it applies

* Frontend calling APIs on different origin
* Cookies/credentials across origins

### ❌ Where it does NOT

* Server-to-server requests (CORS is browser-enforced)

### 🧠 Core principles

* “Simple requests” may skip preflight
* “Non-simple” requests trigger OPTIONS preflight

Triggers typically include:

* method not GET/HEAD/POST
* custom headers
* `Content-Type: application/json` (commonly causes preflight)

### 🧩 Mental model

Browser asks server permission first (OPTIONS), then sends real request if allowed.

### 🧪 Example: required headers

Server must respond with:

* `Access-Control-Allow-Origin`
* `Access-Control-Allow-Methods`
* `Access-Control-Allow-Headers`

### ✅ Best practices

* Don’t use `*` with credentials
* Use reverse proxy in dev for same-origin
* Cache preflight with `Access-Control-Max-Age` (careful)

### ⚠️ Tricky edge cases

* `credentials: "include"` requires:

  * `Access-Control-Allow-Credentials: true`
  * and specific origin (not `*`)

### 🎯 Interview questions

* Why browser sends OPTIONS even if you didn’t?
* Why Postman works but browser fails?

---

## 18) Storage (Browser)

### ✅ What it is

Client-side persistence:

* cookies
* localStorage
* sessionStorage
* IndexedDB

### ✅ Where it applies

* Auth state (careful)
* Caching, preferences
* Offline apps (IndexedDB)

### ❌ Where it does NOT

* localStorage/sessionStorage not available in server rendering (SSR)

### 🧠 Core principles

* cookies: sent with requests; secure flags matter
* localStorage: synchronous, persistent, per-origin
* sessionStorage: per-tab session
* IndexedDB: async, large structured storage

### ✅ Best practices

* Prefer HttpOnly secure cookies for sensitive tokens (common approach)
* Use `SameSite`, `Secure`, `HttpOnly` where possible

### ⚠️ Tricky edge cases

* localStorage is synchronous → can block main thread if abused
* iOS/Safari storage quirks in private mode (practical pain)

### 🎯 Interview questions

* cookie vs localStorage differences?
* Why HttpOnly cookies are safer vs XSS?

---

## 19) Browser Environment + Execution Context Lifecycle

### ✅ What it is

How JS runs:

* global execution context created
* function execution contexts created per call
* each has scope bindings + `this`

### ✅ Where it applies

* Hoisting, TDZ, scope chain
* Debugging closures, `this`

### ❌ Where it does NOT

* Doesn’t mean multi-threaded execution (still single-thread for JS)

### 🧠 Core principles

* Lexical environment holds identifiers
* Scope chain resolves variables
* `this` depends on call-site (except arrows)

### 🧩 Mental model

Each function call gets its own “execution frame” with local variables.

### 🧪 Example

```js
let x = 1;
function f() {
  let y = 2;
  return x + y;
}
```

### 🎯 Interview questions

* What is scope chain?
* How `this` is decided?

---

## 20) Why TDZ exists (Temporal Dead Zone)

### ✅ What it is

`let/const` exist in scope but are **uninitialized** until declaration is executed → accessing before that throws `ReferenceError`.

### ✅ Where it applies

* `let`, `const`, `class` declarations

### ❌ Where it does NOT

* `var` (hoisted + initialized to `undefined`)

### 🧠 Core principles

* Prevents “use before declaration” bugs
* Enables block scoping semantics cleanly

### 🧩 Mental model

Name exists, but “not ready” until line executes.

### 🧪 Example

```js
console.log(a); // ReferenceError (TDZ)
let a = 10;
```

### ⚠️ Tricky edge case

```js
let x = 1;
(function(){
  console.log(x); // ReferenceError (inner x in TDZ)
  let x = 2;
})();
```

### 🎯 Interview questions

* Why `var` prints undefined but `let` throws?
* When does TDZ start/end?

---

## 21) JS Engine (V8)

### ✅ What it is

V8 is a JS engine that parses, compiles (JIT), optimizes code, manages memory (GC).

### ✅ Where it applies

* Node.js and Chromium-based browsers
* Performance & memory tuning

### ❌ Where it does NOT

* Not universal (Safari uses JavaScriptCore, Firefox uses SpiderMonkey)

### 🧠 Core principles

* JIT: interpret/compile → optimize hot code
* Hidden classes & inline caches optimize property access
* GC reclaims unused memory

### 🧩 Mental model

Code becomes faster after it runs a lot (“warming up”).

### 🎯 Interview questions

* What is JIT?
* What are hidden classes / inline caches?

---

## 22) Browser Internals: Event Loop + Task Queues (Deep)

### ✅ What it is

Scheduler that decides when callbacks run.

### ✅ Where it applies

* async behavior, promises, timers, UI events

### ❌ Where it does NOT

* Doesn’t mean parallelism (JS still single-threaded)

### 🧠 Core principles

* Macrotask queue: timers, UI events, message events
* Microtask queue: promise callbacks, `queueMicrotask`, MutationObserver
* After each macrotask: **drain all microtasks**, then render

### 🧩 Mental model

Macrotask = big steps; Microtask = urgent small steps executed before next big step.

### 🧪 Example

```js
console.log("A");
setTimeout(()=>console.log("T"), 0);
Promise.resolve().then(()=>console.log("P"));
console.log("B");
// A B P T
```

### ⚠️ Tricky edge cases

* Infinite microtasks can freeze UI
* `await` resumes via microtasks

### 🎯 Interview questions

* Predict output puzzles
* Why promise callbacks run before setTimeout(0)?

---

## 23) JS Engine & Performance (Hidden classes, GC, Memory leaks)

### ✅ What it is

Real-world performance topics:

* hidden classes shape stability
* garbage collection behavior
* leaks in SPAs

### ✅ Where it applies

* high-scale frontend and Node services
* diagnosing memory spikes and CPU issues

### 🧠 Core principles

* Stable object shapes → faster
* Too many shapes at same callsite → megamorphic (slower)
* GC cost grows with long-lived objects

### ✅ Common memory leaks

* Detached DOM nodes still referenced
* Event listeners not removed
* Intervals not cleared
* Growing caches/maps without eviction

### 🎯 Interview questions

* Why closures can cause memory leaks?
* How to detect leaks (heap snapshots conceptually)?

---

## 24) Array & Object Methods (Must know)

### ✅ Arrays

* Transform: `map`, `flatMap`
* Filter: `filter`
* Reduce: `reduce`
* Search: `find`, `findIndex`, `some`, `every`, `includes`
* Slice vs splice (non-mutating vs mutating)
* Sort pitfalls

```js
[1, 2, 10].sort(); // [1,10,2]
[1, 2, 10].sort((a,b)=>a-b); // [1,2,10]
```

### ✅ Objects

* `Object.keys/values/entries`
* `Object.assign` (shallow)
* `Object.freeze` (shallow)
* `Object.create`, `Object.getPrototypeOf`

### 🎯 Interview questions

* Group by using reduce
* `for..in` vs `for..of`

---

## 25) Immutability in JS

### ✅ What it is

Don’t mutate existing objects/arrays; create new versions.

### ✅ Where it applies

* React state updates
* Redux reducers
* concurrency-safe patterns

### ❌ Where it does NOT

* If performance requires mutation and it’s isolated/controlled (careful)

### 🧠 Core principles

* “New reference” signals change
* Structural sharing (copy only changed parts)

### 🧪 Examples

```js
const next = { ...prev, a: 2 };
const nextArr = prevArr.map(x => x.id === id ? {...x, done:true} : x);
```

### ⚠️ Tricky edge cases

* Spread is shallow, nested objects still shared

### 🎯 Interview questions

* Why immutability helps React?
* Shallow vs deep immutability?

---

## 26) Shallow Copy vs Deep Copy

### ✅ Shallow copy

Copies top-level only.

* `{...obj}`, `Object.assign`, `arr.slice()`

### ✅ Deep copy

* `structuredClone(obj)` (modern best)
* JSON clone (lossy)
* custom deep clone

```js
const deep = structuredClone(obj);
```

### ⚠️ Tricky edge cases

* JSON drops Date, Map, Set, undefined, functions
* Cyclic objects need special handling

### 🎯 Interview questions

* Why JSON deep clone is dangerous?
* How to clone cyclic object?

---

## 27) Error Handling

### ✅ What it is

Handling exceptions and failures safely.

### ✅ Where it applies

* API calls, parsing, business logic
* async flows (await + try/catch)

### 🧠 Core principles

* `try/catch` catches sync errors and awaited async errors
* Promise chains need `.catch` unless awaited

### 🧪 Examples

```js
try {
  await doWork();
} catch (e) {
  // handle
} finally {
  // cleanup
}
```

### ✅ Best practices

* Don’t swallow errors
* Add context and error codes
* Prefer typed/custom errors in apps

### 🎯 Interview questions

* Difference between thrown error and rejected promise?
* What is `unhandledrejection`?

---

## 28) Overloading & Overriding (JS Reality)

### ✅ Overriding

Supported via prototype chain / class inheritance.

### ❌ Overloading

No real signature overloading in JS. Simulate using:

* optional params
* rest params
* checking `arguments.length`

```js
function sum(...nums) {
  return nums.reduce((a,b)=>a+b,0);
}
```

### 🎯 Interview questions

* Why JS doesn’t support overloads like Java?
* Design API without overload confusion

---

## 29) Reference vs Value

### ✅ Value (primitives)

Copied by value:
`number, string, boolean, null, undefined, symbol, bigint`

### ✅ Reference (objects/functions/arrays)

Copied by reference (same object).

```js
const a = { x: 1 };
const b = a;
b.x = 2;
console.log(a.x); // 2
```

### 🎯 Interview questions

* Why `const obj` can mutate?
* How to truly prevent mutation?

---

## 30) OOP in JS

### ✅ What it is

JS supports OOP with:

* prototypes
* classes (syntax sugar)
* encapsulation patterns (closures, private fields)

### ✅ Where it applies

* domain modeling
* frameworks/classes

### ❌ Where it does NOT

* Not required for everything; FP/composition often better

### 🎯 Interview questions

* Class vs prototype-based inheritance?
* Private fields `#x` vs closure privacy?

---

## 31) Compile Time vs Run Time (JS Context)

### ✅ What it is

JS is JIT compiled:

* parsing and compilation happen before and during execution

### ✅ Compile-time-ish

* Syntax errors (parser catches before running)

### ✅ Runtime

* TypeError, ReferenceError etc happen when executing

### 🎯 Interview questions

* What is JIT compilation?
* Why TypeScript types don’t exist at runtime?

---

## 32) Polyfills & Transpilation

### ✅ What it is

* Transpile: convert modern syntax → older JS (Babel/SWC/TS)
* Polyfill: implement missing APIs at runtime (core-js, etc.)

### 🧠 Mental model

* Syntax compatibility → transpile
* API compatibility → polyfill

### ✅ Best practices

* Use `browserslist` targets
* Prefer usage-based polyfills (only what you use)
* Avoid global pollution in libraries

### 🎯 Interview questions

* Transpilation vs polyfill?
* Why code works in Chrome but not Safari?

---

## 33) JS Tooling Ecosystem (Webpack, Babel, ESLint, Prod polyfills)

### ✅ What it is

Build + quality pipeline:

* bundlers: Webpack, Vite, Rollup, esbuild
* transpilers: Babel, SWC
* lint: ESLint
* format: Prettier
* test: Jest/Vitest

### ✅ Where it applies

* production builds, performance, compatibility, CI/CD

### 🎯 Interview questions

* Why bundling is needed?
* Why Babel is still used with modern browsers?

---

# 🔥 Practice: “Tricky Output” Interview Set

1.

```js
console.log(1);
setTimeout(()=>console.log(2), 0);
Promise.resolve().then(()=>console.log(3));
console.log(4);
```

2.

```js
const a = {x:1};
const b = Object.create(a);
console.log(b.x);
b.x = 2;
delete b.x;
console.log(b.x);
```

3.

```js
const obj = { x: 10, f() { return this.x; } };
const g = obj.f;
console.log(g());
```

4.

```js
let x = 1;
(function(){
  console.log(x);
  let x = 2;
})();
```

---
# 🧠 JavaScript Advanced Deep Dive — Interview Master Guide (Part-2 Format)

> Every topic follows the exact structure:
> **What it is → Where it applies → Where it does NOT → Core Principles → Mental Model → Examples → Use cases & Best practices → Tricky edge cases → Interview questions**

---

## 1) Prototype & Prototype Inheritance

### ✅ What it is

JS uses **prototype-based inheritance**: objects inherit from other objects via an internal link called `[[Prototype]]`.

### ✅ Where it applies

* Sharing methods across many objects (memory efficient)
* Inheritance for “instances” created via `new` / `class`
* Built-ins: Array, Date, Function all use prototypes

### ❌ Where it does NOT

* Not copy-based inheritance (it’s a *link*, not cloning)
* Not “classical inheritance” semantics like Java/C# (JS classes are syntax sugar)

### 🧠 Core principles

* Property access: **own property → prototype → prototype of prototype → … → null**
* Writes go to the object itself (unless setter exists)
* Deleting own property reveals inherited one

### 🧩 Mental model

Think of a hidden pointer:
`obj → protoObj → protoProtoObj → null`

### 🧪 Examples

```js
const animal = { eats: true };
const dog = Object.create(animal);
dog.barks = true;

console.log(dog.eats); // true (inherited)
console.log(dog.hasOwnProperty("eats")); // false
```

### ✅ Use cases & Best practices

* Put shared functions on prototype (not recreated per instance)
* Prefer composition if inheritance chain becomes deep/complex
* Use `Object.create(null)` for “dictionary objects”

### ⚠️ Tricky edge cases

```js
const base = { x: 1 };
const obj = Object.create(base);

obj.x = 2;        // shadows base.x
delete obj.x;     // reveals base.x again
console.log(obj.x); // 1
```

### 🎯 Interview questions

* How does JS find `dog.eats`?
* What’s shadowing in prototypes?
* Inheritance vs composition in real apps?

---

## 2) `__proto__` vs `.prototype` vs `[[Prototype]]`

### ✅ What it is

* `[[Prototype]]` = internal hidden slot (real concept)
* `__proto__` = legacy getter/setter exposing `[[Prototype]]` (avoid)
* `.prototype` = property on **functions**, used when calling with `new`

### ✅ Where it applies

* Understanding `new`, `class`, inheritance
* Debugging prototype chain

### ❌ Where it does NOT

* `.prototype` on non-functions doesn’t affect inheritance
* `__proto__` shouldn’t be used in production

### 🧠 Core principles

* Instances created by `new F()` get `[[Prototype]] = F.prototype`
* `Object.getPrototypeOf(obj)` is the standard way to read prototype

### 🧩 Mental model

* `F.prototype` = “prototype assigned to future instances”
* `obj.__proto__ / [[Prototype]]` = “prototype obj currently inherits from”

### 🧪 Examples

```js
function A() {}
const a = new A();

console.log(Object.getPrototypeOf(a) === A.prototype); // true
console.log(a.__proto__ === A.prototype); // true (legacy)
```

### ✅ Use cases & Best practices

* Prefer:

  * `Object.getPrototypeOf(obj)`
  * `Object.create(proto)`
* Avoid `Object.setPrototypeOf` on hot objects (slow)

### ⚠️ Tricky edge cases

```js
const o = Object.create(null);
console.log(o.__proto__); // undefined (no Object.prototype in chain)
```

### 🎯 Interview questions

* Explain `.prototype` vs `__proto__` with an example.
* Why is changing prototype at runtime slow?

---

## 3) Constructor Functions

### ✅ What it is

A function used with `new` to create objects and set their prototype.

### ✅ Where it applies

* Legacy OOP patterns
* Libraries relying on prototype methods
* Understanding ES6 classes internally

### ❌ Where it does NOT

* Calling constructor without `new` (strict mode breaks)
* Not needed if you use factory functions / composition

### 🧠 Core principles

`new Foo()` does:

1. create empty object
2. set prototype: `obj.[[Prototype]] = Foo.prototype`
3. call `Foo` with `this = obj`
4. return obj (unless Foo returns an object explicitly)

### 🧩 Mental model

`new` = “allocate + link prototype + initialize”

### 🧪 Examples

```js
function User(name) {
  this.name = name;
}
User.prototype.sayHi = function () {
  return `Hi ${this.name}`;
};

const u = new User("Surya");
console.log(u.sayHi());
```

### ✅ Use cases & Best practices

* Put methods on `User.prototype`
* New-safety:

```js
function SafeUser(name) {
  if (!new.target) return new SafeUser(name);
  this.name = name;
}
```

### ⚠️ Tricky edge cases

```js
function X() { this.a = 1; return { a: 99 }; }
console.log(new X().a); // 99 (returned object wins)
```

### 🎯 Interview questions

* What does `new.target` mean?
* What happens if constructor returns a primitive?

---

## 4) Prototype Chain

### ✅ What it is

The linked chain used for property/method resolution.

### ✅ Where it applies

* `instanceof`, method lookup, built-in methods
* Polyfills, monkey-patching (careful)

### ❌ Where it does NOT

* Does not “copy methods” into the object
* Not used for private fields (`#x`) in classes

### 🧠 Core principles

* Lookup climbs the chain until found or reaches `null`
* `hasOwnProperty` checks only the object, not prototypes

### 🧩 Mental model

`obj.key` = “search here, else ask prototype, repeat”

### 🧪 Examples

```js
const arr = [];
// arr → Array.prototype → Object.prototype → null
console.log(arr.toString); // from Array.prototype / Object.prototype
```

### ✅ Use cases & Best practices

* Prefer `Object.hasOwn(obj, key)` (modern) or `hasOwnProperty.call`
* Avoid deep inheritance chains

### ⚠️ Tricky edge cases

```js
const dict = Object.create(null);
console.log(dict.hasOwnProperty); // undefined
// Safe check:
console.log(Object.prototype.hasOwnProperty.call(dict, "x"));
```

### 🎯 Interview questions

* How does `instanceof` work internally?
* Why does `[] instanceof Object` return true?

---

## 5) Prototype Edge Cases & V8 De-optimizations

### ✅ What it is

V8 optimizes objects with **hidden classes** and **inline caches**. Prototype mutations and unpredictable object shapes can deoptimize hot code.

### ✅ Where it applies

* Performance-sensitive code
* Large React apps (object shapes, megamorphic callsites)
* Node.js services under load

### ❌ Where it does NOT

* Small scripts where performance doesn’t matter

### 🧠 Core principles

* Consistent property addition order → stable hidden classes
* Changing prototypes dynamically → slows property access
* `delete` on hot objects can degrade performance

### 🧩 Mental model

V8 likes predictable “shapes”. If you keep reshaping objects, it can’t optimize.

### 🧪 Examples

```js
function A(){ this.x = 1; this.y = 2; } // order: x then y
function B(){ this.y = 2; this.x = 1; } // order: y then x (different shape)
```

### ✅ Use cases & Best practices

* Initialize properties in same order for all instances
* Avoid `delete` in hot paths; use `obj.key = null`
* Don’t mutate prototypes after objects are created

### ⚠️ Tricky edge cases

* Too many different object shapes passed into the same function → **megamorphic** → slower

### 🎯 Interview questions

* What are hidden classes?
* Why is `Object.setPrototypeOf` slow?
* Why is `delete` slow?

---

## 6) Currying

### ✅ What it is

Transform `f(a,b,c)` into `f(a)(b)(c)`.

### ✅ Where it applies

* Reusable utilities, functional pipelines
* Partial application patterns

### ❌ Where it does NOT

* When it reduces readability in business logic

### 🧠 Core principles

* Returns functions until enough args are collected
* Often uses closures

### 🧩 Mental model

“Fix some inputs now, supply rest later.”

### 🧪 Examples

```js
const curry = (fn) => function curried(...args) {
  return args.length >= fn.length
    ? fn(...args)
    : (...rest) => curried(...args, ...rest);
};

const add3 = (a,b,c) => a+b+c;
const cadd3 = curry(add3);

console.log(cadd3(1)(2)(3)); // 6
console.log(cadd3(1,2)(3));  // 6
```

### ✅ Use cases & Best practices

* Best for validators/formatters/loggers
* Don’t over-curry everything; keep team readability

### ⚠️ Tricky edge cases

* `fn.length` breaks when default params exist
* Currying with variadic functions needs custom logic

### 🎯 Interview questions

* Currying vs partial application?
* Implement curry supporting mixed calls

---

## 7) Functional Programming (FP) Concepts in JS

### ✅ What it is

A style focusing on:

* **pure functions**
* **immutability**
* **higher-order functions**
* **composition**

### ✅ Where it applies

* React state, reducers
* data transformations
* predictable code and testability

### ❌ Where it does NOT

* When it adds complexity for simple flows
* Heavy FP without team alignment

### 🧠 Core principles

* Pure function: same input → same output, no side effects
* Higher-order: functions that take/return functions
* Composition: combine small functions to build bigger ones

### 🧩 Mental model

Treat data as flowing through a pipeline.

### 🧪 Examples

```js
const pipe = (...fns) => (x) => fns.reduce((v, fn) => fn(v), x);

const inc = x => x + 1;
const double = x => x * 2;

console.log(pipe(inc, double)(3)); // 8
```

### ✅ Use cases & Best practices

* Use `map/filter/reduce` for transformation
* Avoid mutating shared state in async workflows

### ⚠️ Tricky edge cases

* Overusing `reduce` can become unreadable
* Purity breaks with hidden dependencies (globals)

### 🎯 Interview questions

* What makes a function pure?
* Explain function composition with code

---

## 8) `call`, `apply`, `bind`

### ✅ What it is

Ways to control `this` and arguments.

* `call(this, a, b)`
* `apply(this, [a, b])`
* `bind(this, a)` returns new function

### ✅ Where it applies

* Borrowing methods
* Setting `this` in callbacks
* Partial application

### ❌ Where it does NOT

* Arrow functions ignore `bind/call/apply` for `this`

### 🧠 Core principles

* `bind` creates a new function with fixed `this`
* `call/apply` invoke immediately

### 🧩 Mental model

“Choose what `this` points to.”

### 🧪 Examples

```js
function greet(msg) { return `${msg}, ${this.name}`; }
const user = { name: "Surya" };

console.log(greet.call(user, "Hi"));
console.log(greet.apply(user, ["Hi"]));
const g = greet.bind(user, "Hi");
console.log(g());
```

### ✅ Use cases & Best practices

* Use arrow functions in callbacks to avoid losing `this`
* In classes, bind in constructor only if needed

### ⚠️ Tricky edge cases

* `bind` can’t be overridden:

```js
const bound = greet.bind(user);
bound.call({name:"X"}); // still Surya
```

### 🎯 Interview questions

* Implement bind polyfill
* Why arrows don’t have their own `this`?

---

## 9) Debouncing

### ✅ What it is

Delay execution until events stop firing for a period.

### ✅ Where it applies

* search input
* resize
* autosave

### ❌ Where it does NOT

* If you need steady updates (use throttle)

### 🧠 Core principles

Clear previous timer, set a new one.

### 🧩 Mental model

“Run only after the user pauses.”

### 🧪 Example

```js
const debounce = (fn, delay) => {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), delay);
  };
};
```

### ✅ Best practices

* Add `leading/trailing` options if needed
* Cancel on unmount (React)

### ⚠️ Tricky edge cases

* Debounced function needs `.cancel()` sometimes
* `this` binding inside debounce wrapper

### 🎯 Interview questions

* Debounce with `immediate` option?
* Debounce vs throttle differences?

---

## 10) Throttling

### ✅ What it is

Run at most once per interval.

### ✅ Where it applies

* scroll
* mousemove
* analytics (rate limiting)

### ❌ Where it does NOT

* If last update must always happen (need trailing support)

### 🧠 Core principles

Allow execution only after `wait` time.

### 🧩 Mental model

“Limit rate.”

### 🧪 Example

```js
const throttle = (fn, wait) => {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= wait) {
      last = now;
      fn(...args);
    }
  };
};
```

### ✅ Best practices

* Prefer requestAnimationFrame throttle for UI updates

### ⚠️ Tricky edge cases

* Need “trailing call” guarantee → more advanced throttle

### 🎯 Interview questions

* Throttle scroll handler best approach?
* Build throttle with leading/trailing options

---

## 11) `async` vs `defer` (Script Loading)

### ✅ What it is

* `defer`: downloads in parallel, executes after HTML parsing, **in order**
* `async`: downloads in parallel, executes ASAP, **order not guaranteed**

### ✅ Where it applies

* Performance optimization for script loading

### ❌ Where it does NOT

* `async` for dependency scripts (can break order)

### 🧠 Core principles

* `defer` waits for DOM parse
* `async` doesn’t wait, can execute mid-parse

### 🧩 Mental model

* `defer` = “safe parallel load”
* `async` = “fast but unpredictable execution order”

### 🧪 Example

```html
<script defer src="vendor.js"></script>
<script defer src="app.js"></script>
```

### ✅ Best practices

* Use `defer` for app scripts
* Use `async` for analytics/ads that don’t depend on other scripts

### ⚠️ Tricky edge cases

* `defer` scripts run before `DOMContentLoaded`
* `async` may run before DOM exists

### 🎯 Interview questions

* Why does defer preserve order but async doesn’t?
* Which one is better for React app bundle?

---

## 12) Web APIs (Browser Provided APIs)

### ✅ What it is

APIs provided by the browser environment (not the JS engine):

* DOM, timers, fetch/XHR, storage, media, geolocation, etc.

### ✅ Where it applies

* Frontend apps
* Async IO (fetch, timers)

### ❌ Where it does NOT

* JS language itself (ECMAScript) doesn’t include fetch/timers

### 🧠 Core principles

* JS runs on one main thread
* Web APIs do async work, then enqueue callbacks via event loop queues

### 🧩 Mental model

JS = “brain”, Web APIs = “browser assistants”, event loop = “scheduler”.

### 🧪 Example

```js
setTimeout(() => console.log("timer"), 0);
```

`setTimeout` is Web API; callback runs later via event loop.

### ✅ Best practices

* Avoid heavy work on main thread (use Web Workers for CPU tasks)

### ⚠️ Tricky edge cases

* Timers are not precise (clamping, throttling in background tabs)

### 🎯 Interview questions

* Is setTimeout part of JS?
* Why fetch doesn’t block execution?

---

## 13) DOM Lifecycle (Render Pipeline)

### ✅ What it is

How browser turns HTML/CSS into pixels.

### ✅ Where it applies

* Performance (reflow/repaint)
* DOM manipulation impacts

### ❌ Where it does NOT

* Node.js (no DOM pipeline)

### 🧠 Core principles

* HTML → DOM
* CSS → CSSOM
* DOM + CSSOM → Render tree
* Layout → Paint → Composite

### 🧩 Mental model

“DOM changes can trigger layout and repaint.”

### 🧪 Example (layout thrash risk)

```js
el.style.width = "100px";
const h = el.offsetHeight; // forces layout
el.style.height = (h + 10) + "px";
```

### ✅ Best practices

* Batch DOM reads then writes
* Use CSS transforms for animations (compositor-friendly)

### ⚠️ Tricky edge cases

* Measuring layout (`offsetHeight`) forces sync layout

### 🎯 Interview questions

* Difference between reflow vs repaint?
* Why transforms are faster than top/left animations?

---

## 14) DOM Events + Bubbling/Capturing/Delegation

### ✅ What it is

Event propagation phases:

1. Capture
2. Target
3. Bubble

### ✅ Where it applies

* UI handling
* Delegation patterns

### ❌ Where it does NOT

* Some events don’t bubble (e.g., `focus`/`blur`), use `focusin/focusout`

### 🧠 Core principles

* Bubbling: child → parent
* Capturing: parent → child
* Delegation uses bubbling to handle many items with one listener

### 🧩 Mental model

Event “travels” through ancestors.

### 🧪 Examples

```js
parent.addEventListener("click", () => console.log("bubble"));
parent.addEventListener("click", () => console.log("capture"), true);
```

Delegation:

```js
list.addEventListener("click", (e) => {
  const li = e.target.closest("li");
  if (!li) return;
  console.log(li.dataset.id);
});
```

### ✅ Best practices

* Delegate for large lists
* Use `{ passive: true }` for scroll/touch performance where safe

### ⚠️ Tricky edge cases

* `stopPropagation` vs `stopImmediatePropagation`
* Shadow DOM retargeting (advanced)

### 🎯 Interview questions

* Why delegation improves performance?
* Capturing use cases?

---

## 15) BOM vs DOM

### ✅ What it is

* DOM: `document`, elements
* BOM: `window`, `location`, `history`, `navigator`

### ✅ Where it applies

* Routing without reload (`history.pushState`)
* Reading URL params (`location.search`)
* Browser capabilities (`navigator`)

### ❌ Where it does NOT

* Node.js environment doesn’t have BOM

### 🧪 Example

```js
history.pushState({}, "", "/new-page");
```

### 🎯 Interview questions

* What’s difference between BOM and DOM?
* How SPA routing works using history API?

---

## 16) Network / Communication (Fetch)

### ✅ What it is

Browser API to make network requests (Promise-based).

### ✅ Where it applies

* REST/GraphQL calls
* Streaming responses (advanced)

### ❌ Where it does NOT

* Doesn’t reject on HTTP 404/500 (only rejects on network failure)

### 🧠 Core principles

* Fetch resolves on HTTP response; check `res.ok`
* CORS enforced by browser

### 🧪 Example

```js
const res = await fetch("/api");
if (!res.ok) throw new Error("HTTP error");
const data = await res.json();
```

### 🎯 Interview questions

* Why fetch doesn’t throw on 404?
* Fetch vs XHR?

---

## 17) CORS + Preflight (OPTIONS)

### ✅ What it is

CORS is a browser security policy controlling cross-origin reads.

### ✅ Where it applies

* Frontend calling APIs on different origin
* Cookies/credentials across origins

### ❌ Where it does NOT

* Server-to-server requests (CORS is browser-enforced)

### 🧠 Core principles

* “Simple requests” may skip preflight
* “Non-simple” requests trigger OPTIONS preflight

Triggers typically include:

* method not GET/HEAD/POST
* custom headers
* `Content-Type: application/json` (commonly causes preflight)

### 🧩 Mental model

Browser asks server permission first (OPTIONS), then sends real request if allowed.

### 🧪 Example: required headers

Server must respond with:

* `Access-Control-Allow-Origin`
* `Access-Control-Allow-Methods`
* `Access-Control-Allow-Headers`

### ✅ Best practices

* Don’t use `*` with credentials
* Use reverse proxy in dev for same-origin
* Cache preflight with `Access-Control-Max-Age` (careful)

### ⚠️ Tricky edge cases

* `credentials: "include"` requires:

  * `Access-Control-Allow-Credentials: true`
  * and specific origin (not `*`)

### 🎯 Interview questions

* Why browser sends OPTIONS even if you didn’t?
* Why Postman works but browser fails?

---

## 18) Storage (Browser)

### ✅ What it is

Client-side persistence:

* cookies
* localStorage
* sessionStorage
* IndexedDB

### ✅ Where it applies

* Auth state (careful)
* Caching, preferences
* Offline apps (IndexedDB)

### ❌ Where it does NOT

* localStorage/sessionStorage not available in server rendering (SSR)

### 🧠 Core principles

* cookies: sent with requests; secure flags matter
* localStorage: synchronous, persistent, per-origin
* sessionStorage: per-tab session
* IndexedDB: async, large structured storage

### ✅ Best practices

* Prefer HttpOnly secure cookies for sensitive tokens (common approach)
* Use `SameSite`, `Secure`, `HttpOnly` where possible

### ⚠️ Tricky edge cases

* localStorage is synchronous → can block main thread if abused
* iOS/Safari storage quirks in private mode (practical pain)

### 🎯 Interview questions

* cookie vs localStorage differences?
* Why HttpOnly cookies are safer vs XSS?

---

## 19) Browser Environment + Execution Context Lifecycle

### ✅ What it is

How JS runs:

* global execution context created
* function execution contexts created per call
* each has scope bindings + `this`

### ✅ Where it applies

* Hoisting, TDZ, scope chain
* Debugging closures, `this`

### ❌ Where it does NOT

* Doesn’t mean multi-threaded execution (still single-thread for JS)

### 🧠 Core principles

* Lexical environment holds identifiers
* Scope chain resolves variables
* `this` depends on call-site (except arrows)

### 🧩 Mental model

Each function call gets its own “execution frame” with local variables.

### 🧪 Example

```js
let x = 1;
function f() {
  let y = 2;
  return x + y;
}
```

### 🎯 Interview questions

* What is scope chain?
* How `this` is decided?

---

## 20) Why TDZ exists (Temporal Dead Zone)

### ✅ What it is

`let/const` exist in scope but are **uninitialized** until declaration is executed → accessing before that throws `ReferenceError`.

### ✅ Where it applies

* `let`, `const`, `class` declarations

### ❌ Where it does NOT

* `var` (hoisted + initialized to `undefined`)

### 🧠 Core principles

* Prevents “use before declaration” bugs
* Enables block scoping semantics cleanly

### 🧩 Mental model

Name exists, but “not ready” until line executes.

### 🧪 Example

```js
console.log(a); // ReferenceError (TDZ)
let a = 10;
```

### ⚠️ Tricky edge case

```js
let x = 1;
(function(){
  console.log(x); // ReferenceError (inner x in TDZ)
  let x = 2;
})();
```

### 🎯 Interview questions

* Why `var` prints undefined but `let` throws?
* When does TDZ start/end?

---

## 21) JS Engine (V8)

### ✅ What it is

V8 is a JS engine that parses, compiles (JIT), optimizes code, manages memory (GC).

### ✅ Where it applies

* Node.js and Chromium-based browsers
* Performance & memory tuning

### ❌ Where it does NOT

* Not universal (Safari uses JavaScriptCore, Firefox uses SpiderMonkey)

### 🧠 Core principles

* JIT: interpret/compile → optimize hot code
* Hidden classes & inline caches optimize property access
* GC reclaims unused memory

### 🧩 Mental model

Code becomes faster after it runs a lot (“warming up”).

### 🎯 Interview questions

* What is JIT?
* What are hidden classes / inline caches?

---

## 22) Browser Internals: Event Loop + Task Queues (Deep)

### ✅ What it is

Scheduler that decides when callbacks run.

### ✅ Where it applies

* async behavior, promises, timers, UI events

### ❌ Where it does NOT

* Doesn’t mean parallelism (JS still single-threaded)

### 🧠 Core principles

* Macrotask queue: timers, UI events, message events
* Microtask queue: promise callbacks, `queueMicrotask`, MutationObserver
* After each macrotask: **drain all microtasks**, then render

### 🧩 Mental model

Macrotask = big steps; Microtask = urgent small steps executed before next big step.

### 🧪 Example

```js
console.log("A");
setTimeout(()=>console.log("T"), 0);
Promise.resolve().then(()=>console.log("P"));
console.log("B");
// A B P T
```

### ⚠️ Tricky edge cases

* Infinite microtasks can freeze UI
* `await` resumes via microtasks

### 🎯 Interview questions

* Predict output puzzles
* Why promise callbacks run before setTimeout(0)?

---

## 23) JS Engine & Performance (Hidden classes, GC, Memory leaks)

### ✅ What it is

Real-world performance topics:

* hidden classes shape stability
* garbage collection behavior
* leaks in SPAs

### ✅ Where it applies

* high-scale frontend and Node services
* diagnosing memory spikes and CPU issues

### 🧠 Core principles

* Stable object shapes → faster
* Too many shapes at same callsite → megamorphic (slower)
* GC cost grows with long-lived objects

### ✅ Common memory leaks

* Detached DOM nodes still referenced
* Event listeners not removed
* Intervals not cleared
* Growing caches/maps without eviction

### 🎯 Interview questions

* Why closures can cause memory leaks?
* How to detect leaks (heap snapshots conceptually)?

---

## 24) Array & Object Methods (Must know)

### ✅ Arrays

* Transform: `map`, `flatMap`
* Filter: `filter`
* Reduce: `reduce`
* Search: `find`, `findIndex`, `some`, `every`, `includes`
* Slice vs splice (non-mutating vs mutating)
* Sort pitfalls

```js
[1, 2, 10].sort(); // [1,10,2]
[1, 2, 10].sort((a,b)=>a-b); // [1,2,10]
```

### ✅ Objects

* `Object.keys/values/entries`
* `Object.assign` (shallow)
* `Object.freeze` (shallow)
* `Object.create`, `Object.getPrototypeOf`

### 🎯 Interview questions

* Group by using reduce
* `for..in` vs `for..of`

---

## 25) Immutability in JS

### ✅ What it is

Don’t mutate existing objects/arrays; create new versions.

### ✅ Where it applies

* React state updates
* Redux reducers
* concurrency-safe patterns

### ❌ Where it does NOT

* If performance requires mutation and it’s isolated/controlled (careful)

### 🧠 Core principles

* “New reference” signals change
* Structural sharing (copy only changed parts)

### 🧪 Examples

```js
const next = { ...prev, a: 2 };
const nextArr = prevArr.map(x => x.id === id ? {...x, done:true} : x);
```

### ⚠️ Tricky edge cases

* Spread is shallow, nested objects still shared

### 🎯 Interview questions

* Why immutability helps React?
* Shallow vs deep immutability?

---

## 26) Shallow Copy vs Deep Copy

### ✅ Shallow copy

Copies top-level only.

* `{...obj}`, `Object.assign`, `arr.slice()`

### ✅ Deep copy

* `structuredClone(obj)` (modern best)
* JSON clone (lossy)
* custom deep clone

```js
const deep = structuredClone(obj);
```

### ⚠️ Tricky edge cases

* JSON drops Date, Map, Set, undefined, functions
* Cyclic objects need special handling

### 🎯 Interview questions

* Why JSON deep clone is dangerous?
* How to clone cyclic object?

---

## 27) Error Handling

### ✅ What it is

Handling exceptions and failures safely.

### ✅ Where it applies

* API calls, parsing, business logic
* async flows (await + try/catch)

### 🧠 Core principles

* `try/catch` catches sync errors and awaited async errors
* Promise chains need `.catch` unless awaited

### 🧪 Examples

```js
try {
  await doWork();
} catch (e) {
  // handle
} finally {
  // cleanup
}
```

### ✅ Best practices

* Don’t swallow errors
* Add context and error codes
* Prefer typed/custom errors in apps

### 🎯 Interview questions

* Difference between thrown error and rejected promise?
* What is `unhandledrejection`?

---

## 28) Overloading & Overriding (JS Reality)

### ✅ Overriding

Supported via prototype chain / class inheritance.

### ❌ Overloading

No real signature overloading in JS. Simulate using:

* optional params
* rest params
* checking `arguments.length`

```js
function sum(...nums) {
  return nums.reduce((a,b)=>a+b,0);
}
```

### 🎯 Interview questions

* Why JS doesn’t support overloads like Java?
* Design API without overload confusion

---

## 29) Reference vs Value

### ✅ Value (primitives)

Copied by value:
`number, string, boolean, null, undefined, symbol, bigint`

### ✅ Reference (objects/functions/arrays)

Copied by reference (same object).

```js
const a = { x: 1 };
const b = a;
b.x = 2;
console.log(a.x); // 2
```

### 🎯 Interview questions

* Why `const obj` can mutate?
* How to truly prevent mutation?

---

## 30) OOP in JS

### ✅ What it is

JS supports OOP with:

* prototypes
* classes (syntax sugar)
* encapsulation patterns (closures, private fields)

### ✅ Where it applies

* domain modeling
* frameworks/classes

### ❌ Where it does NOT

* Not required for everything; FP/composition often better

### 🎯 Interview questions

* Class vs prototype-based inheritance?
* Private fields `#x` vs closure privacy?

---

## 31) Compile Time vs Run Time (JS Context)

### ✅ What it is

JS is JIT compiled:

* parsing and compilation happen before and during execution

### ✅ Compile-time-ish

* Syntax errors (parser catches before running)

### ✅ Runtime

* TypeError, ReferenceError etc happen when executing

### 🎯 Interview questions

* What is JIT compilation?
* Why TypeScript types don’t exist at runtime?

---

## 32) Polyfills & Transpilation

### ✅ What it is

* Transpile: convert modern syntax → older JS (Babel/SWC/TS)
* Polyfill: implement missing APIs at runtime (core-js, etc.)

### 🧠 Mental model

* Syntax compatibility → transpile
* API compatibility → polyfill

### ✅ Best practices

* Use `browserslist` targets
* Prefer usage-based polyfills (only what you use)
* Avoid global pollution in libraries

### 🎯 Interview questions

* Transpilation vs polyfill?
* Why code works in Chrome but not Safari?

---

## 33) JS Tooling Ecosystem (Webpack, Babel, ESLint, Prod polyfills)

### ✅ What it is

Build + quality pipeline:

* bundlers: Webpack, Vite, Rollup, esbuild
* transpilers: Babel, SWC
* lint: ESLint
* format: Prettier
* test: Jest/Vitest

### ✅ Where it applies

* production builds, performance, compatibility, CI/CD

### 🎯 Interview questions

* Why bundling is needed?
* Why Babel is still used with modern browsers?

---

# 🔥 Practice: “Tricky Output” Interview Set

1.

```js
console.log(1);
setTimeout(()=>console.log(2), 0);
Promise.resolve().then(()=>console.log(3));
console.log(4);
```

2.

```js
const a = {x:1};
const b = Object.create(a);
console.log(b.x);
b.x = 2;
delete b.x;
console.log(b.x);
```

3.

```js
const obj = { x: 10, f() { return this.x; } };
const g = obj.f;
console.log(g());
```

4.

```js
let x = 1;
(function(){
  console.log(x);
  let x = 2;
})();
```

---


