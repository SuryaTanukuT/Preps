## `useAuth` in React

`useAuth` is **not a built-in React hook**. It’s a **custom hook pattern** you create to expose authentication data + actions (login/logout, user, token, roles) in a clean way across your app.

---

## Why we create `useAuth`

* Avoid **props drilling** (passing user/token everywhere)
* Keep auth logic **centralized**
* Provide a consistent API:

  * `user`, `isAuthenticated`, `isLoading`
  * `login()`, `logout()`, `hasRole()`, `hasPermission()`

---

## The standard pattern (AuthContext + useAuth hook)

### 1) Create Context + Provider

```jsx
import React, { createContext, useContext, useMemo, useState } from "react";

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);        // { id, name, roles, ... }
  const [token, setToken] = useState(null);      // access token
  const [loading, setLoading] = useState(false);

  async function login(credentials) {
    setLoading(true);
    try {
      // call API
      // const { user, token } = await api.login(credentials);
      setUser({ id: "1", name: "Surya", roles: ["admin"] });
      setToken("access-token");
    } finally {
      setLoading(false);
    }
  }

  function logout() {
    setUser(null);
    setToken(null);
  }

  const value = useMemo(() => {
    const isAuthenticated = !!user && !!token;

    return {
      user,
      token,
      isAuthenticated,
      loading,
      login,
      logout,
      hasRole: (role) => user?.roles?.includes(role) ?? false,
    };
  }, [user, token, loading]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
}
```

### 2) Create the `useAuth` hook (safe + clean)

```jsx
export function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) {
    throw new Error("useAuth must be used inside <AuthProvider>");
  }
  return ctx;
}
```

### 3) Use it anywhere

```jsx
function Header() {
  const { user, isAuthenticated, logout } = useAuth();

  return (
    <div>
      {isAuthenticated ? (
        <>
          <span>Hi, {user.name}</span>
          <button onClick={logout}>Logout</button>
        </>
      ) : (
        <span>Guest</span>
      )}
    </div>
  );
}
```

---

## Best practices (interview points)

* Keep context value **stable** (`useMemo`) to reduce unnecessary re-renders.
* Split contexts if needed (e.g., `AuthStateContext` and `AuthActionsContext`) when auth state changes frequently.
* Don’t store sensitive tokens in unsafe places; prefer **httpOnly cookies** when possible (depends on your architecture).
* Expose helper methods like `hasRole/hasPermission` for RBAC-style UI gating.

---

## 20-second interview answer

`useAuth` is a custom hook typically built on top of React Context to provide authentication state and actions across the app—like current user, token, login/logout, and role checks—without props drilling. It centralizes auth logic and gives a clean, reusable API, usually implemented as `AuthProvider + useAuth()`.


