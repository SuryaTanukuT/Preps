# Rendering elements in React (interview-friendly)
1) What is a “React element”?
A React element is a plain JavaScript object that describes what you want to see on the screen.

JSX:
<h1>Hello</h1>

Rendering elements in React means converting JSX into React element objects, running components to produce an element tree, then reconciling it with the previous tree to compute minimal changes. ReactDOM commits those changes to the real DOM. Elements are immutable descriptions; React re-creates elements on each render and updates only what changed.

creates a React element (not HTML, not a DOM node).

Interview line:
“Elements are immutable UI descriptions.”


# Elements vs Components (important)

Element: the output (the UI description object)
Component: a function/class that returns elements

Example:
function Title() {
  return <h1>Hello</h1>; // element
}

# What does “rendering an element” mean?

Rendering means React:
takes your element tree (created from JSX/components),
builds/updates its internal tree (Fiber),
and then commits changes to the real DOM via ReactDOM.

So the flow is:
JSX → React elements → reconciliation → DOM updates

# Initial render (mount)
When you do:
createRoot(rootEl).render(<App />);


React:
creates elements from <App />
renders components to produce a full element tree
mounts corresponding DOM nodes

# Re-render (update)

When state/props change:
React re-runs affected components to produce a new element tree
it diffs (reconciles) old vs new
applies minimal DOM mutations

Interview line:
“A re-render doesn’t mean React updates the DOM everywhere—only where the diff shows changes.”

# Elements are immutable

React elements are not updated in-place.
Instead, every render creates new elements, and React decides what DOM updates are needed.


# List rendering is “rendering elements from an array”
{items.map(item => <Row key={item.id} item={item} />)}


This creates an array of elements, which React renders as siblings.
Keys help React match element identity during updates.


