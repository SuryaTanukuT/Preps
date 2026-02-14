# `useEffect` in React (high-level interview perspective)

`useEffect` lets you run side effects after React updates the UI (after the commit).
Side effects = anything outside pure rendering.

`useEffect` runs side effects after React commits updates to the DOM. It can run on mount, on dependency changes, and cleanup runs before re-running and on unmount.

I use it for external synchronization like fetching, subscriptions, and timers, and always clean up listeners/timers to avoid leaks and race conditions.

[https://feature-sliced.design/blog/mastering-react-useeffect](https://feature-sliced.design/blog/mastering-react-useeffect)

Examples of side effects:

* API calls / data fetching
* Subscriptions (WebSocket, events)
* Timers
* Logging
* Updating `document.title`

Interview one-liner:
“useEffect synchronizes your component with external systems after render.”

---

# Data fetching with `useEffect` (correct pattern)

Key points:

* Don’t fetch in render
* Handle loading/error
* Cancel stale requests (race condition)

```js
useEffect(() => {
  const controller = new AbortController();

  (async () => {
    const res = await fetch(`/api/users/${id}`, {
      signal: controller.signal,
    });
    // handle res...
  })();

  return () => controller.abort();
}, [id]);
```

---

# Common pitfalls (interview favorites)

## Pitfall 1: Missing deps → stale closure bug

```js
useEffect(() => {
  console.log(count);
}, []); // count is stale if it changes
```

---

## Pitfall 2: Infinite loop

```js
useEffect(() => {
  setX(x + 1);
}, [x]); // can loop if not guarded
```

---

## Pitfall 3: Putting derived values in state

Often causes extra effects and loops. Prefer derived computation.

---

# `useEffect` vs `useLayoutEffect` (quick)

`useEffect`:

* Runs after paint (preferred for most side effects)

`useLayoutEffect`:

* Runs before paint after DOM mutations (use for layout measuring)

Interview line:
“useLayoutEffect is for measuring DOM; useEffect is for everything else.”
