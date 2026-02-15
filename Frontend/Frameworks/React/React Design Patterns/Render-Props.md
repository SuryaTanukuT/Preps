## Render Props in React

### What it is

**Render Props** is a pattern where a component receives a **function prop** that it calls to decide what to render.

Instead of:

* “I render my own UI”
  it becomes:
* “I provide behavior/data, you provide UI”

Interview one-liner:

> “Render props share logic by passing a function that returns UI.”

---

## Basic example

```jsx
function Mouse({ children }) {
  const [pos, setPos] = React.useState({ x: 0, y: 0 });

  return (
    <div
      style={{ height: 200, border: "1px solid #ccc" }}
      onMouseMove={(e) => setPos({ x: e.clientX, y: e.clientY })}
    >
      {children(pos)}
    </div>
  );
}

// usage
<Mouse>
  {(pos) => <p>X: {pos.x}, Y: {pos.y}</p>}
</Mouse>;
```

Here:

* `Mouse` contains logic (track mouse)
* consumer chooses UI via the function `(pos) => ...`

---

## Another example: Data fetch render prop

```jsx
function Fetch({ url, children }) {
  const [state, setState] = React.useState({ data: null, error: null, loading: true });

  React.useEffect(() => {
    let alive = true;
    setState({ data: null, error: null, loading: true });

    fetch(url)
      .then((r) => r.json())
      .then((data) => alive && setState({ data, error: null, loading: false }))
      .catch((e) => alive && setState({ data: null, error: e, loading: false }));

    return () => {
      alive = false;
    };
  }, [url]);

  return children(state);
}

// usage
<Fetch url="/api/users">
  {({ data, loading, error }) => {
    if (loading) return <p>Loading...</p>;
    if (error) return <p>Error</p>;
    return data.map((u) => <div key={u.id}>{u.name}</div>);
  }}
</Fetch>;
```

---

## Why it’s used

✅ Shares logic without inheritance
✅ Very flexible composition
✅ Works well in class components and before hooks existed

---

## Downsides (interview points)

❌ Can lead to deeply nested JSX (“render prop hell”)
❌ Harder to follow than hooks in modern React
❌ Can create new functions every render (usually fine, but can affect memoization)

Interview line:

> “Render props were common before hooks; today hooks often provide the same reuse with cleaner code.”

---

## Render Props vs HOC vs Hooks

* **Render Props**: logic component + function child renders UI
* **HOC**: wrapper adds behavior and injects props
* **Hooks**: reuse logic inside components (modern default)

---

## 30-second interview answer

Render props is a pattern where a component shares logic by taking a function prop that returns UI. The component manages state/behavior, and the consumer controls rendering by implementing the function. It’s flexible and was widely used before hooks, but can cause nested JSX, so many modern apps prefer custom hooks for the same logic reuse.
