# Batching in React (interview perspective)
Batching = React groups multiple state updates into a single re-render for performance.Instead of re-rendering after every setState, React waits and renders once.
Why batching mattersFewer renders → faster UIConsistent state updates inside the same “event”
Interview one-liner:“Batching reduces unnecessary renders by combining multiple updates into one commit.”

# React 17 vs React 18 batching (key difference)# React 17 and earlierBatching happened mostly inside React event handlers only.
Example (React 17):
setA(1);setB(2);// usually 1 render (inside onClick)

But in async code (timeouts/promises), React 17 often rendered twice:
setTimeout(() => {  setA(1); // render #1  setB(2); // render #2}, 0);
# React 18 (with createRoot)React 18 introduced automatic batching:React batches updates in more places:
setTimeoutPromise.thenasync/await callbacksnative event listeners (often)many other async boundaries
So the same timeout example becomes 1 render in React 18 (createRoot).
Interview line:“React 18 automatically batches updates across async boundaries when using createRoot.”
# What batching is notIt does not merge your state objects magically.It just controls when React re-renders.