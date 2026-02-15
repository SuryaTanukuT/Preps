## `useDarkMode` in React (custom hook)

A typical `useDarkMode` hook:

* tracks current theme: `"light"` / `"dark"`
* persists to `localStorage`
* respects OS preference (`prefers-color-scheme`)
* toggles a class on `<html>` (common with Tailwind: `dark`)

---

## Best-practice implementation (Tailwind-friendly)

```jsx
import { useEffect, useMemo, useState, useCallback } from "react";

function getSystemTheme() {
  if (typeof window === "undefined") return "light";
  return window.matchMedia?.("(prefers-color-scheme: dark)").matches ? "dark" : "light";
}

function getStoredTheme(key) {
  if (typeof window === "undefined") return null;
  try {
    return localStorage.getItem(key); // "light" | "dark" | null
  } catch {
    return null;
  }
}

export function useDarkMode(options = {}) {
  const {
    storageKey = "theme",
    defaultTheme = "system", // "light" | "dark" | "system"
    applyTo = "documentElement", // "documentElement" or "body"
    className = "dark",          // Tailwind uses "dark"
  } = options;

  const [theme, setTheme] = useState(() => {
    const stored = getStoredTheme(storageKey);
    if (stored === "light" || stored === "dark") return stored;

    if (defaultTheme === "light" || defaultTheme === "dark") return defaultTheme;

    // defaultTheme === "system"
    return getSystemTheme();
  });

  const toggle = useCallback(() => {
    setTheme((t) => (t === "dark" ? "light" : "dark"));
  }, []);

  const setLight = useCallback(() => setTheme("light"), []);
  const setDark = useCallback(() => setTheme("dark"), []);

  // Apply class + persist
  useEffect(() => {
    if (typeof document === "undefined") return;

    const root = applyTo === "body" ? document.body : document.documentElement;

    if (theme === "dark") root.classList.add(className);
    else root.classList.remove(className);

    try {
      localStorage.setItem(storageKey, theme);
    } catch {}
  }, [theme, storageKey, applyTo, className]);

  // If user chose system theme behavior (optional advanced):
  // If you want to auto-switch when OS theme changes, you usually store "system"
  // separately. Here we keep it simple (stores actual theme).
  const value = useMemo(
    () => ({ theme, toggle, setLight, setDark }),
    [theme, toggle, setLight, setDark]
  );

  return value;
}
```

### Usage

```jsx
function ThemeToggle() {
  const { theme, toggle } = useDarkMode();

  return (
    <button onClick={toggle}>
      Current: {theme}
    </button>
  );
}
```

---

## Interview points (short)

* Use `prefers-color-scheme` for system default.
* Persist choice to `localStorage`.
* Apply `dark` class to `<html>` so CSS/Tailwind can switch themes.
* Guard for SSR (`typeof window === "undefined"`) to avoid hydration issues.

---
