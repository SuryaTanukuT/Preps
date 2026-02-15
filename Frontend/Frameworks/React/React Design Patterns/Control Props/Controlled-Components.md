## Controlled Components in React (interview-ready)

### What it is

A **controlled component** is a form input whose value is **controlled by React state**.

Meaning:

* React state is the “single source of truth”
* Input shows `value={state}`
* Input updates through `onChange → setState`

Interview one-liner:

> “Controlled components bind input value to state, so React fully controls the form data.”

---

## Basic example (text input)

```jsx
import { useState } from "react";

function NameForm() {
  const [name, setName] = useState("");

  return (
    <>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
        placeholder="Enter name"
      />
      <p>You typed: {name}</p>
    </>
  );
}
```

---

## Checkbox (controlled)

```jsx
function Terms() {
  const [accepted, setAccepted] = useState(false);

  return (
    <input
      type="checkbox"
      checked={accepted}
      onChange={(e) => setAccepted(e.target.checked)}
    />
  );
}
```

---

## Select (controlled)

```jsx
function Country() {
  const [country, setCountry] = useState("IN");

  return (
    <select value={country} onChange={(e) => setCountry(e.target.value)}>
      <option value="IN">India</option>
      <option value="DE">Germany</option>
    </select>
  );
}
```

---

## Why controlled components are used

✅ Instant validation (onChange/onBlur)
✅ Conditional UI based on input value
✅ Easy to submit full form state
✅ Reset/clear programmatically
✅ Keep UI consistent with state

Interview line:

> “Controlled inputs are great when you need validation, dynamic behavior, or programmatic control.”

---

## Downsides / tradeoffs

❌ More boilerplate (`value + onChange + state`)
❌ Every keystroke triggers state update → re-render (usually fine, but large forms can feel heavy if poorly structured)
❌ Requires careful handling for performance in huge forms

Interview line:

> “For very large forms, uncontrolled inputs or libraries like React Hook Form can reduce re-render overhead.”

---

## Controlled vs Uncontrolled (quick compare)

### Controlled

* value stored in React state
* easier validation, dynamic UI
* more code

### Uncontrolled

* value stored in DOM (read using ref on submit)
* less re-rendering
* harder live validation

---

## Common interview pitfalls

### 1) Don’t mix `value` with uncontrolled usage

If you set `value`, React expects you to manage it. Use `defaultValue` for uncontrolled.

### 2) `onChange` is required

If you pass `value` without `onChange`, input becomes read-only.

### 3) Number input gives string

```js
setAge(Number(e.target.value));
```

---

## 30-second interview answer

Controlled components are form inputs whose value is driven by React state using `value`/`checked` props and updated through `onChange`. They make validation, conditional UI, and programmatic control easy because React is the source of truth. The tradeoff is more boilerplate and potential performance concerns in very large forms, where uncontrolled inputs or React Hook Form may be better.

---
