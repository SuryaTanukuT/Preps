# `useReducer` in React (high-level interview perspective)

`useReducer` is a hook for managing state using a reducer function (like Redux pattern) instead of multiple `useState` calls.

It’s best when state logic is:

* Complex
* Has multiple sub-values
* Updated in different ways
* Depends on “what happened” (actions)

`useReducer` manages state through a reducer function and dispatching actions. It’s useful when state logic is complex, involves multiple related values, or when different events cause different transitions.

It centralizes updates into a predictable switch-case reducer, improves maintainability, and avoids scattered `useState` updates. For simple state I still use `useState`.

Interview one-liner:
“useReducer is good for complex state transitions where actions describe changes.”

---

# Why use `useReducer` (when it’s better than `useState`)

Use `useReducer` when:

* Many related state variables change together
* Updates depend on previous state (non-trivial)
* You want predictable transitions (action-based)
* You want to centralize logic and avoid scattered `setState` calls
* Forms / wizards / multi-step workflows

`useState` is better when:

* State is simple (count, toggle, small form)
* Few independent values

---

# How it works

`useReducer(reducer, initialState)` returns:

* `state`
* `dispatch(action)`

Reducer signature:

```js
(state, action) => newState
```

---

# Simple example

```js
import { useReducer } from "react";

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "inc":
      return { count: state.count + 1 };
    case "dec":
      return { count: state.count - 1 };
    case "reset":
      return initialState;
    default:
      return state;
  }
}

export default function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "inc" })}>+</button>
      <button onClick={() => dispatch({ type: "dec" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </>
  );
}
```

---

# Form / complex state example (common use case)

```js
const initial = { name: "", email: "", errors: {} };

function reducer(state, action) {
  switch (action.type) {
    case "field":
      return { ...state, [action.field]: action.value };
    case "setErrors":
      return { ...state, errors: action.errors };
    case "reset":
      return initial;
    default:
      return state;
  }
}
```

Use:

```js
dispatch({ type: "field", field: "name", value: e.target.value });
```

---

# Advantages

* Centralized state update logic (easy to debug)
* Predictable transitions (actions)
* Easier to manage “many updates” cleanly
* Works well with complex UI flows

---

# Disadvantages

* More boilerplate than `useState`
* Overkill for simple state
* Reducers must stay pure (no side effects inside reducer)

Interview line:
“Reducers should be pure; async work stays outside (effects/handlers).”

---

# Best practices

* Keep reducer pure (no API calls, no random, no mutations)
* Use meaningful action types
* Normalize complex state shape
* Combine with Context when multiple components need same reducer state

`useReducer + Context = light Redux pattern`

If state grows very large or needs devtools/middleware:

Consider Redux Toolkit / Zustand
