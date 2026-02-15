## HOC (Higher-Order Component) in React

### What it is

A **Higher-Order Component (HOC)** is a function that takes a component and returns a new component with added behavior.

```js
const Enhanced = withSomething(BaseComponent);
```

Interview one-liner:

> “An HOC is a pattern for reusing component logic by wrapping a component.”

---

## Why HOCs were used (historically)

Before Hooks, HOCs were a common way to share logic such as:

* authentication / route protection
* data fetching
* logging / analytics
* theming
* permissions (RBAC)
* injecting props from context/store (older React-Redux patterns)

Today, many HOC use cases are better served by:

* custom hooks
* render props
* composition

But HOCs still exist in real codebases and libraries.

---

## Basic example: `withLoading`

```jsx
import React from "react";

function withLoading(Component) {
  return function WithLoading(props) {
    if (props.loading) return <p>Loading...</p>;
    return <Component {...props} />;
  };
}

// usage
function UserList({ users }) {
  return users.map(u => <div key={u.id}>{u.name}</div>);
}

const UserListWithLoading = withLoading(UserList);
```

---

## Example: Auth guard HOC

```jsx
function withAuth(Component) {
  return function WithAuth(props) {
    const isLoggedIn = Boolean(localStorage.getItem("token")); // demo only
    if (!isLoggedIn) return <p>Please login</p>;
    return <Component {...props} />;
  };
}
```

---

## Advantages

✅ Reuse logic across multiple components
✅ Works well for cross-cutting concerns (logging, permissions)
✅ Familiar pattern in older ecosystems / libraries

---

## Disadvantages (common interview points)

❌ Wrapper nesting (“wrapper hell”)
❌ Harder debugging (component tree becomes deeper)
❌ Props collisions (HOC injects prop names that conflict)
❌ Ref forwarding issues (need `forwardRef`)
❌ Static methods not auto-copied (`displayName`, etc.)

Interview line:

> “HOCs can make trees harder to trace and are less ergonomic than hooks.”

---

## Best practices

1. **Use `displayName`** for better DevTools debugging

```js
WithLoading.displayName = `withLoading(${Component.displayName || Component.name || "Component"})`;
```

2. **Avoid prop name collisions**

* namespace injected props or document them clearly

3. **Forward refs if needed**

```jsx
const Enhanced = React.forwardRef((props, ref) => (
  <Component {...props} forwardedRef={ref} />
));
```

4. Prefer **hooks** for shared logic unless:

* you need to wrap class components
* you’re integrating with a library expecting HOCs
* you want a consistent “wrapper” API at component export time

---

## HOC vs Custom Hook (interview comparison)

* **HOC**: wraps component, injects behavior via props
* **Hook**: called inside component, returns values/functions

Modern recommendation:

> “Use hooks for logic reuse; keep HOCs for legacy or cross-cutting wrappers.”

---

## 30-second interview answer

A Higher-Order Component is a function that takes a component and returns a new component with extra behavior. It was widely used before hooks for sharing logic like auth, data fetching, and logging. Downsides include wrapper nesting, debugging complexity, and prop collisions, so today hooks are usually preferred, but HOCs still appear in legacy and library code.
