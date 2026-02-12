A React component is a reusable UI building block.
It takes inputs (props) and returns UI (React elements / JSX).

Types of React Components::

1) Function Component (most used)
function Greeting({ name }) {
  return <h1>Hello, {name}</h1>;
}

2) Arrow Function Component
const Greeting = ({ name }) => <h1>Hello, {name}</h1>;

3) Class Component (older, still seen)
import React from "react";

class Greeting extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}

Key parts of a component::
Props (inputs from parent)
function Button({ label }) {
  return <button>{label}</button>;
}

State (internal data that changes UI)
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}

Component categories (common interview wording)::
Presentational (UI only): mostly JSX, gets data via props
Container (logic/data): fetches data, handles state, passes props down
Controlled component: form input controlled by React state
Uncontrolled component: form input controlled by DOM (ref)

Component rules (important)::
Component name must start with Capital letter: MyCard, not mycard
Must return JSX or null
Must be a pure render (don’t do API calls directly inside render; use effects)