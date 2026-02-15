# `React.memo` (interview + practical)

React.memo is a higher-order component that memoizes a functional component and skips re-rendering it when its props haven’t changed (by shallow comparison).

React.memo memoizes a functional component and skips re-render when props haven’t changed based on shallow equality. It’s useful for expensive components that re-render often with the same props, especially in lists. It can be defeated by unstable object/function props, so I pair it with stable references (`useCallback` / `useMemo`) when necessary and apply it selectively after profiling.

```js
const UserRow = React.memo(function UserRow({ user, onSelect }) {
  return <div onClick={() => onSelect(user.id)}>{user.name}</div>;
});
```

Interview one-liner:
“React.memo prevents unnecessary re-renders by shallowly comparing props.”

---

# Why we use it

React re-renders children when a parent re-renders. Often the child output is the same → wasted work.

React.memo helps when:

* Component render is expensive
* It re-renders frequently due to parent state changes
* Props are usually stable

---

# How it works (key detail)

Default comparison is shallow:

* Primitives compare by value
* Objects/arrays/functions compare by reference

So this breaks memo:

```jsx
<UserRow user={user} onSelect={() => setId(user.id)} />
// new function each render
```

To benefit, pass stable references:

* `useCallback` for functions
* `useMemo` for objects/arrays (only if needed)

---

# When `React.memo` is NOT useful

* Component is cheap to render
* Props change almost every render
* You create new object/function props every render and can’t stabilize them
* The component uses `useContext` that changes often (it will re-render when context value changes)
