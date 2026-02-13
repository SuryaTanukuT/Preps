# Infinite scroll in React (interview + practical)

Infinite scroll = load the next chunk of data automatically when the user approaches the bottom (or top) of a list.

Infinite scroll loads data in chunks as the user approaches the end of the list. 
I typically implement it with an IntersectionObserver watching a sentinel element, 
and I guard against duplicate requests using loading flags and cursor-based pagination. 
For large lists, I combine infinite loading with virtualization so DOM nodes stay limited, and I provide a “Load more” fallback for accessibility.

Interview one-liner:
“Infinite scroll is incremental data loading triggered by scroll position, 
usually implemented with IntersectionObserver, often combined with virtualization for performance.”

# Infinite scroll + virtualization (best combo for big feeds)

Infinite scroll handles data fetching
Virtualization handles DOM size
This is the “production-grade” approach.

Interview-ready statement:
“For large feeds, I combine infinite loading with virtualization to avoid thousands of DOM nodes.”

