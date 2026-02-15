# Error Boundaries in React (interview-ready)

Error Boundaries are React components that catch JavaScript errors during rendering and show a fallback UI instead of crashing the whole app.

Error boundaries are class components that catch errors during rendering and lifecycle methods in their child tree and render a fallback UI instead of unmounting the whole app. They don’t catch errors from event handlers or async code, which I handle separately with try/catch or query error states. I usually place them around routes and major feature areas, and provide a reset or remount mechanism for recovery plus logging via componentDidCatch.

One-liner:
“Error boundaries catch render-time errors in a component subtree and let you render a fallback UI.”

# 1) What they catch (and what they don’t)
✅ They catch errors in:
rendering (JSX/render logic)
lifecycle methods (class components)
constructors of child components

❌ They do NOT catch errors in:
event handlers (onClick, onChange)
async code (setTimeout, promises, fetch, async/await)
errors thrown inside the error boundary itself
(in SSR) server rendering errors are handled differently

# How to handle those?
Use try/catch in event handlers / async functions
Set error state / show toast
Use React Query error states for data fetching

# 2) How to implement an Error Boundary
In React today, error boundaries must be class components (React hasn’t shipped a hook-based equivalent).

Key lifecycle APIs:
static getDerivedStateFromError(error) → update state to show fallback
componentDidCatch(error, info) → log/report error

# Common interview questions + answers
Q: Why don’t error boundaries catch event handler errors?
Because event handlers run outside render/lifecycle. You handle them with try/catch and local error UI.

Q: Should we wrap every component in an error boundary?
No—too noisy. Wrap logical boundaries (routes/features) where fallback UI makes sense.

Q: How do you log errors?
Use componentDidCatch to send errors to monitoring (Sentry, Datadog, etc.).

