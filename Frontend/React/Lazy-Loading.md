# Lazy loading in React (interview + practical)

Lazy loading = load code (or data/assets) only when needed, 
reducing initial bundle size and improving first load (LCP/TTI).

Lazy loading in React is usually code splitting via dynamic imports using React.lazy() and Suspense so heavy code loads only when needed, improving initial load time. 
I prioritize route-level splitting first, then split heavy features like charts or editors. 
I use Suspense fallbacks for loading states and wrap lazy boundaries with an error boundary, and optionally prefetch likely-next chunks to make navigation feel instant.

Interview one-liner:
“In React, lazy loading is mainly code-splitting with React.lazy() + Suspense, 
typically at route level or heavy component boundaries.”

# Code splitting vs Lazy loading

Code splitting: bundler creates separate JS chunks.
Lazy loading: app loads those chunks on demand.
Most React lazy loading is done via dynamic import:

import("./HeavyComponent");

# React.lazy + Suspense (core)
import { lazy, Suspense } from "react";

const Heavy = lazy(() => import("./Heavy"));

export default function Page() {
  return (
    <Suspense fallback={<div>Loading…</div>}>
      <Heavy />
    </Suspense>
  );
}


# Key points interviewers want:
React.lazy works for default exports
Suspense provides the fallback while chunk is loading
Great for routes, modals, large dashboards, charts, editors

# Where to lazy load (best practice)
✅ Route-level (biggest win)
Each route/page chunk loads when user navigates.

✅ Feature-level
heavy widgets (charts, editor, maps)
modals opened rarely
admin panels

Usually NOT worth it
tiny components (splitting too much creates extra network overhead)

Interview line:
“Start with route-level splitting; then split heavy features used rarely.”

# Lazy loading images/assets
Code splitting doesn’t solve image cost. For images:
use native loading="lazy"
responsive srcSet
compress/optimize

<img src="/banner.webp" loading="lazy" alt="..." />

# Lazy loading data ≠ lazy loading code

Don’t confuse:
React.lazy = code
data fetching = use React Query/SWR or route loaders

But they combine well: show skeleton + fetch.

