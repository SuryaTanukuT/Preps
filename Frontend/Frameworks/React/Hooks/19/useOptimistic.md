## `useOptimistic` in React

### What it is

`useOptimistic` is a React Hook to show an **optimistic UI update** while an async action (network request/server action) is in progress. It gives you an “optimistic copy” of your state plus a function to apply optimistic updates. ([react.dev][1])

**Signature**

```js
const [optimisticState, addOptimistic] = useOptimistic(state, updateFn?);
```

* `state`: your real source-of-truth state
* `updateFn(state, input)`: how to compute the optimistic state from the current optimistic state + an input you pass to `addOptimistic` ([react.dev][1])

---

## Why it’s used

Without optimistic UI: user clicks “Like/Add/Save” → UI waits for server → feels slow.

With `useOptimistic`: UI updates **immediately**, assuming success; if the server fails, you **revert** by syncing back to the real state.

Interview one-liner:

> “`useOptimistic` makes interactions feel instant by temporarily showing the expected result while the request is pending.” ([react.dev][1])

---

## Basic example: optimistic “add comment”

```jsx
"use client";
import { useOptimistic, useState } from "react";

export default function Comments() {
  const [comments, setComments] = useState([]);

  // optimisticState starts from real comments
  const [optimisticComments, addOptimisticComment] = useOptimistic(
    comments,
    (current, newComment) => [...current, newComment]
  );

  async function addComment(text) {
    const temp = { id: crypto.randomUUID(), text, optimistic: true };
    addOptimisticComment(temp); // UI updates instantly

    try {
      // call server
      const saved = await fakeServerSave(text);

      // commit real state from server result
      setComments((prev) => [...prev, saved]);
    } catch (e) {
      // on failure: do NOT commit. optimistic entry disappears
      // because optimistic state is derived from `comments` (real state)
      console.error(e);
    }
  }

  return (
    <>
      <button onClick={() => addComment("Hello")}>Add</button>

      {optimisticComments.map((c) => (
        <div key={c.id}>
          {c.text} {c.optimistic ? "(sending...)" : ""}
        </div>
      ))}
    </>
  );
}

// demo
async function fakeServerSave(text) {
  await new Promise((r) => setTimeout(r, 500));
  return { id: crypto.randomUUID(), text };
}
```

**How rollback works here**

* If server fails, `setComments` is not called.
* Since optimistic UI is a *temporary layer* derived from `comments`, the optimistic item disappears when the real state remains unchanged. ([he.react.dev][2])

---

## Where it’s commonly used

* Like/unlike, follow/unfollow
* Adding items to a list (comments, todos)
* Inline edits (rename, toggle)
* Form submissions with Server Actions (common in Next.js App Router) ([Next.js][3])

---

## Best practices

1. **Keep real state as the source of truth.** Optimistic state should be derived from it. ([he.react.dev][2])
2. **Make optimistic items distinguishable** (e.g., `optimistic: true`) to show “Sending…” UI and avoid confusion.
3. **Use stable IDs** for optimistic rows (temporary IDs) so React rendering is stable.
4. **Commit with server response** (use returned ID/data) rather than assuming the optimistic payload is final.
5. **Handle failure explicitly** (toast/error UI). Rollback happens naturally if you don’t commit real state, but user should still be informed.

---

## Common pitfalls (interview)

* **Mutating state** inside the update function (should return new state)
* **Creating optimistic updates but also committing wrong “assumed” data** instead of the server response (causes inconsistencies)
* **Not marking optimistic entries** → hard to show pending state and handle edge cases
* **Overusing it** for complex multi-step operations where server response can differ greatly (optimistic UI becomes misleading)

---

## 20–30 sec interview answer

`useOptimistic` provides an optimistic version of state and a function to apply temporary updates while an async action is pending. You update the UI immediately, then either commit the real server result into your source-of-truth state or, on failure, skip committing so the UI naturally rolls back. It’s commonly used for likes, comments, and form submissions to make the app feel instant. ([react.dev][1])

[1]: https://react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic"
[2]: https://he.react.dev/reference/react/useOptimistic?utm_source=chatgpt.com "useOptimistic"
[3]: https://nextjs.org/docs/app/guides/forms?utm_source=chatgpt.com "How to create forms with Server Actions"
