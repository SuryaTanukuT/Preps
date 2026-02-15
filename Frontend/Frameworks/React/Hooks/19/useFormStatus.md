## `useFormStatus` (React / React DOM) — interview-ready

### What it is

`useFormStatus` is a hook that lets a component **read the submission status of the nearest parent `<form>`**. It’s mainly used with **Server Actions / form actions** to show loading UI while the form is submitting. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))

It returns a status object like:

* `pending` (boolean)
* plus some metadata about the submission (depends on environment) ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))

Interview one-liner:

> “`useFormStatus` lets any child of a form know if that form is currently submitting, so you can disable buttons and show pending UI.”

---

## Why it’s used

Without it, passing `isLoading` down to a nested submit button becomes props drilling. With `useFormStatus`, the submit button can decide on its own:

* disable while submitting
* show “Submitting…” text
* prevent double submits

---

## Where it’s used (most common)

✅ **A custom `<SubmitButton />` component** inside a form
✅ Buttons or UI deep inside the form tree
✅ Showing global “Saving…” banners inside a form

---

## Basic example: Submit button that auto-disables

```jsx
"use client";

import { useFormStatus } from "react-dom";

function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? "Submitting..." : "Submit"}
    </button>
  );
}

export default function SignupForm({ action }) {
  return (
    <form action={action}>
      <input name="email" />
      <SubmitButton />
    </form>
  );
}
```

This matches the official pattern: `useFormStatus` must be used in a **descendant** of the form. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))

---

## Important rules / gotchas (interview points)

### 1) Must be inside the form subtree

If the component using `useFormStatus` is not a descendant of the `<form>`, it won’t work. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))

### 2) It is scoped to the *nearest* parent form

If you nest forms (generally not recommended), it tracks the nearest one.

### 3) Works best with form actions / server actions

It’s designed around the action-based form submission model. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))

---

## `useFormStatus` vs `useActionState` (clear difference)

* **`useFormStatus`**: read-only form submission state (`pending`) for UI control in descendants. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))
* **`useActionState`**: stores action result state + gives you `formAction` + `pending`. ([react.dev](https://react.dev/reference/react/useActionState))

Interview line:

> “`useActionState` manages action result state; `useFormStatus` is for pending UI anywhere inside the form.”

---

## Best practices

* Create a reusable `<SubmitButton />` that uses `useFormStatus` so every form has consistent pending behavior. ([react.dev](https://react.dev/reference/react-dom/hooks/useFormStatus))
* Disable submit during `pending` to avoid duplicates.
* Show inline pending text rather than blocking the whole page for small forms.

---
