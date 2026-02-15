# Life Cycle
React first renders your component to compute UI, then commits DOM updates and runs side effects. In classes, side effects go into componentDidMount/DidUpdate with cleanup in componentWillUnmount. 
With hooks, useEffect expresses the same lifecycle: it runs after commit, re-runs when dependencies change, and cleanup runs before re-run and on unmount. The dependency array defines what values the effect depends on, preventing stale closures and unnecessary work. Hooks improve composition and reduce boilerplate, but you must manage deps and cleanup carefully.
# Life Cycle Hooks Based:
Hooks “lifecycle” mapping (interview-friendly)
Hooks don’t have explicit lifecycle methods. You express lifecycle via effects.# Mount
function runs → React renders UI
after commit: useEffect(() => { ... }, []) runs once (mount effect)
# Update
after commit: useEffect(() => { ... }, [deps]) runs when deps changed
# Unmount
cleanup function runs:
useEffect(() => {  // subscribe  return () => {    // cleanup unsubscribe  };}, []);
Layout timing (important)
useLayoutEffect runs after DOM mutations but before paintuseEffect runs after paint (non-blocking)
Interview line:“useLayoutEffect is for measuring/layout sync; useEffect is for most side effects.”
# Class component lifecycle (interview flow)Mounting (first time)constructor(props)
initialize state, bind handlers (rare now)
static getDerivedStateFromProps() (rare)
render()componentDidMount() 
run side effects (fetch, subscriptions)
Updating (re-render)
static getDerivedStateFromProps() (rare)
shouldComponentUpdate() (perf gate)
render()
getSnapshotBeforeUpdate() (rare, read DOM before commit)
componentDidUpdate(prevProps, prevState) 
run side effects based on changes
Unmounting
componentWillUnmount() 
cleanup subscriptions/timers
Error lifecycle
static getDerivedStateFromError()
componentDidCatch()
