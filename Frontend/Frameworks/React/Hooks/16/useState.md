# `useState` in React (high-level interview perspective)

`useState` is a hook that lets a functional component store local state. When you update state, React re-renders the component with the new value.

`useState` is a hook for local component state. It returns the current state and a setter; calling the setter schedules an update and triggers a re-render. I use functional updates when the next state depends on the previous state, avoid mutating objects/arrays by creating new copies, and keep state minimal and close to the components that use it.

```js
const [count, setCount] = useState(0);
```

Interview one-liner:
“useState stores component-local data that changes over time and triggers re-render on update.”

---

# Basic examples

## Counter

```js
function Counter() {
  const [count, setCount] = useState(0);

  return (
    <>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </>
  );
}
```

---

## Controlled input

```js
function Form() {
  const [name, setName] = useState("");

  return (
    <input
      value={name}
      onChange={(e) => setName(e.target.value)}
    />
  );
}
```

---

# How it works (core flow)

* Component renders → `useState` gives current value + setter
* You call setter (`setCount`)
* React schedules an update
* Component re-renders with updated state

---

# When NOT to use `useState`

* If you need a value that shouldn’t trigger render → use `useRef`
* If state logic is complex with many transitions → use `useReducer`
* If state must be shared widely → consider Context / Redux / Zustand (depending)

---

# Best practices

* Keep state minimal and closest to where it’s used
* Prefer derived values instead of storing duplicates (avoid “derived state” bugs)
* Use functional updates for safety in concurrent/batched updates
* Split state to avoid huge objects that update frequently
