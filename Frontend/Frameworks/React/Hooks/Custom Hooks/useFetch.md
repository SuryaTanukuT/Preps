## `useFetch` in React (custom hook)

React doesn’t ship a built-in `useFetch`. Usually when people say “useFetch” they mean a **custom hook** that wraps fetching + loading/error state + cancellation.

Below is a solid, interview-friendly version.

---

## 1) Best-practice: `useFetch(url, options)`

✅ handles loading/error
✅ cancels on unmount or URL change (AbortController)
✅ avoids setting state after abort
✅ re-fetches when dependencies change

```jsx
import { useEffect, useMemo, useRef, useState } from "react";

export function useFetch(url, options = {}) {
  const { immediate = true, ...fetchOptions } = options;

  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(immediate);

  // keep options stable if caller passes new object each render
  const stableOptions = useMemo(() => fetchOptions, [JSON.stringify(fetchOptions)]);

  const abortRef = useRef(null);

  const refetch = async () => {
    if (!url) return;

    abortRef.current?.abort();
    const controller = new AbortController();
    abortRef.current = controller;

    setLoading(true);
    setError(null);

    try {
      const res = await fetch(url, { ...stableOptions, signal: controller.signal });

      if (!res.ok) {
        const text = await res.text().catch(() => "");
        throw new Error(`HTTP ${res.status} ${text}`);
      }

      const contentType = res.headers.get("content-type") || "";
      const parsed = contentType.includes("application/json")
        ? await res.json()
        : await res.text();

      setData(parsed);
      return parsed;
    } catch (e) {
      if (e.name !== "AbortError") setError(e);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    if (!immediate) return;

    refetch();

    return () => abortRef.current?.abort();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [url, immediate, stableOptions]);

  return { data, error, loading, refetch };
}
```

### Usage

```jsx
function Users() {
  const { data, loading, error, refetch } = useFetch("/api/users");

  if (loading) return "Loading...";
  if (error) return <button onClick={refetch}>Retry</button>;

  return data.map((u) => <div key={u.id}>{u.name}</div>);
}
```

---

## 2) Best practices (interview points)

* Don’t fetch in render; fetch in `useEffect` or via a data library.
* Use `AbortController` to avoid race conditions and state updates after unmount.
* Keep fetch options stable (don’t pass new objects each render).
* Separate concerns: consider `apiClient` wrapper, and hooks per feature (`useUsers`, `useOrders`).

---

## 3) When NOT to build `useFetch`

For real apps, you usually want caching, deduping, retries, pagination, optimistic updates.
Use:

* **RTK Query**
* **React Query / SWR**

Those are better than reinventing server-state management.

---
