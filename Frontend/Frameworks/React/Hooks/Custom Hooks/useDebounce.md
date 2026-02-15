## `useDebounce` in React (custom hook)

### What it does

`useDebounce` delays updating a value until the user **stops changing it** for a given time (e.g., 300ms).

Use it for:

* search input API calls
* filtering requests
* autosave
* validation (don’t validate on every keystroke)

---

## 1) Debounce a **value** (most common)

### Hook

```jsx
import { useEffect, useState } from "react";

export function useDebouncedValue(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);

  return debounced;
}
```

### Usage (debounced search)

```jsx
import { useEffect, useState } from "react";
import { useDebouncedValue } from "./useDebouncedValue";

function Search() {
  const [q, setQ] = useState("");
  const debouncedQ = useDebouncedValue(q, 400);

  useEffect(() => {
    if (!debouncedQ) return;
    // call API with debouncedQ
    // fetch(`/api/search?q=${encodeURIComponent(debouncedQ)}`)
  }, [debouncedQ]);

  return <input value={q} onChange={(e) => setQ(e.target.value)} />;
}
```

---

## 2) Debounce a **function** (when you want a stable debounced handler)

### Hook

```jsx
import { useEffect, useMemo, useRef } from "react";

export function useDebouncedCallback(fn, delay = 300) {
  const fnRef = useRef(fn);
  useEffect(() => {
    fnRef.current = fn;
  }, [fn]);

  const debounced = useMemo(() => {
    let timer;
    const wrapped = (...args) => {
      clearTimeout(timer);
      timer = setTimeout(() => fnRef.current(...args), delay);
    };
    wrapped.cancel = () => clearTimeout(timer);
    return wrapped;
  }, [delay]);

  return debounced;
}
```

### Usage

```jsx
function Search() {
  const [q, setQ] = useState("");

  const debouncedLog = useDebouncedCallback((text) => {
    console.log("API call for:", text);
  }, 400);

  return (
    <input
      value={q}
      onChange={(e) => {
        const v = e.target.value;
        setQ(v);
        debouncedLog(v);
      }}
    />
  );
}
```

---

## Debounce vs Throttle (quick)

* **Debounce**: run after user stops (best for search)
* **Throttle**: run at most once every N ms (best for scroll/resize)

---

## Best practices

* Debounce **server calls**, not the input UI update (input should stay instant).
* Always clean up timers in `useEffect`.
* For search + caching, combine debounce with React Query / RTK Query patterns.
* Pick a delay like **250–500ms** for typing UX.

---
