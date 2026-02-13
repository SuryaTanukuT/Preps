# Concurrent Rendering::

Concurrent rendering in React 18 means React can interrupt and prioritize rendering work. 
It can start rendering a new UI, pause it to handle urgent updates like typing, and continue later—keeping the app responsive. 
We mark non-urgent updates using startTransition/useTransition, or defer expensive derived UI with useDeferredValue. 
This works well with Suspense so React can avoid blocking the whole screen. 
Because renders can restart, render must be pure and side effects should live in effects, not in render.

concurrent rendering is a rendering capability where React can:
start rendering,
pause it,
continue later,

or throw it away and retry,
so the UI stays responsive under heavy work.

# Must-know concepts (interview checklist)

1) Rendering vs Committing (two phases)
Render phase: React prepares the next UI (can be interrupted in concurrent mode).
Commit phase: React applies changes to the DOM (must be fast, not interruptible).

Interview angle:
 Why side effects must not run during render phase (because render can restart).

2) Interruptible rendering (the core superpower)
In concurrent mode, React can stop rendering a big tree mid-way to handle urgent updates.

What to say:
“React can yield to the browser, so typing doesn’t lag.”
“It can discard work if new state comes in.”

3) Update priority: urgent vs non-urgent
React categorizes work by priority:
Urgent: input typing, clicks, scrolling feedback
Non-urgent: filtering large lists, expensive transitions, route changes

Tool you must know:
startTransition(() => setState(...))
useTransition() (gives isPending)

Interview question: When should you use startTransition?
Answer: when the update is not required immediately and can be delayed to keep UI responsive.


4) useDeferredValue vs useTransition
These get asked a lot.
# useTransition::
You control which state update is low priority
Good for: search/filter results, tab switching, route transitions

# useDeferredValue::
You keep input state urgent, but defer a derived value
Good for: expensive rendering based on fast-changing value (like search input)

One-liner difference:
useTransition: “defer the update”
useDeferredValue: “defer the value”

5) Automatic batching (React 18 change)

React 18 batches state updates automatically in more cases (timeouts, promises, native events).

Interview angle:

“This reduces re-renders and improves performance.”

If asked how to opt out: flushSync (rare use).