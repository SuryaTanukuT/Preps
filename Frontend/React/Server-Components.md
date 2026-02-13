# Server Components in React

Server Components (RSC) are React components that run only on the server. Their code does not ship to the browser, and the browser receives a streamed result that React can combine with interactive “client components”.

What problems they solve::
Less JavaScript sent to the client → smaller bundle, faster load for data-heavy pages.

Server-side data access: you can fetch from DB/services on the server without exposing secrets to the browser.

Streaming UI with Suspense: server can stream parts of the UI as they’re ready; the client shows fallbacks while waiting.

# The key rule

Server Components cannot use client-only/interactivity APIs like useState (and generally anything that needs the browser). To add interactivity, you compose them with Client Components.


# How it works at a high level

Server renders Server Components and can fetch in parallel on the server.

Output is streamed to the client (often coordinated with Suspense fallbacks).

Client Components are downloaded/hydrated to make parts interactive (buttons, inputs, etc.).

# Where you’ll actually use RSC today

Most developers use Server Components through frameworks like Next.js (App Router), where Server Components are the default and 'use client' marks interactive parts.

# Common interview Q&A (super short)

Can Server Components use useEffect / useState? No—those are client interactivity APIs; use a Client Component.

Do Server Components replace SSR? Not exactly—RSC is a component model that can work with streaming/SSR/hydration patterns; frameworks integrate them together.

Biggest benefit? Smaller client JS + server-side data fetching + streaming
