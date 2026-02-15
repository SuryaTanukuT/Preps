## `useActionState` in React

### What it is

`useActionState` is a hook to keep **component state in sync with the result of a form action** (often a Server Function / Server Action). It returns: ([react.dev][1])

```js
const [state, formAction, isPending] = useActionState(action, initialState, permalink?);
```

* **state**: latest value returned by your action (or `initialState` before first submit) ([react.dev][1])
* **formAction**: a wrapped action you pass to `<form action={...}>` or `<button formAction={...}>` ([react.dev][1])
* **isPending**: whether there’s a pending transition (use it for loading/disable UI) ([react.dev][1])

> Note: In earlier React Canary builds it was called `useFormState` and lived under React DOM; now it’s `useActionState`. ([react.dev][1])

---

## The most important behavior (interview must-know)

When you wrap an action with `useActionState`, **your action’s signature changes**: it receives **previous state as the first argument**, and **FormData becomes the second argument**. ([react.dev][1])

```js
async function action(prevState, formData) {
  // formData is 2nd arg now
  return nextState;
}
```

This is the #1 reason people say “my action can’t read form data anymore.” ([react.dev][1])

---

## Example 1: Basic “button action” (client-only idea)

```jsx
import { useActionState } from "react";

async function increment(prev) {
  return prev + 1;
}

export default function CounterForm() {
  const [count, formAction] = useActionState(increment, 0);

  return (
    <form>
      <p>{count}</p>
      <button formAction={formAction}>Increment</button>
    </form>
  );
}
```

This matches the official pattern: `formAction` goes on `formAction` for a button inside a form. ([react.dev][1])

---

## Example 2: Next.js Server Action + validation error message + pending

**Server action** (`app/actions.ts`):

```ts
"use server";

export async function createUser(prevState: { message: string }, formData: FormData) {
  const email = String(formData.get("email") || "");

  if (!email.includes("@")) {
    return { message: "Invalid email" };
  }

  // mutate data...
  return { message: "User created ✅" };
}
```

Next.js notes the **`prevState` first param** when using `useActionState`. ([Next.js][2])

**Client component** (`app/ui/signup.tsx`):

```jsx
"use client";

import { useActionState } from "react";
import { createUser } from "@/app/actions";

const initialState = { message: "" };

export function Signup() {
  const [state, formAction, pending] = useActionState(createUser, initialState);

  return (
    <form action={formAction}>
      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="text" required />

      <p aria-live="polite">{state.message}</p>

      <button type="submit" disabled={pending}>
        {pending ? "Submitting..." : "Sign up"}
      </button>
    </form>
  );
}
```

Next.js explicitly documents using `pending` from `useActionState` to disable/indicate loading. ([Next.js][2])

---

## Progressive enhancement: `permalink?` (advanced but interview-worthy)

`permalink` is an optional URL string for pages with dynamic content (feeds, etc.). If the action is a **server function** and the form is submitted **before JS loads**, the browser will navigate to the permalink instead of the current URL. After hydration it has no effect. ([react.dev][1])

---

## Advantages

* **Less boilerplate**: no manual `useState` for “server response message” + pending flag wiring ([react.dev][1])
* Works nicely with frameworks supporting Server Components: can show the server response even before hydration completes (progressive UX). ([react.dev][1])
* Clear pattern for **validation messages** and **pending UI**. ([Next.js][2])

---

## Disadvantages / gotchas

* Action signature changes (prevState first) → easy to break FormData handling. ([react.dev][1])
* `initialState` is ignored after the first invocation; after that the state is whatever your action returns. ([react.dev][1])
* The state should be **serializable** if you rely on server-driven behavior and progressive enhancement (practically: return plain objects/strings).

---

## Best practices

1. **Return a structured state** (e.g., `{ message, fieldErrors }`) so your UI can render errors predictably. ([Next.js][2])
2. Use **`pending`** for disabling submit and showing inline loading text/spinner. ([Next.js][2])
3. Remember the signature: `action(prevState, formData)` when wrapped. ([react.dev][1])
4. For button-level server actions inside a form, pass `formAction` to the specific `<button formAction={...}>` when needed. ([react.dev][1])

If you want the next hook in this “React 19 forms” family, the natural pair is **`useFormStatus`** (pending + form metadata scoped to a form subtree). ([Next.js][2])

[1]: https://react.dev/reference/react/useActionState "useActionState – React"
[2]: https://nextjs.org/docs/app/guides/forms "Guides: Forms | Next.js"
