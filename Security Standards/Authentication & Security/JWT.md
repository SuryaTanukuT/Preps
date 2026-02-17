https://blog.galmalachi.com/react-nodejs-and-jwt-authentication-the-right-way
https://medium.com/@prashantramnyc/authenticate-rest-apis-in-node-js-using-jwt-json-web-tokens-f0e97669aad3
https://www.digitalocean.com/community/tutorials/nodejs-jwt-expressjs


## Rotate secrets/keys & support **JWKS key rotation** for JWT

When you use **asymmetric JWT signing** (recommended: `RS256` / `ES256`), you rotate **signing keys** safely by publishing **public keys** via a **JWKS endpoint** and letting APIs fetch the right key using the token’s `kid`.

---

# 1) Key terms

### **JWK**

A JSON representation of a key (public or private). For APIs you publish **public** keys only.

### **JWKS (JSON Web Key Set)**

A JSON document containing **one or more public keys**:

* `GET https://issuer.example.com/.well-known/jwks.json`

### **kid (Key ID)**

A short identifier in the JWT header:

```json
{ "alg": "RS256", "kid": "key-2026-02", "typ": "JWT" }
```

The API uses `kid` to pick the right public key from JWKS.

---

# 2) Recommended signing strategy

✅ Prefer **asymmetric** keys (best practice):

* `RS256` (RSA) or `ES256` (ECDSA)
* Private key stays in the Auth Server / IdP only
* APIs verify using the public key from JWKS

⚠️ Avoid `HS256` secret rotation for multi-service setups unless you *really* know what you’re doing (shared secret distribution becomes hard and risky).

---

# 3) How JWKS rotation works (safe “overlap” rotation)

### Step-by-step

1. **Generate new keypair**

   * New private key + public key
   * Assign a new `kid` (e.g., `key-2026-02`)

2. **Publish public key to JWKS**

   * JWKS now contains:

     * old public key (kid old)
     * new public key (kid new)

3. **Start signing new tokens with the new private key**

   * JWT header now uses `kid=new`

4. **Keep old key in JWKS during overlap window**

   * So tokens already issued with old `kid` keep verifying until they expire

5. **Remove old key from JWKS only after all old tokens expire**

   * Overlap duration ≥ max token TTL (plus a small buffer)

### Overlap rule of thumb

* If access tokens live for **15 minutes**, keep old keys for **at least 15–30 minutes**
* If you also validate long-lived tokens, adjust accordingly

---

# 4) API-side verification logic (what your Nest/API must do)

Your API should:

* Parse JWT header → read `kid`
* Fetch JWKS → find matching key
* Verify signature + claims (`iss`, `aud`, `exp`, etc.)
* Cache JWKS keys (to avoid fetching on every request)
* Refresh JWKS on:

  * cache expiry
  * unknown `kid` (“cache miss”)

---

# 5) Caching strategy (important)

### What to do

* Cache JWKS for **5–60 minutes** (depends on traffic + rotation frequency)
* Respect `Cache-Control` headers if your JWKS endpoint sets them
* On `kid` not found:

  1. re-fetch JWKS immediately (single-flight / lock to avoid stampede)
  2. retry verification once
  3. if still missing → reject token (401)

### Why

* Prevents downtime right after rotation
* Avoids hammering JWKS endpoint under high traffic

---

# 6) NestJS / Node example using `jose` (JWKS + rotation-ready)

### Install

```bash
npm i jose
```

### JWKS client + verifier (rotation-safe)

```ts
import { createRemoteJWKSet, jwtVerify } from "jose";

const ISSUER = "https://issuer.example.com";
const AUDIENCE = "my-api";
const JWKS = createRemoteJWKSet(new URL(`${ISSUER}/.well-known/jwks.json`));

export async function verifyAccessToken(token: string) {
  const { payload, protectedHeader } = await jwtVerify(token, JWKS, {
    issuer: ISSUER,
    audience: AUDIENCE,
  });

  // protectedHeader.kid is used internally by jose to pick the right key from JWKS
  // payload contains claims (sub, scope, roles, exp, etc.)
  return { payload, kid: protectedHeader.kid };
}
```

**Why this works well**

* `createRemoteJWKSet` handles fetching + caching
* It automatically selects the correct key by `kid`
* When keys rotate, verification continues as long as JWKS contains the right public keys

---

# 7) “Secret rotation” vs “Key rotation” (JWT context)

### HS256 (shared secret)

* Rotate by changing the shared secret everywhere
* To support overlap, you must allow **multiple secrets** temporarily (try old, then new)
* Operationally painful in microservices

### RS256/ES256 (keypair)

* Rotate by:

  * start signing with new private key
  * publish new public key in JWKS
  * keep old public key until old tokens expire
* Much cleaner for distributed systems

---

# 8) Operational best practices

* Use **short-lived access tokens** (5–15 min)
* Ensure tokens include `kid`
* Keep JWKS endpoint highly available (CDN/cache it if possible)
* Alert on:

  * sudden spike in “unknown kid”
  * signature verification failures
* Have an “emergency rotation” plan:

  * rotate keys
  * reduce token TTL temporarily
  * revoke refresh tokens / invalidate sessions if compromise suspected

---
## JWT-based auth: logging, rate-limiting, validation, and authorization

---

# 1) Log auth events (login, refresh, revoke, suspicious failures)

### What to log (minimum)

* **LOGIN_SUCCESS / LOGIN_FAILURE**
* **TOKEN_ISSUED** (access token)
* **TOKEN_REFRESH_SUCCESS / TOKEN_REFRESH_FAILURE**
* **TOKEN_REVOKED** (logout / admin revoke)
* **AUTHZ_DENIED** (permission/scope failure)
* **TOKEN_INVALID** (bad signature, expired, wrong issuer/audience, unknown `kid`)
* **SUSPICIOUS_ACTIVITY** (replay, brute force, impossible travel, unusual IP/device)

### Include these fields (structured logs)

* `timestamp`
* `eventType`
* `userId` (`sub`) and `tenantId/orgId` (if multi-tenant)
* `clientId` / `appId` (OAuth client)
* `requestId` / `correlationId` (trace id)
* `ip` (real client IP via trusted proxy headers)
* `userAgent` (or device id)
* `result` (success/failure) + `reasonCode`
* `tokenKid` (JWT header `kid`) when relevant

### Don’t log

* Raw tokens (`access_token`, `refresh_token`)
* Passwords, OTPs, secrets
* Full PII (mask where needed)

### Suspicious patterns to detect & log

* Many login failures for same user / same IP
* Refresh token **reuse** (rotation replay)
* Repeated “unknown `kid`” (misconfig / attack)
* Tokens with wrong `aud`/`iss` (token swapping)
* High 401 rate from one IP / ASN

---

# 2) Rate-limit auth endpoints (JWT flows)

Even if your main API uses JWT, the **auth endpoints** are attack magnets.

### Apply rate limits to

* `/auth/login`
* `/auth/token` (code exchange) *(if you run it)*
* `/auth/refresh`
* `/auth/logout`
* `/auth/forgot-password`, `/auth/otp`, `/auth/mfa/*`

### Use layered throttling

* **Per IP** (basic)
* **Per username/email** (prevents distributed brute force)
* **Per device/session** (if you have a device fingerprint)

### Typical policies (example)

* Login: `5–10 req/min/IP` + `3–5 req/min/username`
* Refresh: `10–30 req/min/user` (depends on token TTL)
* OTP: strict + cooldown windows

### Add protections

* Exponential backoff after failures
* Temporary lockouts (carefully to avoid attacker-driven DoS)
* Captcha after threshold (public apps)

---

# 3) Validate tokens (or sessions) on every request

### JWT validation (every protected request)

✅ Verify **signature** using:

* JWKS (for RS256/ES256) or shared secret (HS256, less preferred)

✅ Validate claims:

* `iss` (issuer must match)
* `aud` (audience must match your API)
* `exp` (not expired)
* `nbf` (not-before, if used)
* `iat` (optional sanity)
* `kid` should resolve to a key in JWKS (rotation-friendly)

✅ Additional best checks

* Clock skew tolerance (30–120 seconds)
* Reject `alg=none`
* Ensure expected algorithm (don’t accept arbitrary algs)
* For multi-tenant: validate tenant/org claims and route constraints

### Revocation reality with JWT

Access tokens are usually **not revocable** until expiry, so:

* Keep them **short-lived** (5–15 minutes)
* Use **refresh tokens** (rotating, server-stored) for long sessions
* Maintain a revoke list for refresh tokens / session ids
* For critical revokes: reduce TTL + invalidate refresh token + denylist jti (optional)

---

# 4) Enforce authorization with scopes/roles/permissions (JWT)

### Common JWT claims used

* `scope` / `scp`: `payments:read payments:write`
* `roles`: `Admin`, `Ops`
* `permissions`: `txn.refund`, `ledger.adjust`
* `sub`: user id
* `tenantId` / `orgId`: multi-tenant boundary

### How to enforce

* **Route-level scope checks**

  * Example: `POST /payments` requires `payments:write`
* **Role checks**

  * Example: `GET /admin/*` requires `Admin`
* **Permission checks** (fine-grained)

  * Example: `POST /txn/:id/refund` requires `txn.refund`
* **Resource-level policies (ABAC)**

  * Example: user can access only accounts where `tenantId` matches token claim

### Best practice model

* **Default deny**
* Centralized policy engine/service:

  * `can(user, action, resource)` → allow/deny
* Avoid putting *too much* authorization data in JWT:

  * Keep roles/scopes minimal
  * For complex entitlements, fetch from DB/cache (or use short TTL tokens)

---

## Minimal “JWT security chain” (what every request should go through)

1. Extract token (`Authorization: Bearer …`)
2. Verify signature using JWKS / expected algorithm
3. Validate claims (`iss`, `aud`, `exp`, etc.)
4. Build `principal` object (user + tenant + scopes/roles)
5. Enforce authorization (scopes/roles/permissions + resource checks)
6. Log outcome (authn fail / authz deny / success), with correlation id

