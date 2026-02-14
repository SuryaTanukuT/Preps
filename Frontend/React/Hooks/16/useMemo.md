# useMemo in React (interview-ready)
useMemo memoizes a computed value. It re-computes the value only when its dependency array changes.

useMemo memoizes the result of a computation so React doesn’t recompute it on every render. 
It’s useful for expensive derived values like filtering/sorting big lists, and for keeping object/array references stable when passing props to memoized children. It’s not a default pattern because it adds overhead and wrong dependencies can cause stale values, so I apply it selectively after profiling.

const value = useMemo(() => computeExpensive(x, y), [x, y]);

Interview one-liner:
“useMemo caches the result of a calculation between renders based on dependencies.”

# Why we use it
React re-renders can happen often. If you do an expensive calculation during render, it may repeat unnecessarily.

useMemo helps to:
avoid expensive recomputation
keep stable object/array references so memoized children (React.memo) don’t re-render

# Advantages
Can significantly reduce CPU work on re-renders
Helps prevent unnecessary child re-renders by stabilizing references
Makes render predictable for heavy computations

# Disadvantages / pitfalls
Memoization has overhead (React still tracks deps + stores values)
Overusing it can reduce readability
Wrong deps cause stale values (bugs)
Often unnecessary for cheap calculations

Interview line:
“I use useMemo only when profiling shows expensive recalculation or when stable references are needed.”

# Dependency array rules (critical)
Include every value used inside the memo callback that can change.
useMemo uses reference equality on deps.
ESLint react-hooks/exhaustive-deps helps avoid mistakes.

Bad (stale bug):
const total = useMemo(() => price * qty, [price]); // ❌ missing qty

Correct:
const total = useMemo(() => price * qty, [price, qty]);


# Best practices
✅ Use it for:
heavy calculations
sorting/filtering large arrays
stable props for memoized children
expensive derived data

⚠️ Avoid it for:
tiny computations (a + b, simple map)
“just in case” optimization

✅ Always:
measure using React Profiler
keep deps correct
prefer simpler fixes first (split components, move state down, virtualization for huge lists)

