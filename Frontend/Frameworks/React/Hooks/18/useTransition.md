## `useTransition` in React (React 18) — interview-ready

### What it is

`useTransition` lets you mark some state updates as **non-urgent (low priority)** so the UI stays responsive for urgent updates like typing, clicking, and input caret movement.

It returns:

* `isPending` (boolean: transition is in progress)
* `startTransition` (function to wrap low-priority updates)

```jsx
const [isPending, startTransition] = useTransition();
```

Interview one-liner:

> “`useTransition` keeps the UI responsive by deprioritizing expensive state updates.”

---

## Why it’s used

Problem: typing in a search box triggers a heavy list filter/render → typing feels laggy.

Solution:

* Input update = urgent
* List update = transition (non-urgent)

React can:

* render urgent updates immediately
* delay/interrupt expensive transition rendering
* keep the app interactive

---

## How to use it (common pattern)

```jsx
import { useMemo, useState, useTransition } from "react";

function Search({ items }) {
  const [query, setQuery] = useState("");
  const [filter, setFilter] = useState("");
  const [isPending, startTransition] = useTransition();

  function onChange(e) {
    const next = e.target.value;
    setQuery(next); // urgent: input stays snappy

    startTransition(() => {
      setFilter(next); // non-urgent: heavy UI depends on this
    });
  }

  const filtered = useMemo(() => {
    const q = filter.toLowerCase();
    return items.filter(x => x.name.toLowerCase().includes(q));
  }, [items, filter]);

  return (
    <>
      <input value={query} onChange={onChange} />
      {isPending && <p>Updating…</p>}
      {filtered.map(x => <div key={x.id}>{x.name}</div>)}
    </>
  );
}
```

Key point:

* `query` is immediate (fast typing)
* `filter` drives expensive UI, updated in a transition

---

## `useTransition` vs `useDeferredValue` (must-know difference)

### `useTransition`

* You control *which state update* is low priority:

  * `startTransition(() => setState(...))`

### `useDeferredValue`

* You keep a “lagging” version of a value:

  * `const deferred = useDeferredValue(value)`

Interview line:

> “Transition marks an update as non-urgent; DeferredValue gives you a deferred version of a value.”

---

## What `isPending` means

* Transition work is ongoing (React hasn’t committed the final result yet)
* Use it to show lightweight feedback:

  * “Updating…”
  * spinner on results area (not on the input)

---

## Advantages

* Smooth typing / interactions during heavy renders
* Better perceived performance
* Simple way to prioritize UI

---

## Limitations / gotchas

* Doesn’t make expensive work disappear; it schedules it better
* For huge lists, you still need **virtualization**
* Don’t wrap everything in transitions—only non-urgent UI updates
* Avoid putting urgent state updates inside `startTransition` (input should remain urgent)

---

## 30-second interview answer

`useTransition` is a concurrency hook that lets us mark certain state updates as non-urgent. React prioritizes urgent updates like typing and can delay or interrupt transition renders, keeping the UI responsive. It returns `isPending` for showing progress, and it’s commonly used for search/filtering and navigation-like updates that trigger heavy rendering.

---
