
---

# 🔥 1️⃣ Polling vs WebSocket – How You Should Explain

When interviewer asks:

> How did you implement near real-time updates?

You must answer in decision format.

---

## OPTION A — Polling (Most realistic for 2019 enterprise)

You say:

> For transaction dashboards, we implemented short-interval polling (5–10 seconds) to fetch aggregated metrics.

### Why polling?

* Simpler infra
* No need persistent connections
* Enterprise firewall friendly
* Aggregated dashboards don’t need millisecond updates

### Example flow:

Frontend:

* setInterval → call API
* Fetch aggregated metrics (TPS, fraud rate, etc.)
* Update Redux store
* Re-render charts

Backend:

* API returns pre-aggregated values
* Avoid heavy computation per request

That’s safe answer.

---

## OPTION B — WebSocket (if used)

If you say WebSocket, you must explain properly:

> For high-frequency updates like transaction per second counters, we used WebSockets to push updates from server to client.

Flow:

Backend:

* On new transaction event → publish to WebSocket channel

Frontend:

* WebSocket client listens
* Dispatch Redux action on event
* Update state → charts update

When to choose WebSocket?

* If sub-second updates required
* If thousands of updates per minute

If you didn’t truly implement WebSocket, don’t overclaim. Polling is fine.

---

# 🔥 2️⃣ Redux State Management – How It Actually Fits

In dashboard architecture:

You have:

* Filters (date range, region, card type)
* API data
* User role info
* UI state (selected graph, zoom level)

Redux is used for:

* Centralized state
* Predictable updates
* Debugging via devtools
* Avoiding prop drilling

Architecture:

Component → dispatch action
Action → reducer
Reducer → update store
Store → subscribed components re-render

Explain:

> Each polling/WebSocket update dispatched Redux actions, and charts subscribed only to relevant slices of state to avoid unnecessary re-renders.

That sounds senior.

---

# 🔥 3️⃣ Performance Optimization (Very Important)

If they grill you, this is where they test real skill.

---

## A) Memoization

Problem:
Large dashboards re-render too often.

Solution:

* React.memo for pure components
* useMemo for expensive data transformations
* useCallback for stable function references

Example:

```javascript
const processedData = useMemo(() => {
  return transformData(rawData);
}, [rawData]);
```

Why?
Avoid recomputing heavy D3 data processing.

---

## B) Virtualization

If you had:

* Large transaction tables
* Thousands of rows

Use:

* react-window
* react-virtualized

Explain:

> Instead of rendering 5000 DOM rows, we rendered only visible rows inside viewport.

This saves DOM memory and improves FPS.

---

## C) Data Sampling

For charts:

Instead of plotting 50,000 points:

* Bucket into intervals
* Downsample data

Say:

> We reduced dataset size on client side by aggregating per minute instead of per second for larger time ranges.

That’s a mature answer.

---

## D) WebGL vs SVG

Say:

> When rendering high-density fraud heatmaps, SVG performance degraded, so we used WebGL for GPU acceleration.

Strong technical point.

---

# 🔥 Now — VISA TECHNICAL GRILLING ROUND

I’ll simulate like real interviewer.

You answer mentally.

Then I’ll give you the ideal answer.

---

## 🔴 Question 1

> If your dashboard is polling every 5 seconds and 10,000 users are active, won’t that overload backend?

Think…

---

### ✅ Ideal Answer

* We never returned raw transactions.
* We returned pre-aggregated metrics.
* API layer cached results for short window (e.g., Redis 3–5 seconds).
* Horizontal scaling behind load balancer.

Key phrase:
**“Cache + pre-aggregation + horizontal scaling.”**

---

## 🔴 Question 2

> How do you prevent Redux store from becoming too large?

Ideal answer:

* Store only normalized, minimal required data.
* Don’t store raw massive datasets.
* Clear stale data when filters change.
* Use pagination for transaction lists.

---

## 🔴 Question 3

> How do you handle chart re-render performance issues?

Ideal answer:

* Memoization
* Avoid deep object mutation
* Immutable updates
* Selectors (reselect)
* Only re-render changed components

---

## 🔴 Question 4

> If TPS spikes suddenly from 500 to 5000 per second, what breaks first?

Strong answer:

* Backend aggregation service
* Database write bottleneck
* Network bandwidth
* Client rendering if pushing raw data

Then say mitigation:

* Autoscaling
* Rate limiting
* Buffering
* Backpressure

That sounds architectural.

---

## 🔴 Question 5 (Very Senior)

> Why not use server-side rendering for dashboards?

Answer:

* Data is user-specific and dynamic.
* SSR provides little SEO benefit for internal dashboards.
* CSR better for highly interactive real-time apps.

---

## 🔴 Question 6 (Micro-frontend Trap)

> Why split into micro-frontends? Isn’t that overengineering?

Answer:

* Large enterprise teams working independently.
* Separate deployment cycles.
* Reduced blast radius.
* But introduces complexity (shared deps, version conflicts).

Mention tradeoff.

---

# 🧠 Now Important

Don’t try to sound like:

“I built distributed streaming system”

For Infosys, you were:

* Senior UI engineer
* Visualization architect
* Performance optimizer
* Early micro-frontend experiment contributor

That’s strong enough.

---


