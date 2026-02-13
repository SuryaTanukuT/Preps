# React Optimization (high-level interview perspective)

React optimization = reduce unnecessary renders, reduce expensive work during render, and reduce DOM cost. In interviews, speak in this order:

Measure → fix biggest re-render sources → memoize carefully → optimize data fetching & bundle → optimize DOM (virtualize).

I optimize React by profiling first, then reducing unnecessary re-renders through component splitting, stable props, and React.memo for expensive components. For expensive computations I use useMemo, and for heavy UI updates during typing or navigation I use startTransition or useDeferredValue so urgent interactions stay responsive. For large lists I use virtualization to reduce DOM nodes. Finally, I improve perceived performance with caching/prefetching data and route-level code splitting plus asset optimization.

# The golden rule: optimize the right thing
What to measure
React DevTools Profiler: which components re-render & how long
Browser Performance tab: scripting/layout/paint
Web vitals: LCP / INP / CLS
