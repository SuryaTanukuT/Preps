https://www.freecodecamp.org/news/how-to-choose-a-web-application-firewall-for-web-security/


https://medium.com/@kshitizsharma.00/crafting-web-application-firewall-using-node-js-92eeb5e88f6b

https://www.npmjs.com/package/@mertcanureten/node-waf



## Web Application Firewall (WAF)

A **WAF** protects web apps at **Layer 7 (HTTP/HTTPS)** by inspecting requests/responses and applying rules to **block, challenge, rate-limit, or log** malicious traffic. It sits at the **edge** (CDN / reverse proxy / load balancer) before your app.

---

## What a WAF typically does

* **Detects known attack patterns** (signatures + heuristics)
* **Blocks/limits abusive clients** (rate limits, IP reputation, geo rules)
* **Adds challenges** (CAPTCHA / JS challenge) for bots
* **Virtual patching** (temporary protection until code is fixed)
* Detailed **security logging** and alerting

---

# Coverage by attack type

## 1) SQL Injection (SQLi)

**How WAF helps**

* Detects common SQLi payload patterns:

  * `' OR 1=1--`, `UNION SELECT`, `SLEEP()`, stacked queries, etc.
* Blocks suspicious query strings, headers, body params.

**Best practice**

* Use WAF as **defense-in-depth**.
* Still must do **parameterized queries** + least-privilege DB user.

---

## 2) Cross-Site Scripting (XSS)

**How WAF helps**

* Blocks reflected XSS payloads in:

  * query params, form bodies, headers
* Detects tags/JS patterns:

  * `<script>`, `onerror=`, `javascript:`, SVG-based XSS, etc.

**Best practice**

* Still must do:

  * output encoding + sanitization
  * CSP header
  * avoid unsafe DOM injection

---

## 3) CSRF

**How WAF helps**

* Not a perfect “CSRF fix,” but can reduce risk via:

  * blocking cross-site requests based on **Origin/Referer** anomalies
  * enforcing method policies (deny state changes via GET)
  * requiring custom headers for sensitive endpoints (harder for CSRF)
  * bot/challenge on suspicious state-changing traffic

**Best practice**

* Still must do:

  * SameSite cookies + CSRF tokens/double-submit
  * Origin checks on sensitive endpoints

---

## 4) Bot attacks

**Examples**

* Credential stuffing / brute force
* Scraping, inventory hoarding
* Fake account creation, abusive signups
* Card testing attacks (fintech/ecom)

**How WAF helps**

* Rate limiting (per IP / per user / per path)
* Bot signatures + behavioral detection
* IP reputation / ASN blocking
* CAPTCHA / JS challenges
* Device/browser fingerprinting (platform dependent)

**Best practice**

* Combine with:

  * MFA / risk-based auth
  * lockout/backoff policies
  * login anomaly detection

---

## 5) Layer 7 DDoS (HTTP floods)

**How WAF helps**

* Absorbs + filters high-volume HTTP traffic:

  * request rate limiting
  * connection limiting
  * caching + CDN offload
  * geo blocks / allowlists during incidents
  * challenge mode for suspicious bursts

**Best practice**

* Pair with:

  * CDN caching for static content
  * autoscaling + circuit breakers
  * upstream L3/L4 DDoS protection (cloud provider)

---

## Common WAF controls you should enable (practical)

* ✅ Managed rule sets (OWASP Top 10)
* ✅ Rate limits:

  * `/login`, `/otp`, `/signup`, `/reset-password`
  * `/search`, `/graphql`, expensive endpoints
* ✅ Request size limits (body/header)
* ✅ Allowlist/denylist (IP, geo, ASN) for admin routes
* ✅ Bot protection / challenge on suspicious traffic
* ✅ Central logging → SIEM (ELK/Sentinel/Splunk)

---

## WAF is not enough (what it can’t replace)

* Secure coding (parameterized queries, encoding, CSRF tokens)
* Proper authZ/authN
* Fixing business-logic abuse (refund fraud, IDOR)
* Protecting against insiders or valid credential misuse (needs detection)

---