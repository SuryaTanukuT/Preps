Diffing = how React figures out what changed between the previous UI and the new UI, so it can update the real DOM with minimum operations.

React does this during reconciliation:

You update state/props

React renders → creates a new Virtual DOM tree (React elements)

React diffs old vs new tree

React updates the real DOM only where needed


Core rules React uses (interview-important)
1) Different element type ⇒ replace the whole subtree

If the element type changes, React doesn’t try to compare deeply.

Example:

// before
<div>...</div>

// after
<span>...</span>


React will remove <div> subtree and create <span> subtree.

2) Same element type ⇒ update attributes + diff children

Example:

// before
<button className="a">OK</button>

// after
<button className="b">OK</button>


React keeps the same <button> node and just updates className.


3) Children lists are matched by key

When rendering arrays, React uses keys to know which item is which.

items.map(item => <Row key={item.id} item={item} />)


✅ Good: stable key like id
❌ Bad: index as key (can break updates when items are inserted/reordered)

Bad: using index key
{todos.map((t, i) => <Todo key={i} todo={t} />)}


If you insert a todo at the top, React thinks “same indexes” = same items, so it may:
reuse wrong DOM nodes
keep wrong input values
cause weird UI bugs


Good: use stable IDs
{todos.map(t => <Todo key={t.id} todo={t} />)}

Diffing vs Rendering vs Commit (often mixed in interviews)
Render phase: React builds new virtual tree + diffs it (can be paused in concurrent mode)
Commit phase: React applies changes to DOM (cannot be interrupted)

Common interview Q&A::
Q: What is React diffing?
A: Comparing old and new virtual DOM trees to compute minimal DOM updates.

Q: What triggers a full remount of a subtree?
A: When element/component type changes, or when keys change (React treats as new).

Q: Why keys are important?
A: Keys help React preserve element identity across renders, preventing wrong updates and improving performance.