## Lifting State Up (React)

### What it means

**Lifting state up** is moving state to the **closest common parent** so multiple child components can **share the same source of truth**.

Interview one-liner:

> “When two components need the same changing data, lift the state to their nearest common parent and pass it down via props.”

---

## Why we do it

* Two siblings must stay in sync (filters, selected item, stepper state)
* One component updates, another component displays
* Avoid duplicated state and mismatch bugs

---

## Classic example: two inputs share same value

### Before (bad: duplicated state)

```jsx
function Celsius() {
  const [c, setC] = useState("");
  return <input value={c} onChange={(e) => setC(e.target.value)} />;
}

function Fahrenheit() {
  const [f, setF] = useState("");
  return <input value={f} onChange={(e) => setF(e.target.value)} />;
}
```

They won’t sync.

### After (lifted state to parent)

```jsx
import { useState } from "react";

function TempInputs() {
  const [celsius, setCelsius] = useState("");

  const fahrenheit =
    celsius === "" ? "" : String((Number(celsius) * 9) / 5 + 32);

  return (
    <>
      <CelsiusInput value={celsius} onChange={setCelsius} />
      <FahrenheitInput value={fahrenheit} />
    </>
  );
}

function CelsiusInput({ value, onChange }) {
  return <input value={value} onChange={(e) => onChange(e.target.value)} />;
}

function FahrenheitInput({ value }) {
  return <input value={value} readOnly />;
}
```

---

## What changes when you lift state up

* Parent owns state: `const [value, setValue] = useState()`
* Child becomes controlled:

  * gets `value` as prop
  * calls `onChange(...)` to update parent
* Siblings automatically stay consistent

---

## Pros

✅ Single source of truth
✅ Easier debugging (state lives in one place)
✅ Predictable data flow (top-down props)

---

## Cons / tradeoffs

❌ Prop drilling if lifted too high and passed through many layers
❌ Parent can become “god component” if it owns too much state

---

## Best practices

* Lift state **only as high as necessary**
* For deep trees:

  * use Context for global-ish state (theme/auth)
  * use state libraries or selector-based stores (Redux/Zustand) for large shared state
* Prefer derived values instead of duplicating state (avoid “two sources of truth”)

---

## 30-second interview answer

Lifting state up means moving shared state to the nearest common ancestor so sibling components can stay in sync. The parent owns the state and passes values and callbacks down as props, making the children controlled components. This avoids duplicated state bugs, but if it causes heavy prop drilling, context or a store can be used.
