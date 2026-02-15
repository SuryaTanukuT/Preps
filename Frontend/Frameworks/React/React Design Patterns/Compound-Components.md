## Compound Components in React (interview-ready)

### What it is

**Compound Components** is a design pattern where multiple components work together as a single “component API”, sharing state implicitly—usually via **Context**.

Example API:

```jsx
<Tabs>
  <Tabs.List>
    <Tabs.Trigger value="a">A</Tabs.Trigger>
    <Tabs.Trigger value="b">B</Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="a">Panel A</Tabs.Content>
  <Tabs.Content value="b">Panel B</Tabs.Content>
</Tabs>
```

Interview one-liner:

> “Compound components let consumers compose UI freely while the parent manages shared state, often via Context.”

---

## Why it’s used

✅ Great DX (developer experience): clean, expressive API
✅ Flexible layout: consumer controls markup order and structure
✅ Encapsulates logic: state + interactions live inside the component system
✅ Avoids prop drilling (child parts don’t need many props)

Common places:

* Tabs
* Accordion
* Modal/Dialog
* Dropdown/Menu
* Form components (Field, Label, Error)

---

## How it works (core mechanism)

1. Parent component creates shared state (e.g., active tab)
2. Parent provides that state via **Context**
3. Child “subcomponents” read/write state from context

---

## Full example: Tabs (compound components)

```jsx
import React, { createContext, useContext, useMemo, useState } from "react";

const TabsContext = createContext(null);

function useTabsContext() {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error("Tabs components must be used inside <Tabs>");
  return ctx;
}

export function Tabs({ defaultValue, value, onValueChange, children }) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = value !== undefined;

  const active = isControlled ? value : internal;

  const setActive = (next) => {
    onValueChange?.(next);
    if (!isControlled) setInternal(next);
  };

  const ctx = useMemo(() => ({ active, setActive }), [active]);

  return <TabsContext.Provider value={ctx}>{children}</TabsContext.Provider>;
}

Tabs.List = function TabsList({ children }) {
  return <div role="tablist">{children}</div>;
};

Tabs.Trigger = function TabsTrigger({ value, children }) {
  const { active, setActive } = useTabsContext();

  return (
    <button
      role="tab"
      aria-selected={active === value}
      onClick={() => setActive(value)}
    >
      {children}
    </button>
  );
};

Tabs.Content = function TabsContent({ value, children }) {
  const { active } = useTabsContext();
  if (active !== value) return null;

  return <div role="tabpanel">{children}</div>;
};
```

### Usage

```jsx
<Tabs defaultValue="a">
  <Tabs.List>
    <Tabs.Trigger value="a">Tab A</Tabs.Trigger>
    <Tabs.Trigger value="b">Tab B</Tabs.Trigger>
  </Tabs.List>

  <Tabs.Content value="a">Content A</Tabs.Content>
  <Tabs.Content value="b">Content B</Tabs.Content>
</Tabs>
```

---

## Controlled vs Uncontrolled (important interview point)

Compound components often support both:

* **Uncontrolled:** parent manages internal state (`defaultValue`)
* **Controlled:** consumer passes `value` + `onValueChange`

This is common in UI libraries.

---

## Advantages

* Very clean API for users of the component
* Flexible composition (consumer decides layout)
* Encapsulation of complex interaction logic
* Avoids prop drilling

---

## Disadvantages / pitfalls

* Context updates can cause many re-renders if not designed well
* Harder to type in TypeScript unless carefully structured
* More internal complexity than simple “single component” APIs

---

## Best practices

* Provide a “safe” internal hook (like `useTabsContext`) that throws if used outside provider
* Memoize provider value (`useMemo`) to avoid extra re-renders
* Split context if necessary (state vs actions) for performance
* Add accessibility roles/ARIA for components like Tabs/Menu/Modal

---

## 30-second interview answer

Compound components are a pattern where a parent component coordinates multiple child components that form one cohesive UI API. The parent manages shared state and exposes subcomponents that consume that state via Context, enabling flexible composition without prop drilling. It’s commonly used in Tabs, Accordions, Menus, and Modals, and often supports both controlled and uncontrolled usage.
