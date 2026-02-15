## Single Source of Truth (React)

### Meaning

**Single Source of Truth (SSOT)** means for any piece of data in your app, there should be **one authoritative place** where it lives, and everything else should **derive from it**, not duplicate it.

Interview one-liner:

> “Keep one authoritative state for a value; derive everything else to avoid inconsistencies.”

---

## Why it matters

If you store the same information in multiple places, it’s easy for them to get out of sync → bugs.

Example of bug:

* `selectedUserId` stored in Parent
* also stored in Child
* updates happen differently → UI mismatch

SSOT prevents:

* inconsistent UI
* complicated syncing logic
* hard-to-debug state issues

---

## Example: avoid duplicated derived state

### ❌ Bad (duplicate state)

```jsx
const [items, setItems] = useState([]);
const [count, setCount] = useState(0);

useEffect(() => {
  setCount(items.length);
}, [items]);
```

### ✅ Good (derive it)

```jsx
const [items, setItems] = useState([]);
const count = items.length;
```

Here, `items` is the single source of truth; `count` is derived.

---

## SSOT with forms (common)

### ✅ Controlled input (React state is SSOT)

```jsx
const [email, setEmail] = useState("");
<input value={email} onChange={(e) => setEmail(e.target.value)} />;
```

### ✅ Uncontrolled form (DOM is SSOT)

```jsx
<input defaultValue="x" ref={ref} />; // DOM holds the value
```

Pick one approach; don’t mix for the same field.

---

## SSOT across components (lifting state up)

If two components need the same data:

* move state to the **nearest common parent**
* pass it down via props

That parent becomes the SSOT for that shared state.

---

## SSOT for server state (important distinction)

For API data:

* avoid copying server data into many places
* use one cache/store as SSOT:

  * RTK Query cache
  * React Query cache
  * Redux store (if you manage server state there)

---

## Best practices

* Store the **minimal state** needed
* Prefer **derived values** over extra state
* Lift state only as high as required
* Avoid syncing state between components—share state instead

---

## 30-second interview answer

Single source of truth means keeping each piece of state in exactly one place and deriving all dependent values from it. This avoids out-of-sync bugs and simplifies reasoning. In React, we achieve it by avoiding duplicated derived state, lifting shared state to a common parent, and using a single cache/store for server state like RTK Query or React Query.
