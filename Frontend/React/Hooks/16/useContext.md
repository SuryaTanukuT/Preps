# `useContext` in React (high-level interview perspective)

`useContext` is a React Hook used to read data from a Context inside a function component, without passing props through every level.

One-liner:
“useContext lets components consume shared values (like theme/auth) directly from a Context Provider.”

[https://www.freecodecamp.org/news/react-context-api-tutorial-examples/](https://www.freecodecamp.org/news/react-context-api-tutorial-examples/)

---

# Why we use it

It solves props drilling for values needed by many components, such as:

* Theme (dark/light)
* Auth user + token
* Language (i18n)
* Permissions / roles
* Feature flags

---

# `useContext` vs Redux / Zustand (quick)

**useContext:**

* Great for simple global values
* Low ceremony
* Built-in to React

**Redux / Zustand:**

Better when you need:

* Complex updates
* Selectors (to avoid unnecessary re-renders)
* Debugging tools / middleware
* Large shared state

---

`useContext` reads values from React Context, allowing shared data like theme or auth to be accessed deeply without props drilling. A Provider supplies the value and any descendant can consume it. When the context value changes, all consumers re-render, so I keep context focused, split contexts, and memoize provider values. For large frequently changing state, I prefer a store with selectors.
