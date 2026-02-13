# Virtualization in React (interview perspective)

Virtualization (a.k.a. windowing) = rendering only the visible items in a large list/grid (plus a small buffer), 
instead of mounting thousands of DOM nodes.

Why it matters:
DOM nodes are expensive (layout, paint, memory)
React rendering + reconciliation cost grows with list size
Scrolling becomes janky if you render too much

Interview one-liner:
“Virtualization keeps performance stable by limiting DOM to what’s on screen.”


Virtualization renders only the items visible in the viewport plus a small buffer, while maintaining the full scroll height with spacers. This dramatically reduces DOM nodes, lowering layout/paint cost and React reconciliation work, keeping scrolling smooth for large lists. I prefer fixed-height rows when possible; for variable heights I’d use measurement/caching. Often I combine virtualization with infinite loading for very large datasets.

# When to use virtualization

Use it when you have:
hundreds/thousands of rows/cards
slow rendering/scrolling
heavy row components (images, complex layouts)

You usually don’t need it for 20–100 items unless rows are very heavy.


# popular libraries (what interviewers expect)

react-window (lightweight, most common)
react-virtualized (older, heavier, more features)
TanStack Virtual (framework-agnostic, modern)

If asked “which would you choose?”:
react-window for simple lists/grids
TanStack Virtual for more control / dynamic scenarios


# Fixed-size vs Variable-size lists
Fixed-size (easiest + fastest)
each row has same height
simplest math
Variable-size (harder)
row height differs
you need measurement/caching
more edge cases (jumping scroll, reflow)

# Key pitfalls interviewers ask
1) Keys still matter

Use stable IDs:
✅ key={item.id}
❌ key={index} (causes wrong row reuse when sorting/filtering)

2) Don’t rely only on memoization
React.memo helps, but it still keeps thousands of DOM nodes.
Virtualization reduces DOM size — bigger win.

3) Overscan tuning
Overscan = render a few extra items above/below to avoid blank gaps while fast scrolling.

Too low → flicker
Too high → extra work

4) Images and dynamic content
Images loading can change heights → breaks variable-size virtualization unless handled (measure after load).

5) Accessibility
Virtualized lists may confuse screen readers since off-screen items aren’t in DOM.
Solutions: aria patterns, alternative rendering mode, or pagination for accessibility-heavy screens.
