## `useLocalStorage` in React (custom hook)

A `useLocalStorage` hook keeps state in sync with `localStorage` so:

* value persists across refresh
* React UI stays in sync

---

## Best-practice implementation

✅ supports lazy init
✅ supports functional updates
✅ handles JSON parse errors
✅ listens to `storage` events (sync across tabs)
✅ SSR-safe guard

```jsx
import { useCallback, useEffect, useState } from "react";

export function useLocalStorage(key, initialValue) {
  const readValue = useCallback(() => {
    if (typeof window === "undefined") return initialValue;

    try {
      const item = window.localStorage.getItem(key);
      return item !== null ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  }, [key, initialValue]);

  const [storedValue, setStoredValue] = useState(readValue);

  const setValue = useCallback(
    (value) => {
      setStoredValue((prev) => {
        const next = value instanceof Function ? value(prev) : value;

        if (typeof window !== "undefined") {
          try {
            window.localStorage.setItem(key, JSON.stringify(next));
          } catch {
            // ignore quota/security errors
          }
        }

        return next;
      });
    },
    [key]
  );

  // Update state if key changes or storage changes in another tab
  useEffect(() => {
    setStoredValue(readValue);

    function onStorage(e) {
      if (e.key !== key) return;
      setStoredValue(readValue());
    }

    window.addEventListener("storage", onStorage);
    return () => window.removeEventListener("storage", onStorage);
  }, [key, readValue]);

  const remove = useCallback(() => {
    if (typeof window !== "undefined") {
      try {
        window.localStorage.removeItem(key);
      } catch {}
    }
    setStoredValue(initialValue);
  }, [key, initialValue]);

  return [storedValue, setValue, remove];
}
```

---

## Usage example

```jsx
function Demo() {
  const [theme, setTheme, removeTheme] = useLocalStorage("theme", "light");

  return (
    <>
      <p>Theme: {theme}</p>
      <button onClick={() => setTheme(t => (t === "light" ? "dark" : "light"))}>
        Toggle
      </button>
      <button onClick={removeTheme}>Reset</button>
    </>
  );
}
```

---

## Interview points

* `localStorage` stores **strings only** → use JSON serialize/parse.
* Reads should be **lazy** (init function) to avoid repeated parsing.
* `storage` event helps **sync across tabs** (it fires in other tabs, not the same tab).
* SSR guard (`typeof window`) prevents hydration/runtime errors.

---
