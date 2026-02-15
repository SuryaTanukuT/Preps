## `useClickOutside` in React (custom hook)

`useClickOutside` is a common custom hook used to detect when a user clicks **outside** a target element (dropdown, modal, tooltip) and then close it.

---

## Why it’s used

* Close dropdown when clicking anywhere else
* Close popover/tooltip on outside click
* Dismiss context menu
* Common UX pattern

---

## Best practice approach

Use:

* a `ref` to the element you want to protect
* a document-level listener
* capture phase (often more reliable)
* ignore clicks inside the element

### ✅ Hook

```jsx
import { useEffect } from "react";

export function useClickOutside(ref, onOutsideClick, enabled = true) {
  useEffect(() => {
    if (!enabled) return;

    function handler(e) {
      const el = ref.current;
      if (!el) return;

      // if click is inside, ignore
      if (el.contains(e.target)) return;

      onOutsideClick(e);
    }

    // capture phase helps with portals / stopPropagation cases
    document.addEventListener("pointerdown", handler, true);

    return () => {
      document.removeEventListener("pointerdown", handler, true);
    };
  }, [ref, onOutsideClick, enabled]);
}
```

### ✅ Usage

```jsx
import { useRef, useState, useCallback } from "react";
import { useClickOutside } from "./useClickOutside";

function Dropdown() {
  const [open, setOpen] = useState(false);
  const ref = useRef(null);

  const close = useCallback(() => setOpen(false), []);
  useClickOutside(ref, close, open);

  return (
    <div>
      <button onClick={() => setOpen(o => !o)}>Toggle</button>

      {open && (
        <div ref={ref} style={{ border: "1px solid #ccc", padding: 8 }}>
          Dropdown content
        </div>
      )}
    </div>
  );
}
```

---

## Important interview points / edge cases

### 1) Why `pointerdown` over `click`?

* closes earlier (better UX)
* works across mouse/touch/pen
* avoids some focus timing issues

### 2) Why capture phase (`true`)?

* If inner elements call `stopPropagation`, bubble listeners won’t run
* More reliable for overlays / portals

### 3) Portals

This pattern still works because `document` receives the pointer event; the hook checks `ref.current.contains(e.target)`.

### 4) Multiple refs

If you need “outside of both button + dropdown panel”, pass an array of refs and treat click as inside if any contains the target.

---

## Quick multi-ref version (common)

```js
export function useClickOutsideMany(refs, onOutsideClick, enabled = true) {
  useEffect(() => {
    if (!enabled) return;

    const handler = (e) => {
      const inside = refs.some(r => r.current && r.current.contains(e.target));
      if (!inside) onOutsideClick(e);
    };

    document.addEventListener("pointerdown", handler, true);
    return () => document.removeEventListener("pointerdown", handler, true);
  }, [refs, onOutsideClick, enabled]);
}
```

---

## 20–30 sec interview answer

`useClickOutside` is a custom hook that attaches a document-level pointer listener, checks whether the click target is outside a referenced element using `element.contains`, and triggers a callback to close dropdowns/modals. Using `pointerdown` and capture phase makes it more reliable, even with portals or stopPropagation.

---
