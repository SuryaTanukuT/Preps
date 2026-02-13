# Portals in React (interview-ready)

A Portal lets you render a React component’s UI into a different DOM node outside its parent DOM hierarchy, 
while still keeping it in the same React component tree.

Portals allow rendering children into a DOM node outside the parent hierarchy—commonly for modals, tooltips, and overlays to avoid z-index and overflow issues. Even though the DOM placement changes, the components remain in the same React tree, so props/state/context work normally and React synthetic events bubble through the React tree. For production modals, I also handle focus trapping, escape-to-close, scroll lock, and proper ARIA attributes.
