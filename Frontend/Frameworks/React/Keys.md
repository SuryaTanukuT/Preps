# Keys in React (interview + real-world)


Keys are stable identifiers React uses during reconciliation to match list items between renders. 
With stable unique keys, 
React can preserve DOM and component state correctly when items are inserted, removed, or reordered. 
Using indexes or unstable keys can cause incorrect state reuse, input value jumps, and unnecessary remounts.

Keys are special props React uses to identify elements in a list across renders so it can reconcile (diff) correctly.

One-liner:
“Keys tell React which list item is which between renders, so it can reuse the right DOM and component state.”

Why keys matter
Without stable keys, React matches list items by position (index).
When you insert/delete/reorder, React can:

reuse the wrong DOM nodes
preserve the wrong component state

cause UI bugs (inputs swapping values, wrong rows updating)