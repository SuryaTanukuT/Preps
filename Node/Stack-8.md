

https://blog.greenroots.info/javascript-promise-chain-the-art-of-handling-promises

---

## 1) Promise chaining rules (this is the “art”)

### ✅ Rule A — `.then()` always returns a **new Promise**

So a chain is literally a pipeline of Promises.

### ✅ Rule B — What you `return` decides the next step

Inside a `then/catch` handler:

1. **Return a value** → next promise **fulfills** with that value
2. **Throw an error** → next promise **rejects** with that error
3. **Return a promise** → chain **waits** for it (promise “flattens/unwraps”)
4. **Return nothing** → you returned `undefined` (next gets `undefined`)

```js
Promise.resolve(10)
  .then(x => x + 1)              // returns value -> next gets 11
  .then(x => Promise.resolve(x)) // returns promise -> waits/unwrapped
  .then(x => { throw new Error("boom"); }) // throws -> rejection
  .catch(err => "recovered")     // returns value -> fulfilled
  .then(v => console.log(v));    // "recovered"
```

This “flattening” behavior is part of the spec and is why promise chains compose cleanly. ([nodejs.org][1])

---

## 2) `.catch()` placement strategy (senior-level)

### Pattern 1 — **One catch at the end** (most common)

Good when you want “fail the whole pipeline”.

```js
doA()
  .then(doB)
  .then(doC)
  .catch(handleError);
```

### Pattern 2 — **Local catch to recover and continue**

Good when a step is optional (cache miss, best-effort call).

```js
getFromCache()
  .catch(() => null)        // recover locally
  .then(val => val ?? getFromDB());
```

### Pattern 3 — **Rethrow after logging** (don’t swallow)

```js
doWork()
  .catch(err => {
    log(err);
    throw err;              // keep failure semantics
  });
```

---

## 3) `finally()` gotchas (common interview trap)

### ✅ `finally()` gets **no result argument**

It’s for cleanup, not transformation.

### ✅ `finally()` does **not change** the resolved value…

…unless you **throw** or **return a rejecting promise** inside it.

```js
Promise.resolve("OK")
  .finally(() => { throw new Error("cleanup failed"); })
  .then(console.log)
  .catch(console.error); // cleanup failed overrides OK
```

MDN documents `finally()` semantics clearly. ([MDN Web Docs][2])

---

## 4) Promise combinators — exact behavior (no hand-wavy)

### `Promise.all([...])`

* ✅ resolves when **all succeed**
* ❌ rejects on the **first rejection** (“fail fast”)
* Important: other operations may still be running; you just stop waiting.

### `Promise.allSettled([...])`

* ✅ waits for **all to finish** (success/fail)
* Returns `{status, value|reason}` results.

### `Promise.race([...])`

* Settles with the **first settled** promise (fulfill OR reject)

### `Promise.any([...])`

* Resolves with the **first fulfilled**
* Rejects only if **all reject** → gives **AggregateError** ([MDN Web Docs][2])

**Interview one-liner:**

> `all` is “everyone must win”, `allSettled` is “everyone reports back”, `race` is “first response (good or bad)”, `any` is “first success”. ([MDN Web Docs][2])

---

## 5) Node.js reality check: unhandled rejections (production critical)

In Node.js, an unhandled promise rejection can crash your process depending on version/config.

* Node **changed default behavior in v15**: if you don’t handle `unhandledRejection`, the default mode moved to **throw** (becomes an uncaught exception). ([nodejs.org][3])

### Practical best practice

* Always end chains with `.catch(...)`
* For safety in services, add a process-level handler (and alert)

```js
process.on("unhandledRejection", (reason) => {
  // log + metrics + alert
});
process.on("uncaughtException", (err) => {
  // log + attempt graceful shutdown
});
```

(Still: prefer fixing the code path, not relying on global handlers.)

---

## 6) Microtasks + promise chains: “why promises feel faster”

Your earlier mental model is right: `.then/.catch/.finally` handlers run as **microtasks**. Long microtask chains can **starve** timers/I/O if abused. ([MDN Web Docs][2])

**Rule of thumb (server-side):**

* If you’re doing heavy loops with promise microtasks, occasionally yield using `setImmediate()`.

---

## 7) Timeouts + cancellation (Promises don’t cancel by themselves)

Native Promises don’t provide cancellation. The correct modern approach is **AbortController** for APIs that support it (fetch, some drivers, your own code). MDN covers the general behavior of promises; cancellation is typically done via host APIs. ([MDN Web Docs][2])

**Timeout wrapper (generic):**

```js
function withTimeout(p, ms) {
  return Promise.race([
    p,
    new Promise((_, rej) => setTimeout(() => rej(new Error("Timeout")), ms))
  ]);
}
```

**Better when supported:** abort the underlying work (so you don’t waste resources).

---

## 8) `Promise.withResolvers()` (modern feature — when to use)

`Promise.withResolvers()` creates a promise plus external `resolve/reject`. It’s basically the clean standard replacement for the old “deferred” pattern.

```js
const { promise, resolve, reject } = Promise.withResolvers();
// later:
resolve("done");
```

### Use it for:

* Bridging event/callback style code into a promise **once**
* Building primitives like “wait until event fires”

### Don’t overuse:

Exposing `resolve/reject` widely can become a footgun (harder reasoning, memory leaks, dangling promises).

---

## 9) Production patterns you should mention in interviews

### A) Concurrency limiting (avoid `Promise.all(users.map(...))` on huge lists)

Use batching/semaphore style limits to protect DB pools and memory.

**Conceptual pattern:**

* “max N in-flight promises”
* queue the rest

### B) Retries (only for idempotent/transient failures)

* exponential backoff + jitter
* cap attempts (2–3)
* add per-call timeout

### C) Always clean up resources

* `finally()` for `close()` / `release()` (but remember the override rules)

---

## 10) Senior “one-liner” answer (copy-paste)

> “A Promise represents the eventual completion of async work and can be composed via chaining. In a chain, returning a value fulfills the next step, throwing rejects it, and returning a promise makes the chain wait (flatten). I use `all` for parallel success-required work, `allSettled` for best-effort fanout, `race` for deadlines, and `any` for first-success fallbacks. In Node, I treat unhandled rejections as production-critical (Node v15+ defaults can throw), so every chain is terminated with a catch and we have process-level handlers for visibility.” ([nodejs.org][3])

---


