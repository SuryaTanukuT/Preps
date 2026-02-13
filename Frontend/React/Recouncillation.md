# Reconciliation in React (interview perspective)
Reconciliation is React’s process of comparing the previous render tree with the next render tree to compute the minimal updates. 
React uses heuristics: if element types match it reuses nodes; 
if types change it replaces subtrees; and for lists it uses keys to match children and preserve state.
Under the hood this happens on the Fiber tree, which also enables scheduling and concurrent rendering.

Reconciliation is the process where React:
builds a new virtual UI tree (from your render output), then
compares it with the previous tree, and
decides the minimum set of changes needed to update the UI (DOM for ReactDOM).

One-liner:
“Reconciliation is React’s diffing process to figure out what changed between renders.”

# The two phases involved
1) Render phase (compute next UI)
React calls your components to produce the next element tree.
This is “pure computation” (should have no side effects).

2) Commit phase (apply changes)
ReactDOM applies DOM mutations: insert/update/remove nodes
Then effects run (useLayoutEffect then useEffect)

Reconciliation mostly happens between render and commit: “what do we need to change?”
