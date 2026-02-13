# JSX
JSX is basically syntax sugar for creating React elements. Here’s a solid “interview + real understanding” overview.

JSX is a syntax extension that lets us write UI declaratively. It gets compiled into React element creation calls—plain JavaScript objects describing the UI, not HTML. React uses these elements to reconcile changes efficiently. JSX supports embedding expressions with {}, component composition via children, and lists using stable keys for reconciliation.

# What JSX is
JSX (JavaScript XML) lets you write UI like HTML inside JS:const el = <h1 className="title">Hello</h1>;
But browsers don’t understand JSX. It must be compiled (by Babel/TS/Next/Vite) into JS.

# What JSX becomes (important)
JSX compiles to React element creation:Classic runtime (older)<h1>Hello</h1>

 compiles toReact.createElement("h1", null, "Hello");Automatic runtime (modern React 17+ setup)JSX compiles to calls from react/jsx-runtime:
jsx("h1", { children: "Hello" });
That’s why you often don’t need import React from "react" anymore.
# React Elements (what JSX creates)JSX produces React elements (plain objects) describing UI:type: "div" or your component functionprops: attributes + childrenkey: used for listsref: for refs (special)
React then uses these elements to build the Fiber tree and update the DOM.