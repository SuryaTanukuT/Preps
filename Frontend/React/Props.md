# Props in React (high-level interview perspective)
Props (properties) are the inputs a component receives from its parent. They let you configure a component and share data down the component tree.

Props are read-only inputs passed from parent to child that configure a component and enable one-way data flow. A child shouldn’t mutate props; instead it communicates changes upward via callback props. props.children supports composition. Props changes cause re-renders, and for performance we can memoize components and keep prop references stable when necessary.

Key rule: props are read-only in the child.

1) Why props exist
Reusability: same component, different data
Composition: parent controls what child shows/does
One-way data flow: parent → child makes UI predictable

# How props are passed
Basic::
<UserCard name="Surya" role="Lead" />

Child receives::
function UserCard({ name, role }) {
  return <div>{name} - {role}</div>;
}

Passing objects/arrays::
<UserCard user={userObj} />


# Props are immutable (do not mutate)

Bad:
props.user.name = "x"; // ❌


Good:
ask parent to update via callback
or copy and compute

# Passing functions as props (child → parent communication)
This is super common in interviews.

Parent:
<AddTodo onAdd={handleAdd} />


Child:
function AddTodo({ onAdd }) {
  return <button onClick={() => onAdd("task")}>Add</button>;
}


Interview line:
“Data flows down via props; events flow up via callback props.”

# Default props
Modern way: default values in destructuring
function Button({ type = "button", disabled = false }) { ... }




