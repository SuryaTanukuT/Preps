# `useDebugValue` in React

`useDebugValue` is a hook used only for debugging.
It lets you display a custom label/value for a hook in React DevTools, so it’s easier to understand what that hook is doing.

`useDebugValue` is used in custom hooks to provide readable labels in React DevTools. It helps debugging by showing hook state like “loading/success” or “Online/Offline” without affecting UI logic.

Mostly used inside custom hooks.

Interview one-liner:
“useDebugValue improves DevTools visibility for custom hooks; it doesn’t affect UI behavior.”

---

# Where you use it

Inside a custom hook to show:

* Current status (loading/error/success)
* Derived values
* Selected user/theme/locale
* Connection state (websocket connected/disconnected)

---

# Basic example

```js
import { useDebugValue, useState } from "react";

function useOnlineStatus() {
  const [online, setOnline] = useState(navigator.onLine);

  useDebugValue(online ? "Online" : "Offline");

  return online;
}
```

In React DevTools, when you inspect a component using `useOnlineStatus()`, you’ll see:

Online or Offline

---

# Formatting with a function (performance-friendly)

If formatting is expensive, pass a formatter function. React will call it only when DevTools needs it.

```js
useDebugValue(user, (u) => (u ? `User: ${u.name}` : "No user"));
```

Why this matters:

* Avoids doing string formatting on every render unnecessarily

---

# When NOT to use it

* Inside normal components (not very useful)
* For business logic (it’s not logging)
* When values are already obvious
