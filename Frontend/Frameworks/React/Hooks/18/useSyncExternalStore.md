## `useSyncExternalStore` (React 18) — interview-ready

### What it is

`useSyncExternalStore` is the official React hook for **subscribing to external stores** (state that lives outside React), in a way that is **safe with concurrent rendering** and **SSR/hydration**.

External store examples:

* Redux / Zustand (internally use this or similar)
* custom event emitter store
* browser APIs like `matchMedia`, `navigator.onLine` (when treated as a store)

Interview one-liner:

> “`useSyncExternalStore` is the concurrency-safe way to read and subscribe to external state in React.”

---

## Why it exists (the problem)

Before React 18, you might do:

* subscribe in `useEffect`
* keep store value in `useState`

In concurrent rendering, React may render, pause, restart — and your subscription pattern can cause:

* **tearing** (UI reads inconsistent store values across components in the same render)
* hydration inconsistencies with SSR

`useSyncExternalStore` ensures React reads the store in a consistent way and re-renders when the snapshot changes.

---

## The API shape

```js
useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)
```

* `subscribe(listener)` → register listener, return unsubscribe
* `getSnapshot()` → read current store value
* `getServerSnapshot()` → value used during SSR (optional but recommended for SSR)

---

## Simple example: online/offline store

```jsx
import { useSyncExternalStore } from "react";

function subscribe(callback) {
  window.addEventListener("online", callback);
  window.addEventListener("offline", callback);
  return () => {
    window.removeEventListener("online", callback);
    window.removeEventListener("offline", callback);
  };
}

function getSnapshot() {
  return navigator.onLine;
}

export function useOnlineStatus() {
  return useSyncExternalStore(subscribe, getSnapshot, () => true);
}

function Status() {
  const online = useOnlineStatus();
  return <p>{online ? "Online" : "Offline"}</p>;
}
```

---

## Common usage pattern: custom store

If you have a store object:

* `store.subscribe(fn)`
* `store.getState()`

```jsx
import { useSyncExternalStore } from "react";

function useStore(store) {
  return useSyncExternalStore(
    store.subscribe,
    store.getState,
    store.getState // SSR: same snapshot if store exists server-side
  );
}
```

---

## Selectors (important)

In practice, you often don’t want the whole store — you want a slice.

```jsx
function useStoreSelector(store, selector) {
  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState())
  );
}
```

⚠️ Best practice: selector should return a stable value when unchanged (or you’ll re-render too much).

---

## Advantages

* Correct with **concurrent rendering** (prevents tearing)
* Works well with **SSR/hydration**
* Standard way for libraries to integrate external state with React

---

## Disadvantages / when not needed

* More boilerplate than `useState`/`useEffect`
* Mostly needed for true external stores; for normal local state just use `useState`/`useReducer`

---

## 30-second interview answer

`useSyncExternalStore` is React 18’s official hook for subscribing to external stores in a way that’s safe for concurrent rendering and SSR. You provide a subscribe function and a snapshot reader, and React ensures consistent reads and re-renders when the snapshot changes, preventing issues like tearing that older subscription patterns could cause.

---

