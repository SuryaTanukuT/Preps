# Pure Components in React (high-level interview perspective)

A Pure Component is a component that skips re-rendering when its inputs haven’t changed.
React decides this by doing a shallow comparison of:

props
and (for class pure components) state
If nothing changed (by shallow equality), React reuses the previous render output.

Pure components avoid unnecessary re-renders by shallowly comparing props (and state for PureComponent). In classes we use React.PureComponent; in function components we use React.memo. They’re useful when expensive children receive the same props often, but they rely on stable references—inline objects/functions can defeat memoization—so I apply them selectively after profiling.