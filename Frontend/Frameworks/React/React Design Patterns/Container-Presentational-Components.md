## Container vs Presentational Components (React pattern)

This is a classic pattern to keep UI clean and logic reusable by separating:

* **Container (smart)**: data + state + side effects
* **Presentational (dumb)**: UI rendering only

Interview one-liner:

> “Containers handle data/logic; presentational components focus on rendering based on props.”

---

## 1) Presentational Components (UI-only)

**Characteristics**

* Receives data via **props**
* No API calls
* Minimal state (maybe UI-only like open/close)
* Easy to test and reuse
* No knowledge of where data comes from

Example:

```jsx
function UserListView({ users, loading, error, onRefresh }) {
  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;

  return (
    <>
      <button onClick={onRefresh}>Refresh</button>
      <ul>
        {users.map((u) => (
          <li key={u.id}>{u.name}</li>
        ))}
      </ul>
    </>
  );
}
```

---

## 2) Container Components (data + logic)

**Characteristics**

* Calls APIs / uses RTK Query / React Query
* Owns state (loading/error/data)
* Handles side effects (`useEffect`)
* Passes props down to presentational component

Example:

```jsx
import { useEffect, useState, useCallback } from "react";

function UsersContainer() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState("");

  const fetchUsers = useCallback(async () => {
    setLoading(true);
    setError("");
    try {
      const res = await fetch("/api/users");
      if (!res.ok) throw new Error(`HTTP ${res.status}`);
      setUsers(await res.json());
    } catch (e) {
      setError(e.message || "Failed");
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);

  return (
    <UserListView
      users={users}
      loading={loading}
      error={error}
      onRefresh={fetchUsers}
    />
  );
}
```

---

## 3) Why this pattern is useful

✅ Cleaner components (UI is separate from fetching)
✅ Reusable UI across different data sources
✅ Easier unit testing (test view with props, test container with mocks)
✅ Better maintainability in large apps

---

## 4) Tradeoffs / downsides

❌ Can create too many wrapper components
❌ Can feel verbose in small apps
❌ With modern hooks and data libraries, you often replace “containers” with:

* custom hooks (`useUsers()`)
* route-level loaders
* RTK Query hooks in the page component

Interview line:

> “Today I often use custom hooks instead of dedicated container components.”

---

## 5) Modern version (recommended): View + Hook

Instead of a container component, keep UI component + custom hook:

```jsx
function useUsers() {
  // fetch logic here (or RTK Query)
  // return { users, loading, error, refetch }
}

function UsersPage() {
  const { users, loading, error, refetch } = useUsers();
  return <UserListView users={users} loading={loading} error={error} onRefresh={refetch} />;
}
```

---

## 30-second interview answer

The container/presentational pattern separates concerns: containers handle data fetching, state, and side effects, while presentational components are pure UI driven by props. This improves reuse and testability. In modern React, containers are often replaced by custom hooks + page components, but the principle remains the same: keep UI pure and keep data logic isolated.
