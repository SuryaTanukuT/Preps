## `use` in React (`use(resource)`)

### What it is

`use` is a React API that lets you **read (unwrap) a “resource” during render** — most commonly:

* a **Promise** (async data)
* a **Context** (similar to `useContext`) ([react.dev][1])

```js
import { use } from "react";
const value = use(resource);
```

([react.dev][1])

---

## Why it exists

Before `use`, if you wanted async data in a component you often did:

* `useEffect` + `useState` + loading states + dependency management

With `use(promise)`, React can **Suspend** rendering until the promise resolves, and you show a fallback with `Suspense`. ([react.dev][2])

Interview line:

> “`use` lets React read async resources during render and integrate with Suspense for loading.”

---

## 1) Using `use` with a Promise (async data)

### Pattern: suspend until data is ready

```jsx
import { Suspense, use } from "react";

function Comments({ commentsPromise }) {
  const comments = use(commentsPromise); // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}

export default function Page({ commentsPromise }) {
  return (
    <Suspense fallback={<p>Loading comments…</p>}>
      <Comments commentsPromise={commentsPromise} />
    </Suspense>
  );
}
```

This is the canonical idea shown in React’s React 19 announcement and reference docs. ([react.dev][2])

### If the Promise rejects

A rejected promise will surface as an error that should be handled with an **Error Boundary** (React docs cover this under troubleshooting/usage). ([react.dev][1])

---

## 2) Using `use` with Context

`use(ContextObject)` can read context value similar to `useContext(ContextObject)`. ([react.dev][1])

Conceptually:

```js
const theme = use(ThemeContext);
```

([react.dev][1])

---

## Where you’ll typically see it

### ✅ React Server Components / Next.js App Router

In frameworks like **Next.js App Router**, Server Components are common, and `use` is often used to consume promises/resources as part of streaming + Suspense patterns. ([Next.js][3])

---

## Key rules / constraints (interview points)

* `use` is meant to be called **during render** (inside a component), not inside event handlers or random functions (same general “React calls components/hooks” rule). ([react.dev][4])
* When you `use(promise)`, you must rely on **Suspense** for the loading UI (fallback).
* Don’t replace normal client data fetching blindly: for many client-only cases, **React Query / RTK Query** style solutions still make sense (cache, retries, dedupe). (`use` is about reading resources with Suspense; it’s not a full server-state cache by itself.)

---

## Best practices

* Wrap the part that uses `use(promise)` in a **`<Suspense fallback=...>`** boundary. ([react.dev][2])
* Handle failures with an **Error Boundary** for a clean UX. ([react.dev][1])
* Prefer passing promises/resources **from a higher layer** (server/page loader) rather than creating new promises every render.

---

## 20–30 sec interview answer

`use(resource)` is a React API that lets a component read a Promise or Context during render. If you pass a Promise, React will suspend rendering until it resolves, and you show loading UI via Suspense; if it rejects, it’s handled via error boundaries. It’s especially useful in Server Components/streaming scenarios to reduce boilerplate compared to `useEffect + useState`. ([react.dev][2])

[1]: https://react.dev/reference/react/use?utm_source=chatgpt.com "use"
[2]: https://react.dev/blog/2024/12/05/react-19?utm_source=chatgpt.com "React v19"
[3]: https://nextjs.org/docs/app/getting-started/server-and-client-components?utm_source=chatgpt.com "Getting Started: Server and Client Components"
[4]: https://react.dev/reference/rules/react-calls-components-and-hooks?utm_source=chatgpt.com "React calls Components and Hooks"
