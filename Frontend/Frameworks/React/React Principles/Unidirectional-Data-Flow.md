## Unidirectional Data Flow (React)

### Meaning

**Unidirectional data flow** means data moves in **one direction**:

* **Parent → Child** via props (top-down)
* Child communicates back via **callbacks/events**, not by directly changing parent state

Interview one-liner:

> “State flows down through props; updates flow up through callbacks.”

---

## Why React uses it

✅ Predictable UI updates
✅ Easier debugging (you know where data came from)
✅ Avoids hidden side effects / circular updates
✅ Components become easier to test and reason about

---

## Simple example

### Parent owns state

```jsx
import { useState } from "react";

function Parent() {
  const [count, setCount] = useState(0);

  return <Child count={count} onInc={() => setCount(c => c + 1)} />;
}
```

### Child receives data + triggers update via callback

```jsx
function Child({ count, onInc }) {
  return (
    <>
      <p>{count}</p>
      <button onClick={onInc}>+</button>
    </>
  );
}
```

Flow:

* `count` goes **down**
* action `onInc()` goes **up**

---

## How it relates to “lifting state up”

When siblings need shared data:

* lift state to common parent
* parent becomes single source of truth
* children receive props

This keeps the same one-way flow.

---

## React Context / Redux still follow the same principle

Even with global state:

* state is updated in one place (provider/store)
* components subscribe/read
* updates happen via dispatch/actions or context setters

Still predictable: data is read → actions update → UI re-renders.

---

## Common pitfalls (interview)

### 1) Mutating props

Props are read-only. Mutation breaks predictable flow.

```js
props.user.name = "x"; // ❌
```

### 2) Two sources of truth

Same data stored in parent + child causes sync issues. Prefer SSOT.

### 3) “Sideways” communication

Avoid child directly touching sibling state; use parent callback or shared store.

---

## 30-second interview answer

Unidirectional data flow in React means state is owned by a component (or store) and passed down as props. Child components never mutate parent data directly; they notify via callbacks or dispatched actions. This makes updates predictable, supports single source of truth, and simplifies debugging and testing.
