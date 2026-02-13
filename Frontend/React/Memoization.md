# React Memoization (interview-ready)
Memoization in React = reusing previous results so you don’t redo work when inputs didn’t change. It’s a performance tool, not a default pattern.

1) What can be memoized in React?A) Component render output → React.memoPrevents a component from re-rendering when props are the same (shallow compare).
const Row = React.memo(function Row({ item, onSelect }) {  return <div onClick={() => onSelect(item.id)}>{item.name}</div>;});
B) Expensive calculation result → useMemoCaches the value of an expensive computation until dependencies change.
const filtered = useMemo(() => heavyFilter(list, query), [list, query]);
Use when:calculation is expensivedependencies change less frequently than renders
C) Function identity → useCallbackCaches a function reference so it doesn’t change unless dependencies change.
const onSelect = useCallback((id) => {  setSelectedId(id);}, []);

Why it matters:prevents child re-render when child is memoized (React.memo)prevents re-running effects that depend on a callback
Key interview truth:“useCallback(fn, deps) is basically useMemo(() => fn, deps).”

2) The #1 concept: Referential equality
Most memoization benefits come down to reference stability:Objects/arrays/functions created inline become new references every render
That breaks shallow comparisons and triggers re-rendersBad (breaks React.memo):
<Row item={item} onSelect={(id) => doSomething(id)} />

Better:const onSelect = useCallback((id) => doSomething(id), [doSomething]);<Row item={item} onSelect={onSelect} />