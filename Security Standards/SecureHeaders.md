
https://expressjs.com/en/advanced/best-practice-security.html
https://blog.risingstack.com/node-js-security-checklist/
https://nodejs.org/en/learn/getting-started/security-best-practices
https://medium.com/@dangeorge35/security-headers-and-how-to-set-up-in-react-node-d20804a105b0
https://cheatsheetseries.owasp.org/cheatsheets/Nodejs_Security_Cheat_Sheet.html
https://www.freecodecamp.org/news/how-to-harden-your-nodejs-apis-security-best-practices/


## Security Headers (Must-have)

### 1) Content-Security-Policy (CSP)

**Purpose:** Mitigates **XSS** by restricting what scripts/resources the browser can load/execute.

**Common baseline (adjust for your app/CDN)**

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
```

**Notes**

* Avoid `script-src 'unsafe-inline'` (weakens CSP).
* For modern apps, use **nonces/hashes** for inline scripts if needed.

---

### 2) Strict-Transport-Security (HSTS)

**Purpose:** Forces HTTPS (prevents SSL stripping).

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

**Notes**

* Only enable when you’re sure HTTPS works everywhere (including subdomains).
* `preload` is optional but strong once committed.

---

### 3) X-Frame-Options

**Purpose:** Prevents **clickjacking** (your site being framed by others).

```http
X-Frame-Options: DENY
```

**Notes**

* Prefer CSP `frame-ancestors` for modern control, but this header is still widely used.

---

### 4) X-Content-Type-Options

**Purpose:** Stops MIME sniffing (helps prevent content-type confusion attacks).

```http
X-Content-Type-Options: nosniff
```

---

### 5) Referrer-Policy

**Purpose:** Controls how much referrer info is sent to other sites (privacy + reduces leakage).

**Good default**

```http
Referrer-Policy: strict-origin-when-cross-origin
```

---

### 6) Permissions-Policy

**Purpose:** Restricts powerful browser features (camera, mic, geolocation, etc.).

**Example baseline**

```http
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=(), usb=()
```

---

## Recommended “starter set” (copy-paste)

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'none'
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=(), payment=()
```

---

## Bonus headers (often added)

* `Cross-Origin-Opener-Policy: same-origin`
* `Cross-Origin-Resource-Policy: same-origin`
* `Cross-Origin-Embedder-Policy: require-corp` *(only if you need cross-origin isolation)*
* `Cache-Control: no-store` for sensitive pages (banking/account pages)

