# Pagination in React (interview + practical)
Pagination = split a large dataset into pages and render only one page at a time. It’s both a UX choice and a data/performance strategy.
1) Types of pagination (must know)A) Client-side pagination
Fetch all items once, paginate in memory.Good when dataset is small/moderate (few hundred–few thousand).
Pros: fast page switches, simple backendCons: initial load heavy, stale data, memory cost
B) Server-side pagination (most real apps)Backend returns one page at a time using limit/offset or cursor.
Good for large datasets.
Pros: scalable, fast initial loadCons: more API calls, must handle loading/error states

# 2) Offset vs Cursor pagination (interview favorite)Offset-based::API: GET /items?page=3&limit=20 or offset=40&limit=20
Pros: easy to implement, supports “jump to page 10”Cons:slow for huge offsets (DB scanning)can show duplicates/missing items if data changes between requests
Cursor-based (recommended for big data / infinite scroll)::
API: GET /items?limit=20&cursor=eyJpZCI6...
Pros:stable when new items are inserted
efficient for large datasets
Cons:harder to “jump to page N”
requires a stable sort key (createdAt/id)