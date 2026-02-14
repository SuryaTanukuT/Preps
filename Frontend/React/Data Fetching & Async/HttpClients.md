# HTTP clients in React (high-level interview perspective)

In React, an “HTTP client” is how your UI talks to APIs (REST/GraphQL). React itself doesn’t include one—you choose patterns + libraries. React I can use fetch or axios as the HTTP client, but for production I usually layer them under a data-fetching library like React Query or SWR to handle caching, deduping, retries, pagination, and loading/error states. 

I centralize an API client for base URL and auth headers, handle token refresh on 401, abort requests to avoid race conditions, and invalidate cached queries after mutations.

# Common options
A) Native fetch (built-in)
✅ No dependency, standard Web API
✅ Works everywhere modern browsers + Node (newer versions / polyfills)
❌ Doesn’t reject on HTTP errors (404/500 are still “resolved”)
❌ No request/response interceptors built-in
❌ Timeout not built-in (use AbortController)
❌ Need to manually parse JSON + handle error bodies

Best for: simple apps, small number of calls.

```
async function getUsers() {
  const res = await fetch("/api/users");

  // fetch only rejects for network errors, not for 4xx/5xx
  if (!res.ok) {
    const errBody = await res.text(); // or res.json() if API returns JSON
    throw new Error(`HTTP ${res.status}: ${errBody}`);
  }

  return res.json();
}

```


B) axios
✅ Rejects promise for non-2xx (easy error handling)
✅ Interceptors (auth token, refresh, logging)
✅ Request cancellation support
✅ Automatic JSON transform + better defaults
❌ Extra dependency
❌ Slightly larger bundle than fetch

Built-in interceptors, automatic JSON transform, nicer error handling

Best for: apps needing interceptors (auth tokens, refresh, logging), consistent request pipeline.

```
import axios from "axios";

async function getUsers() {
  const res = await axios.get("/api/users");
  return res.data; // axios already parses JSON
}

```

C) Data-fetching libraries (recommended for real apps)
Not just “HTTP client”—they handle caching & state:

TanStack Query (React Query)
SWR

They manage:
caching, dedupe, retries
loading/error states
pagination/infinite scroll
background refetching, stale-while-revalidate

Interview line:
“In production, I prefer React Query/SWR because data fetching is more than calling HTTP—it’s caching, retries, and sync.”

D) GraphQL clients
Apollo Client
urql
They handle caching, normalized store, query batching, etc.

# Where to call APIs (important)
✅ Best practice
Call APIs in:
useEffect (basic)

or via React Query/SWR hooks (preferred)

❌ Avoid
calling APIs directly inside render (impure render → repeated calls)
