# React Fiber (interview + “how it works”)

“Fiber is React’s incremental rendering engine. 
It breaks rendering into units of work so React can schedule updates, pause work, 
and keep the UI responsive—especially with Concurrent Rendering in React 18.”

# why Fiber was introduced (the problem it solved)

Before Fiber, rendering was more “all at once” for a subtree. For big trees, that could block the main thread → jank (typing/scroll lag).

Fiber enables:
Incremental rendering (do work in chunks)
Prioritization (urgent vs non-urgent updates)
Interruptible rendering (pause a render to handle input)
Better foundation for Suspense and Concurrent features

# Fiber vs Reconciliation

Reconciliation = “figure out what changed”
Fiber = “how React organizes and schedules the work of figuring it out”

