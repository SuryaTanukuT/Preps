# `useCallback`

`useCallback` memoizes a function reference. It returns the same function instance between renders until its dependencies change.
useCallback memoizes function references to keep them stable across renders. It’s mainly useful when function identity impacts React.memo children or hook dependency arrays (effects/subscriptions). Overusing it adds complexity and can introduce stale closures, so it should be applied selectively after profiling or when a known pattern requires stable callbacks.

* Yes, useCallback uses closures.
* The dependency array controls when React re-creates the closure with fresh values.
* Empty `[]` → closure is frozen with initial values.
* Proper dependencies → closure updates when needed.

```js
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

Think of it as:
`useCallback(fn, deps) ≈ useMemo(() => fn, deps)`

---

# Why it’s used

React re-renders often. Every render creates new function objects:

```jsx
<button onClick={() => setCount(c => c + 1)} />
```

That’s fine usually. But new function references can cause:

* Memoized children (`React.memo`) to re-render (because props changed by reference)
* Effects that depend on functions to re-run unnecessarily
* Unstable handlers passed deep into component trees

So useCallback helps keep function identity stable when that stability matters.

---

# Where it is used (most common)

## 1) Passing callbacks to memoized child components

If child is wrapped with `React.memo`, it shallow-compares props.
New function reference = “prop changed” → re-render.

```jsx
const Row = React.memo(({ item, onSelect }) => {
  return <button onClick={() => onSelect(item.id)}>{item.name}</button>;
});

function List({ items }) {
  const [selected, setSelected] = useState(null);

  const onSelect = useCallback((id) => {
    setSelected(id);
  }, []);

  return items.map(i => (
    <Row key={i.id} item={i} onSelect={onSelect} />
  ));
}
```

---

# When is `useCallback` not needed?

* The child doesn’t receive functions as props (only primitives or stable objects).
* You don’t care if the child re-renders (small list, cheap renders).
* The function prop is stable for other reasons:

  * It comes from a stable source (e.g., `dispatch` from `useReducer` is stable).
  * You memoize at a higher level (e.g., you `useMemo` the props object you pass).
  * You use a custom comparison with `React.memo(Component, areEqual)` that ignores certain props.
* You’re already using patterns like child supplies arguments (child calls `onSelect(item.id)`), and the callback identity is stable.

---

# Without `useCallback`, child happens every time parent re-renders

```jsx
import React, { useState } from "react";

const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Click me</button>;
});

function App() {
  const [count, setCount] = useState(0);

  // New function created on every render
  const handleClick = () => {
    console.log("clicked");
    setCount(c => c + 1);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <Child onClick={handleClick} />
    </div>
  );
}

export default App;
```

---

# With `useCallback`, only renders once initially, then stays stable while the parent re-renders

```jsx
import React, { useState, useCallback } from "react";

const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Click me</button>;
});

function App() {
  const [count, setCount] = useState(0);

  // Stable function reference
  const handleClick = useCallback(() => {
    console.log("clicked");
    setCount(c => c + 1);
  }, []);

  return (
    <div>
      <p>Count: {count}</p>
      <Child onClick={handleClick} />
    </div>
  );
}

export default App;
```

---

# Without using `useCallback` and `React.memo`

App rendered  <-- happens every time state changes
clicked
App rendered  <-- happens again after setCount triggers re-render

```jsx
import React, { useState } from "react";

function App() {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    console.log("clicked");
    setCount(c => c + 1);
  };

  console.log("App rendered");

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleClick}>Click me</button>
    </div>
  );
}

export default App;
```

---

# When a function is a dependency of `useEffect`

Without `useCallback`, the function changes every render → effect re-runs.

```jsx
const fetchUsers = useCallback(async () => {
  const res = await api.get("/users");
  setUsers(res.data);
}, [api]);

useEffect(() => {
  fetchUsers();
}, [fetchUsers]);
```

---

# Stable handlers for subscriptions / listeners

If you add/remove listeners, stable callbacks prevent unnecessary re-subscribe.

---

# Disadvantages / pitfalls

* Not free: React still must store memoized functions + compare deps
* Overuse adds complexity and makes code harder to read
* Can create stale closures if dependencies are wrong
* Often gives no benefit if:

  * Child isn’t memoized
  * Callback isn’t used in deps/effects
  * Renders aren’t expensive anyway

Interview line:
“useCallback is an optimization tool; I use it when function identity affects memoization or effects.”

---

# Common mistakes (interview favorites)

## Mistake 1: Missing dependencies → stale values

```js
const onSave = useCallback(() => api.save(form), []); // stale form
```

Fix:

```js
const onSave = useCallback(() => api.save(form), [form]);
```

---

## Mistake 2: Using it everywhere

```js
const onClick = useCallback(() => setX(1), []); // often unnecessary
```

Better: only when needed.

---

## Mistake 3: Memoizing parent callback but child still re-renders

If you pass new objects/arrays too, memo still breaks:

```jsx
<Child onClick={onClick} options={{ a: 1 }} /> // options new every time
```

---

# Rule of thumb

* Empty `[]` → only safe if the callback doesn’t depend on changing values.
* Add dependencies → whenever you use props/state inside the callback.
* React’s ESLint plugin will warn you if you forget dependencies — that’s a good guide to follow.

---

# Practical use cases

* Large lists with memoized row components
* Reusable UI components (Button, Input) receiving callbacks
* Debounced handlers (search input) where you pass handler to hooks
* Custom hooks that accept callbacks and depend on stability
* Event subscriptions (WebSocket, window listeners) with stable handler identity
* Memoized computed props where callbacks are part of object passed down
