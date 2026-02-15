Code splitting = breaking your JavaScript bundle into smaller chunks so the browser downloads only what’s needed for the current screen.

Result: faster initial load, better performance, especially for large apps.

Why it matters::
Initial bundle gets smaller → quicker first render (better LCP/TTI)
Routes / features load on demand
Users on slow networks benefit a lot
Works well with caching (unchanged chunks stay cached)

The 3 common ways to do code splitting::

1) Route-based splitting (most common)
Load each page only when user navigates to it.

import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));

export default function AppRoutes() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

✅ Best for: SPAs with many routes


2) Component-based splitting (load heavy components only when needed)
Example: load a heavy chart only when user opens a modal/tab.

import { lazy, Suspense, useState } from "react";

const HeavyChart = lazy(() => import("./HeavyChart"));

export default function Analytics() {
  const [open, setOpen] = useState(false);

  return (
    <div>
      <button onClick={() => setOpen(true)}>Open Chart</button>

      {open && (
        <Suspense fallback={<div>Loading chart...</div>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}

✅ Best for: modals, tabs, admin-only panels, rarely used UI

3) Library-level splitting (import only what you use)
Avoid importing huge libraries everywhere.

Bad (may pull a lot):
import _ from "lodash";


Better:
import debounce from "lodash/debounce";

✅ Best for: lodash, date libs, icon packs, rich editors, charts

React tools used for code splitting
React.lazy()
Lazy loads default export components only

Must be used inside <Suspense>
<Suspense fallback>
UI shown while chunk is downloading

Pro-level patterns (useful in interviews)::

Preload a route/component (reduce waiting after click)
You can “warm up” the chunk on hover/focus:

const loadSettings = () => import("./pages/Settings");
const Settings = lazy(loadSettings);

<button onMouseEnter={loadSettings}>Go to Settings</button>


Handle lazy load errors (network failure / chunk mismatch)::
Wrap lazy-loaded parts with an Error Boundary.

class ErrorBoundary extends React.Component {
  state = { hasError: false };
  static getDerivedStateFromError() { return { hasError: true }; }
  render() {
    if (this.state.hasError) return <div>Something went wrong.</div>;
    return this.props.children;
  }
}

Use:

<ErrorBoundary>
  <Suspense fallback={<div>Loading...</div>}>
    <Dashboard />
  </Suspense>
</ErrorBoundary>

Common interview questions & answers::

Q: What is code splitting?
A: Splitting JS into smaller chunks so only required code loads per route/feature, improving initial load and performance.

Q: How do you do code splitting in React?
A: React.lazy + Suspense for component/route splitting; bundlers (Webpack/Vite) create chunks via dynamic import().

Q: Route-based vs component-based splitting?
A: Route-based loads per page navigation; component-based loads heavy UI only when user needs it (modal/tab).

Q: Any downside?
A: More network requests/chunks; if overdone it can cause visible loading states. Fix with good boundaries + preload critical chunks.

Quick best practices::
✅ Split by routes first
✅ Split rare/heavy components (charts, editors)
✅ Keep shared layout (navbar/footer) in main bundle
✅ Use meaningful fallback UI
✅ Consider preloading for likely next navigation
✅ Avoid too many tiny chunks (“over-splitting”)

LCP (Largest Contentful Paint)

What it measures:
Time until the largest visible content element in the viewport is rendered.

Usually the “largest element” is:
Hero image / banner
Big heading (h1)
Large paragraph block
Poster image of a video

Why it matters:
It matches the user feeling: “I can see the main content now.”


TTI (Time To Interactive)

What it measures:
Time until the page is fully interactive (user can click/type and it responds quickly without long JS blocking).

TTI is strongly affected by:
Heavy JavaScript bundles
Long tasks on main thread (big parsing/execution)
Too much hydration work (SSR apps)