# `useImperativeHandle` in React (interview-ready)

`useImperativeHandle` lets a component control what a parent gets when it uses `ref` on that component.

Instead of exposing the raw DOM node (or everything), you expose a clean imperative API like:

* `focus()`
* `scrollToTop()`
* `reset()`

✅ Used with `forwardRef`.

Interview one-liner:
“useImperativeHandle customizes the ref value exposed to the parent—useful for exposing imperative methods like focus/reset.”

---

# Why it’s used

Normally:

* `ref` on a DOM element gives you the DOM node
* `ref` on a custom component doesn’t work unless you forward it

With `forwardRef` + `useImperativeHandle`, you can:

* Keep encapsulation (don’t expose inner DOM structure)
* Expose only what parent needs (safe API)
* Integrate with form libraries or complex widgets

---

# Where it’s used (common use cases)

* Reusable Input component → `focus()`, `clear()`
* Modal / Dialog → `open()`, `close()`
* DataGrid / Table → `scrollToRow()`, `refresh()`
* Rich text editor wrapper → `getContent()`, `setContent()`
* Stepper / Wizard → `next()`, `prev()`, `reset()`

---

`useImperativeHandle` is used with `forwardRef` to customize what a parent sees through a ref.
Instead of exposing internal DOM nodes, we expose a small imperative API like `focus()` or `reset()`.

It’s useful for reusable inputs, modals, and complex widgets, but I use it sparingly and prefer declarative props/state when possible.
