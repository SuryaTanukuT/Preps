

https://www.stackhawk.com/blog/node-js-csrf-protection-guide-examples-and-how-to-enable-it/


## CSRF (Cross-Site Request Forgery)

CSRF happens when a **browser automatically includes your user’s credentials (usually cookies)** on a request triggered by a malicious site, causing state-changing actions (transfer money, change email, etc.) without the user’s intent.

---

## SameSite cookies

**Goal:** Reduce/stop cookies being sent on cross-site requests.

### Key values

* `SameSite=Strict`

  * Cookies **only** sent in same-site context.
  * Strong protection, but may break “open from email link” flows.
* `SameSite=Lax` *(good default for many apps)*

  * Cookies not sent on most cross-site requests.
  * Still sent on **top-level GET navigations** (clicking a link).
* `SameSite=None; Secure`

  * Cookies sent cross-site **only if** `Secure` is set (HTTPS).
  * Needed for third-party/embedded scenarios (iframes, cross-site SSO), but increases CSRF risk → you must add extra CSRF defenses.

### Recommended cookie flags (auth/session)

* `HttpOnly` (prevents JS access)
* `Secure` (HTTPS only)
* `SameSite=Lax` or `Strict` (prefer Lax/Strict where possible)

---

## CSRF tokens (Synchronizer Token Pattern)

**Idea:** Server issues a **random token** tied to the user session, and requires it on every state-changing request.

### Flow

1. User loads page → server embeds CSRF token (HTML meta tag / response body).
2. Client sends token in a header or body with POST/PUT/PATCH/DELETE:

   * `X-CSRF-Token: <token>`
3. Server validates token matches what it expects for that session.

### Pros

* Strong, standard, reliable.
* Works well with cookie-based sessions.

### Cons

* Extra plumbing (token storage, rotation, sending from frontend).
* If you build pure APIs for mobile/3rd party clients, it’s extra complexity.

---

## Double-submit cookies

**Idea:** Put a CSRF value in:

* a cookie, **and**
* a request header/body field
  …and require they match.

### Flow

1. Server sets `csrf=<random>` cookie *(should NOT be HttpOnly if JS needs to read it)*.
2. Frontend reads cookie and sends it in a header:

   * `X-CSRF-Token: <value from cookie>`
3. Server checks:

   * cookie `csrf` == header `X-CSRF-Token`

### Pros

* No server-side token storage required (stateless-ish).
* Good when you don’t want session-backed token storage.

### Cons / Gotchas

* If an attacker can run JS on your site (XSS), they can bypass it.
* Must still use `SameSite`, CORS rules, and strong XSS defenses.
* If the token is readable by JS, it’s not a “secret” against XSS—only against cross-site form submits.

---

## Avoiding cookies for APIs (use JWT)

**Idea:** If your API auth is **not automatically attached by the browser**, CSRF becomes much harder.

### Typical approach

* Store access token in memory (or secure storage on mobile)
* Send explicitly:

  * `Authorization: Bearer <JWT>`

### Why it reduces CSRF risk

* A malicious site cannot force the victim’s browser to attach an `Authorization` header to your API request (without your JS running / without CORS allowing it).

### Trade-offs

* You must handle **XSS seriously** (JWT in `localStorage` is vulnerable).
* Prefer:

  * short-lived access tokens
  * refresh tokens carefully (often still cookie-based → refresh endpoint needs CSRF protection)
  * token rotation & revocation strategy

---

## Practical guidance (quick rules)

* **Cookie session auth + browser app:**
  Use `SameSite=Lax/Strict` **and** CSRF tokens (synchronizer) or double-submit.
* **Pure API consumed by SPA/mobile:**
  Prefer `Authorization: Bearer` access tokens; keep refresh flow secure (often needs CSRF if refresh uses cookies).
* **Cross-site embedding/SSO requiring `SameSite=None`:**
  Assume higher CSRF risk → enforce CSRF tokens + tight CORS + origin checks.

