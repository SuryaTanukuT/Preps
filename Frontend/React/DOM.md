# DOM::

React builds a virtual tree from components during the render phase, 
reconciles it against the previous tree, 
and in the commit phase ReactDOM applies the minimal DOM mutations and then runs effects. 
Internally it uses Fiber to represent the tree and schedule work, 
enabling interruptible concurrent rendering and update prioritization. 

ReactDOM also provides event delegation with SyntheticEvent, hydration for SSR, and portals.


# React (core)
React is the UI engine:
You write components (functions/classes) that return a description of UI (JSX → React elements).

React manages:
state updates
re-rendering
reconciliation (diffing)
hooks
scheduling/priorities (React 18 concurrent features)

# ReactDOM
ReactDOM is the renderer for the web:
Takes React’s “UI description” and applies it to the browser DOM

Also handles:
event system (SyntheticEvent + delegation)
hydration (SSR)
portals
createRoot (React 18)
