# Refs in React (high-level interview perspective)
Refs in React provide a stable object whose .current can point to a DOM element or any mutable value. They persist across renders but changing a ref doesn’t trigger re-render, so they’re used for imperative tasks like focus, scroll, measurements, integrating non-React libraries, or storing timer IDs/previous values. For reusable components we use forwardRef, and useImperativeHandle to expose a clean imperative API.
Refs give you a way to hold a persistent reference to:a DOM element, ora mutable value that survives re-renders without causing a re-render.
Interview one-liner:“Refs are for imperative access and for storing mutable values that shouldn’t trigger renders.”
# Types of refsA) DOM refs (most common)Used when you need to do something imperative like focus, scroll, measure.
function Search() {  const inputRef = useRef(null);
  return (    <>      <input ref={inputRef} />      <button onClick={() => inputRef.current?.focus()}>        Focus      </button>    </>  );}
B) Mutable value refs (not DOM)
Store values across renders without re-rendering:timer idprevious props/state“isMounted” flag (careful)cached values for event handlers
const timerRef = useRef(null);timerRef.current = setTimeout(...);
2) useRef vs createRefuseRef() (function components) → same ref object for entire component lifecreateRef() (class components) → commonly used in classes; in functions it creates a new ref each render (not ideal)
Interview line:“In function components use useRef; in class components use createRef.
3) .current behavior (important)Ref object shape: { current: ... }Updating ref.current does NOT trigger a re-renderReact sets ref.current after the element mounts/updates
4) When to use refs (good use cases) Focus / text selection Scroll to element Measure element size (getBoundingClientRect) Integrate with non-React libraries (charts, maps) Store mutable values like timers / previous value
5) When NOT to use refs Don’t use refs as “state” for UIIf UI should update when a value changes → use stateRefs won’t re-render, so UI can get out of sync
Interview line:“State is for rendering; refs are for imperative escapes.”
6) Forward refs (interview topic)By default, refs don’t pass into custom components.To expose a child DOM node to parent, use forwardRef:
const Input = React.forwardRef(function Input(props, ref) {  return <input ref={ref} {...props} />;});

Use cases:reusable input componentsdesign systems
7) Exposing custom methods: useImperativeHandleWhen you want parent to call a controlled API (not raw DOM):
const FancyInput = React.forwardRef((props, ref) => {  const innerRef = useRef(null);
  useImperativeHandle(ref, () => ({    focus: () => innerRef.current?.focus(),  }));
  return <input ref={innerRef} />;});

Interview line:“useImperativeHandle lets you expose a minimal imperative API.”