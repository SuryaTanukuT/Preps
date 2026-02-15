# SSR — Server-Side Rendering

What it is: HTML is generated on every request on the server.

Flow: Request → server renders page → sends HTML → browser shows → JS hydrates (if interactive).

Best for:
highly dynamic pages (prices, stock, personalized dashboards)
content must be fresh per request

Tradeoffs:
slower than static for high traffic (server work each request)
needs caching/CDN strategy to scale

(Next.js calls this “dynamic rendering” in App Router terminology, depending on caching config.)


# SSG — Static Site Generation

What it is: HTML is generated at build time and served as static files (CDN-friendly).
Flow: Build → generate HTML once → deploy → all users get same static HTML.

Best for:
marketing pages, docs, blogs
content changes rarely

Tradeoffs:
content can become stale until the next build/deploy
large sites can have long build times

Next.js definition: “HTML is generated at build time and reused on each request.”

# ISR — Incremental Static Regeneration

What it is: Pages are static like SSG, but can be regenerated after deployment—either:
time-based revalidation (every N seconds), or
on-demand revalidation (trigger via webhook/event)

Flow (typical): Serve cached page → when stale, next request triggers regeneration in background → cache updates → subsequent users get fresh page.

Best for:
content that changes sometimes (blogs, product/category pages)
want CDN speed + controlled freshness

Tradeoffs:
not real-time (freshness is “eventual” within your revalidate window)
requires correct caching/revalidate setup
