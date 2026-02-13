# Lists in React (interview + practical)

Rendering lists in React = map data → JSX, with the right keys, and careful handling of performance + state

In React, lists are rendered by mapping arrays to JSX elements. 
The key part is using stable unique keys so React can reconcile efficiently and preserve correct component state when items are inserted, removed, or reordered. 
For large datasets, I optimize with memoized row components, memoized filtering/sorting, and virtualization to limit DOM nodes.

# Basic list rendering
function Users({ users }) {
  return (
    <ul>
      {users.map(u => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}


Must know: map() returns an array of React elements.

# Keys (most important interview point)

✅ Use stable unique keys:

key={item.id}


❌ Avoid index keys when list can change order/add/remove:

key={index}


Reason:
React uses keys for reconciliation

wrong keys → wrong DOM/state reuse (inputs “swap”, checkbox state jumps)

# Conditional rendering inside lists
{items.map(item =>
  item.active ? <Row key={item.id} item={item} /> : null
)}


Better approach for readability:

{items.filter(i => i.active).map(i => (
  <Row key={i.id} item={i} />
))}
