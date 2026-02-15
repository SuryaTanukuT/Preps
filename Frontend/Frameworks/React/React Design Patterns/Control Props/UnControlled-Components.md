## Uncontrolled Components in React (interview-ready)

### What it is

An **uncontrolled component** is a form input where the **DOM holds the current value**, not React state.

You read the value when needed (usually on submit) using:

* `ref` (most common)
* or form APIs like `FormData`

Interview one-liner:

> “Uncontrolled components let the DOM manage input state; React reads values via refs when required.”

---

## Basic example (using `ref`)

```jsx
import { useRef } from "react";

function LoginForm() {
  const emailRef = useRef(null);

  function onSubmit(e) {
    e.preventDefault();
    const email = emailRef.current.value; // read from DOM
    console.log(email);
  }

  return (
    <form onSubmit={onSubmit}>
      <input ref={emailRef} type="email" placeholder="Email" />
      <button type="submit">Submit</button>
    </form>
  );
}
```

---

## Uncontrolled with `defaultValue`

Use `defaultValue` to set initial value (instead of `value`).

```jsx
function Profile() {
  const nameRef = useRef(null);

  return <input ref={nameRef} defaultValue="Surya" />;
}
```

---

## Using `FormData` (clean approach for forms)

```jsx
function Signup() {
  function onSubmit(e) {
    e.preventDefault();
    const data = new FormData(e.currentTarget);
    console.log(Object.fromEntries(data)); // { email: "...", password: "..." }
  }

  return (
    <form onSubmit={onSubmit}>
      <input name="email" />
      <input name="password" type="password" />
      <button>Submit</button>
    </form>
  );
}
```

---

## Why uncontrolled components are used

✅ Less boilerplate (no `value/onChange` per field)
✅ Fewer re-renders while typing (DOM handles it)
✅ Works nicely with native forms and large forms
✅ Pairs well with React Hook Form (it prefers uncontrolled inputs)

Interview line:

> “Uncontrolled inputs are good when you don’t need live validation or UI updates per keystroke.”

---

## Downsides / tradeoffs

❌ Harder to do live validation and dynamic behavior
❌ Harder to conditionally format UI based on input value
❌ Need refs or FormData for reading values
❌ Resetting/setting values programmatically needs DOM updates or `reset()`

---

## Controlled vs Uncontrolled (quick compare)

### Controlled

* React state = source of truth
* easy validation & dynamic UI
* more re-renders + more code

### Uncontrolled

* DOM = source of truth
* minimal re-renders, less code
* harder live validation

---

## 30-second interview answer

Uncontrolled components keep form input state in the DOM rather than React state. React reads values using refs or FormData on submit. They reduce boilerplate and re-renders, making them good for large forms or when you don’t need live validation. Controlled components are better when UI and validation must react to every change.

---
