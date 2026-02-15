## Composition over Inheritance (React)

### Meaning

**Composition over inheritance** means you build features by **combining components** (props, children, wrappers) rather than creating deep class hierarchies.

Interview one-liner:

> “In React, we reuse behavior by composing components, not by inheriting from base components.”

---

## Why React prefers composition

Inheritance creates:

* tight coupling (child depends on parent implementation)
* rigid structures (hard to change)
* “diamond problem” / confusing overrides (classic OOP issues)

Composition gives:

* flexible UI building blocks
* easy reuse without hidden behavior
* clear data flow through props/children

---

## 1) Composition with `children`

### Wrapper component

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

<Card>
  <h3>Title</h3>
  <p>Content</p>
</Card>
```

---

## 2) Specialization by configuration (props)

Instead of inheritance like `PrimaryButton extends Button`, you do:

```jsx
function Button({ variant = "primary", children, ...rest }) {
  return <button className={`btn btn-${variant}`} {...rest}>{children}</button>;
}

<Button variant="primary">Save</Button>
<Button variant="danger">Delete</Button>
```

---

## 3) Composition using “slots” (named props)

```jsx
function Modal({ header, body, footer }) {
  return (
    <div className="modal">
      <div>{header}</div>
      <div>{body}</div>
      <div>{footer}</div>
    </div>
  );
}

<Modal
  header={<h2>Confirm</h2>}
  body={<p>Are you sure?</p>}
  footer={<button>OK</button>}
/>
```

---

## 4) Composition via render props

```jsx
function Toggle({ children }) {
  const [on, setOn] = React.useState(false);
  return children({ on, toggle: () => setOn(o => !o) });
}

<Toggle>
  {({ on, toggle }) => <button onClick={toggle}>{on ? "ON" : "OFF"}</button>}
</Toggle>
```

---

## 5) Composition via HOC (wrapper)

```jsx
const withLogger = (Component) => (props) => {
  console.log("render", props);
  return <Component {...props} />;
};
```

---

## 6) Composition via custom hooks (modern)

```jsx
function useToggle() {
  const [on, setOn] = React.useState(false);
  return { on, toggle: () => setOn(o => !o) };
}

function UI() {
  const { on, toggle } = useToggle();
  return <button onClick={toggle}>{on ? "ON" : "OFF"}</button>;
}
```

---

## 30-second interview answer

React encourages composition over inheritance because UI is naturally built by combining components. Instead of creating class hierarchies, we reuse behavior via `children`, props-based configuration, slots, render props, HOCs, or most commonly custom hooks. Composition stays flexible, reduces coupling, and keeps data flow explicit through props.
