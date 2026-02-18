
---

## 0) The few corrections that matter

### ✅ `await` does **not** block the event loop

It **pauses only the current async function**, and schedules the continuation to run later (as a Promise continuation).

### ✅ EventEmitter listeners are **sync by default**

When you do `emitter.emit()`, Node runs listeners **immediately, in order**, on the same call stack (unless *you* explicitly defer with `setImmediate`, `queueMicrotask`, etc.). ([nodejs.org][1])

### ✅ `events.once(emitter, name)` is a *Promise bridge*

It returns a Promise that resolves when the event fires (with an **array of args**), and rejects if `'error'` fires while waiting. ([nodejs.org][1])

### ✅ The “10 listeners” thing is a **warning**, not a hard cap

Default max is **10**, exceeding it emits a **possible memory leak warning**; you can change it with `setMaxListeners` or `events.defaultMaxListeners`. ([nodejs.org][1])

---

## 1) Promises: the *real* mental model interviewers test

### Promise chain rule (most common bug)

If you don’t **return** a Promise/value from `.then()`, the next `.then()` receives `undefined`.

```js
doA()
  .then(() => doB())     // ✅ return the promise
  .then(resultB => doC(resultB))
  .catch(handle);
```

**Error propagation rule:** Throwing inside `.then()` behaves like rejection and is caught by `.catch()`. (That’s why centralized error handling works.) ([MDN Web Docs][2])

### Promise combinators — crisp truth

* **`Promise.all`**: waits for all, **fails fast** on first rejection. ([MDN Web Docs][3])
* **`Promise.allSettled`**: waits for all, returns per-result status. ([MDN Web Docs][4])
* **`Promise.race`**: first one to **settle** (resolve or reject) wins. ([MDN Web Docs][5])
* **`Promise.any`**: first one to **fulfill** wins; rejects only if all reject. ([MDN Web Docs][4])

### `Promise.withResolvers()` (modern)

Useful when you need “external resolve/reject” (like bridging events → promise), but use it carefully to avoid dangling promises. ([MDN Web Docs][3])

---

## 2) Async/Await: what it *really compiles to*

From spec/userland perspective:

* `async function` **always returns a Promise**
* `return x` becomes a fulfilled Promise
* `throw e` becomes a rejected Promise

```js
async function f() { return 10; }   // Promise<10>
async function g() { throw new Error("x"); } // Promise rejection
```

### Accidental sequential (performance bug)

```js
const a = await taskA();
const b = await taskB(); // ❌ runs after A, even if independent
```

Fix:

```js
const [a, b] = await Promise.all([taskA(), taskB()]);
```

### Production “hybrid” flow (best pattern)

```js
const user = await getUser(); // dependent (waterfall start)

const [orders, wallet] = await Promise.all([ // independent (parallel branch)
  getOrders(user.id),
  getWallet(user.id),
]);

return buildDashboard(user, orders, wallet);
```

---

## 3) Cancellation: the practical truth

Native Promises don’t cancel by themselves. In real systems:

* **AbortController** (HTTP/fetch, some libs)
* **timeouts** (`race` with a timer)
* **concurrency control** (limit inflight work)

(Your section here is correct—just keep that “Promises don’t cancel” line very crisp.)

---

## 4) Generators: where they shine (and where they don’t)

### Generator = “pausable function that implements the iterator protocol”

Calling a generator returns an iterator; `.next()` drives execution and yields `{ value, done }`. ([MDN Web Docs][2])

```js
function* range(n) {
  for (let i = 0; i < n; i++) yield i;
}
```

### Generator vs async/await (interview reality)

* Generators are **great** for: lazy iteration, state machines, building custom iterables.
* For business async flows, interviewers typically expect **async/await**, not generator-runners (like `co`) — unless you’re discussing legacy codebases.

### Async generators (modern, very useful)

Async generators are excellent for **streaming async data** and `for await...of`.

```js
async function* stream() {
  yield await fetchChunk1();
  yield await fetchChunk2();
}
for await (const chunk of stream()) { /* ... */ }
```

---

## 5) EventEmitter: the “gotchas” that cause real outages

### 5.1 Listeners run immediately (sync), and `emit` does not await them

If a listener is `async`, it returns a Promise, but `emit()` doesn’t wait.

```js
emitter.on("x", async () => {
  await slow();
  throw new Error("boom");
});

emitter.emit("x"); // doesn't await
```

So you must handle errors inside the listener (or use patterns like `captureRejections` in Node, depending on design). ([nodejs.org][1])

### 5.2 The `'error'` event is special

If an `'error'` event is emitted and there’s **no error listener**, the process will crash (the classic “Unhandled 'error' event”). ([nodejs.org][1])

Minimum safe pattern:

```js
emitter.on("error", (err) => logger.error(err));
```

### 5.3 `events.once()` is the clean bridge (and supports AbortSignal)

```js
import { once } from "node:events";
import { AbortController } from "node-abort-controller"; // if needed in your env

const [data] = await once(emitter, "data"); // resolves with args array
```

It resolves on the event, rejects on `'error'` while waiting, and supports cancellation via `signal`. ([nodejs.org][1])

### 5.4 Memory leak warnings (10 listeners)

Default max listeners is 10; exceeding it warns “possible EventEmitter memory leak”, and you can adjust it. ([nodejs.org][1])

---

## 6) Senior interview “one-liners” (memorize-ready)

* **Promises:** “A Promise is a one-time future value (or error) that can be chained; errors propagate through the chain to the nearest catch.” ([MDN Web Docs][2])
* **Async/await:** “Async/await is syntax over promises—`await` yields control, it doesn’t block the event loop; `async` always returns a Promise.”
* **Generators:** “Generators are pausable functions producing lazy sequences via the iterator protocol; async generators extend that to streaming async data.” ([MDN Web Docs][2])
* **EventEmitter:** “Emit is synchronous; listeners run immediately. The `'error'` event is special—if unhandled, the process crashes. Use `events.once()` to await events safely.” ([nodejs.org][1])

---


---

# 1) Event Loop + Microtasks / Macrotasks (Browser + Node)

## Mental model

JS executes on **one call stack**. Async work completes elsewhere (OS/libuv/Web APIs) and schedules callbacks back onto JS via queues.

### Two priority lanes

* **Microtasks (higher priority):** `Promise.then/catch/finally`, `queueMicrotask`
* **Macrotasks (task queue):** `setTimeout`, `setInterval`, UI events, I/O callbacks (conceptually)

### Key rule

✅ **Microtasks always drain fully before the next macrotask runs**.

---

## Node.js event loop (libuv) phases (important for interviews)

Order (simplified):

1. **timers** → `setTimeout`, `setInterval`
2. **pending callbacks**
3. **poll** → I/O callbacks (most socket/file callbacks land here)
4. **check** → `setImmediate`
5. **close callbacks**

### Where microtasks fit in Node

After executing a callback in a phase, Node drains:

* **`process.nextTick` queue** (highest priority; Node-specific)
* then **Promise microtasks**

**Trap:** `process.nextTick` can starve the loop if abused.

---

## Ordering demo (classic)

```js
console.log("A");

setTimeout(() => console.log("timeout"), 0);
setImmediate(() => console.log("immediate"));

Promise.resolve().then(() => console.log("promise"));
process.nextTick(() => console.log("nextTick"));

console.log("B");
```

Typical output:

```
A
B
nextTick
promise
(timeout vs immediate order can vary depending on context)
```

### `setTimeout(0)` vs `setImmediate()`

* In **I/O callbacks**, `setImmediate` often fires before `setTimeout(0)`.
* Outside that context, ordering can vary.

---

## Interview one-liner

> “JS runs on one stack. Async completions queue callbacks. Microtasks (Promises) run before macrotasks, and in Node, `process.nextTick` runs even before Promise microtasks.”

---

# 2) Promise Semantics + Chaining + Combinators

## What a Promise is (the real definition)

A Promise is a **single-assignment container** for:

* **fulfilled(value)** or **rejected(reason)**
* once settled, it’s **immutable**

---

## Chaining rules (the ones interviewers test)

### Rule 1: `.then` returns a new Promise

```js
p.then(fn) // returns a new promise
```

### Rule 2: Return matters

```js
doA()
  .then(() => doB()) // ✅ return promise
  .then(b => doC(b))
```

If you forget `return`, you accidentally pass `undefined` to the next step.

### Rule 3: Throw == reject

```js
doA()
  .then(() => { throw new Error("x"); })
  .catch(err => console.error(err)); // catches it
```

### Rule 4: `.catch` is just `.then(undefined, onRejected)`

(Useful to explain error propagation.)

---

## Combinators (high-signal summary)

### `Promise.all([...])`

* ✅ parallel
* ❌ **fails fast** on first rejection
* returns values in **input order** (not completion order)

### `Promise.allSettled([...])`

* ✅ gives per-result `{status, value|reason}`
* best for **partial failure** pipelines

### `Promise.race([...])`

* first to **settle** wins (resolve OR reject)

### `Promise.any([...])`

* first to **fulfill** wins
* rejects only if **all reject** (AggregateError)

---

## Unhandled rejection (production must-know)

If a Promise rejects and nobody handles it, it can become an **unhandled rejection**.
In Node, handle globally:

```js
process.on("unhandledRejection", (reason, p) => {
  // log + alert + decide policy
});
process.on("uncaughtException", (err) => {
  // log + crash gracefully (usually)
});
```

> Senior stance: log, fail fast *with graceful shutdown* if correctness is at risk.

---

# 3) Async/Await Patterns (sequential/parallel/hybrid + timeouts + retries + limits)

## What `await` really does

* pauses the **current async function**
* **does not block** the event loop
* resumes later via microtask scheduling

---

## Pattern A: Sequential (waterfall)

Use when steps depend on previous output.

```js
const user = await getUser();
const orders = await getOrders(user.id);
const invoice = await generateInvoice(orders);
```

**Pros:** simple, readable
**Cons:** slow if steps were independent

---

## Pattern B: Parallel (scatter-gather)

Use when independent.

```js
const [profile, orders, wallet] = await Promise.all([
  getProfile(id),
  getOrders(id),
  getWallet(id),
]);
```

**Pros:** lowest latency
**Cons:** resource spikes; fails fast

---

## Pattern C: Hybrid (senior default)

Waterfall until you have the key, then fan out.

```js
const user = await getUser(id);

const [orders, wallet] = await Promise.all([
  getOrders(user.id),
  getWallet(user.id),
]);

return buildDashboard(user, orders, wallet);
```

---

## Timeouts (correct)

Use `AbortController` if supported; else `race`.

### `race` timeout wrapper

```js
function withTimeout(p, ms) {
  return Promise.race([
    p,
    new Promise((_, rej) => setTimeout(() => rej(new Error("Timeout")), ms)),
  ]);
}
```

### Better: AbortController (when possible)

```js
const ac = new AbortController();
const t = setTimeout(() => ac.abort(), 2000);

try {
  const res = await fetch(url, { signal: ac.signal });
} finally {
  clearTimeout(t);
}
```

---

## Retries (production-grade: exponential backoff + jitter)

```js
const sleep = (ms) => new Promise(r => setTimeout(r, ms));

async function retry(fn, { retries = 3, base = 100, factor = 2 } = {}) {
  let err;
  for (let i = 0; i <= retries; i++) {
    try { return await fn(); }
    catch (e) {
      err = e;
      const backoff = base * (factor ** i);
      const jitter = Math.floor(Math.random() * backoff * 0.2);
      await sleep(backoff + jitter);
    }
  }
  throw err;
}
```

**Rule:** Retry **idempotent** operations (or use idempotency keys).

---

## Concurrency limits (avoid “Promise.all of 50k” meltdown)

### Simple semaphore (no libs)

```js
function pLimit(max) {
  let active = 0;
  const queue = [];

  const next = () => {
    if (active >= max || queue.length === 0) return;
    active++;
    const { fn, resolve, reject } = queue.shift();
    fn().then(resolve, reject).finally(() => { active--; next(); });
  };

  return (fn) => new Promise((resolve, reject) => {
    queue.push({ fn, resolve, reject });
    next();
  });
}

const limit = pLimit(5);
await Promise.all(users.map(u => limit(() => sendEmail(u))));
```

---

# 4) Cancellation + Backpressure

## Cancellation

Native Promises don’t cancel themselves. Use:

* **AbortController** (preferred when supported)
* **cancellation tokens** (library-defined)
* **timeouts**
* **stop producing work** (most important)

### Cancel pattern: stop spawning new work

If a downstream is failing, don’t keep enqueuing more promises.

---

## Backpressure (real meaning)

Backpressure = “consumer can’t keep up; slow down producer”.

### Where it matters most in Node

* **Streams** (built-in backpressure)
* **message queues / Kafka consumers**
* **batch jobs** (limits)

### Streams example (backpressure-friendly)

```js
readable.pipe(transform).pipe(writable);
```

`pipe()` automatically pauses/resumes based on `writable` capacity.

### Async-iterator pattern (clean + backpressure-friendly)

```js
for await (const chunk of readableStream) {
  await processChunk(chunk); // your await naturally controls pace
}
```

**Senior line:** “Backpressure is not retries — it’s controlling ingestion rate.”

---

# 5) Generators + Async Generators (lazy + streaming)

## Generators

A generator function (`function*`) returns an **iterator** and can `yield` multiple values lazily.

```js
function* range(n) {
  for (let i = 0; i < n; i++) yield i;
}

for (const x of range(3)) console.log(x);
```

### When generators are great

* huge sequences without allocating arrays
* state machines
* custom iteration protocols

---

## Async generators (modern + powerful)

Async generator yields values over time and is consumed with `for await...of`.

```js
async function* streamChunks() {
  while (true) {
    const chunk = await getNextChunk();
    if (!chunk) return;
    yield chunk;
  }
}

for await (const c of streamChunks()) {
  await handle(c);
}
```

### Why async generators are “senior”

They combine:

* streaming
* backpressure (consumer pace controls producer)
* clean syntax

---

# 6) EventEmitter Production Gotchas (error event + leak warnings + once bridge)

## Gotcha 1: `emit()` is synchronous

Listeners run immediately on the same call stack.

```js
emitter.on("evt", () => console.log("L1"));
emitter.emit("evt"); // runs now
```

## Gotcha 2: async listeners aren’t awaited

```js
emitter.on("evt", async () => {
  await slow();
  throw new Error("boom");
});

emitter.emit("evt"); // doesn't await; error may become unhandled
```

**Fix:** handle errors inside listeners or design a promise-based event API.

---

## Gotcha 3: `'error'` event is special

If an `'error'` event is emitted with no listener → process can crash.

```js
emitter.on("error", (err) => logger.error(err));
```

---

## Gotcha 4: memory leak warnings (10 listeners default)

Node warns when too many listeners are attached to the same event:

* It’s a **warning**, not a limit.
* Better fix: remove listeners correctly (`off/removeListener`) rather than just raising the cap.

---

## Bridge events → Promise (clean pattern)

Use `once` to await an event.

```js
import { once } from "node:events";

const [msg] = await once(emitter, "message");
```

This is safer than manual callback wrapping.

---

# 7) Async Flow Patterns (the “system design” view)

These are patterns you name in senior interviews when describing real pipelines:

## A) Serial / Strict sequence

* one at a time (often for rate limits / ordering)
* example: ledger posting steps

## B) Waterfall (dependent chain)

* each step uses previous output
* example: auth → fetch profile → authorize → fetch data

## C) Parallel / Fan-out Fan-in (scatter-gather)

* kick off independent calls, aggregate result
* example: dashboard aggregation service

## D) Race / Hedged requests

* race cache vs DB, or primary vs replica
* “hedging” = fire a backup if slow tail latency

## E) Batch + throttle

* process N items with concurrency K
* avoids overload + controls memory

## F) Pipeline / staged processing

* transform through stages
* can be implemented with streams or queues
* example: ingest → validate → enrich → persist → publish event

## G) Producer–Consumer (queue-based)

* producer emits jobs, consumers pull with concurrency
* critical for backpressure + smoothing spikes

## H) Saga / Compensating actions (distributed workflows)

* step-by-step with rollback actions
* example: payment → inventory reserve → shipment → email

### Senior summary line

> “In production I default to hybrid flows: waterfall for dependencies, parallel for independent calls, concurrency limits for stability, backpressure for streaming/queues, and timeouts+retries with idempotency for resilience.”

---

