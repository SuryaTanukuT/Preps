# `useRef` in React (interview-ready)

`useRef` creates a stable object like `{ current: ... }` that stays the same across renders.

`useRef` provides a stable object whose `.current` persists across renders.
It’s used to reference DOM elements for imperative actions like focus and scrolling, and to store mutable values like timer IDs or previous state without triggering re-renders. If the UI needs to update, use `useState` instead.

Two main uses:

* DOM reference (focus / scroll / measure)
* Mutable value storage that doesn’t trigger re-render

Interview one-liner:
“useRef stores a persistent mutable value or DOM node without causing re-renders.”

---

# 1) DOM ref use case (most common)

## Focus an input

```js
import { useRef } from "react";

function SearchBox() {
  const inputRef = useRef(null);

  return (
    <>
      <input ref={inputRef} placeholder="Search..." />
      <button onClick={() => inputRef.current?.focus()}>
        Focus
      </button>
    </>
  );
}
```

---

## Scroll into view

```js
const sectionRef = useRef(null);

<button
  onClick={() =>
    sectionRef.current?.scrollIntoView({ behavior: "smooth" })
  }
>
  Go
</button>

<div ref={sectionRef}>Target</div>
```

---

# `useRef` vs `useState` (must know)

## `useState`

* Changing it triggers re-render
* Use when UI must update

## `useRef`

* Changing `.current` does not re-render
* Use for imperative work or values that shouldn’t trigger UI updates

Interview line:
“State drives rendering; refs are an escape hatch for mutable values.”

---

# Common pitfalls

❌ Using ref when UI depends on the value
If UI must show the new value, don’t use ref—use state.

❌ Expecting ref updates to re-render
Updating `ref.current` won’t update the UI automatically.

---

# Best practices

* Use refs for focus / scroll / measure and integration with non-React libs
* Cleanup timers/listeners in `useEffect` cleanup
* Don’t overuse refs to “avoid re-render” — it can create buggy UI
