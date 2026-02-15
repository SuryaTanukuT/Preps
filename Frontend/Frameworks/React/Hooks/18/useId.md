## `useId` in React (interview-ready)

### What it is

`useId()` generates a **stable, unique ID string** that’s safe for **SSR + hydration**.

Main use: connect **labels ↔ inputs** (and other ARIA attributes) without causing hydration mismatches.

```jsx
import { useId } from "react";

function NameField() {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>Name</label>
      <input id={id} />
    </>
  );
}
```

---

## Why React introduced `useId`

In SSR, if you generate IDs using:

* `Math.random()`
* `Date.now()`
* incrementing counters in module scope
  server and client might produce different IDs → **hydration mismatch**.

`useId` avoids that by generating IDs consistently across server/client render.

Interview line:

> “`useId` gives hydration-safe unique IDs for accessibility attributes.”

---

## Where it’s used

✅ Accessibility:

* `htmlFor` / `id`
* `aria-describedby`
* `aria-labelledby`
* linking error text to fields

Example:

```jsx
function EmailField({ error }) {
  const inputId = useId();
  const errId = useId();

  return (
    <div>
      <label htmlFor={inputId}>Email</label>
      <input id={inputId} aria-describedby={error ? errId : undefined} />
      {error && <p id={errId}>{error}</p>}
    </div>
  );
}
```

✅ UI libraries / reusable components:

* multiple instances on page need unique ids without conflicts

---

## Important notes (interview)

### 1) Not for list keys

Don’t use `useId()` as `key` in lists. Keys must be tied to data identity, not generated per render.

### 2) Not for server IDs / DB IDs

It’s for UI wiring (labels/ARIA), not business identifiers.

### 3) It’s stable per component instance

The id stays the same across re-renders of that component instance.

---

## 20-second interview answer

`useId` generates stable unique IDs that work correctly with SSR and hydration. It’s mainly used for accessibility—linking labels to inputs and ARIA attributes—without risking hydration mismatches that can happen with random or time-based IDs.
