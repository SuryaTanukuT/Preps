# ReactDOM
ReactDOM is the package that connects React (UI logic) to the browser DOM.

It is responsible for:
mounting your app into a DOM node (createRoot(...).render(...))
applying UI updates to the DOM during the commit phase
handling hydration (SSR), portals, and React’s event system (SyntheticEvents)

Interview line:
“React builds the UI description; ReactDOM renders and commits it to the browser DOM.”

# Real DOM
Real DOM is the actual browser DOM tree (document, real <div>, <input>, etc.).
Updating it triggers browser work:
style recalculation
layout (reflow)
paint / composite
which can be expensive if done too often.