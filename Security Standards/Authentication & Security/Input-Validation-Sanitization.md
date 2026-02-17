https://www.nodejs-security.com/blog/input-validation-best-practices-for-nodejs

## Input Validation & Sanitization

**Goal:** Treat all external input as untrusted and prevent it from being interpreted as code (SQL, JS, shell, templates).

**Two layers**

* **Validation:** Is the input *allowed* (type, format, range, length, whitelist)?
* **Sanitization/Encoding:** Make it *safe to output/execute* in a specific context (HTML, SQL, shell, template, URL).

---

## SQL Injection

**What it is:** Attacker injects SQL into a query via user input.

**Example (bad)**

* `SELECT * FROM users WHERE email = '${email}'`

**Impact**

* Data leaks, auth bypass, data modification, dropping tables.

**Best defenses**

* ✅ **Parameterized queries / prepared statements** (always)
* ✅ Least-privilege DB user (no `DROP`, no superuser)
* ✅ Input validation (especially IDs, enums)
* ✅ Avoid dynamic SQL; if needed, whitelist column names/order-by fields

---

## XSS (Cross-Site Scripting)

**What it is:** Attacker injects JavaScript into pages viewed by other users.

**Types**

* **Stored XSS:** saved in DB, served to many users
* **Reflected XSS:** echoed from request into response
* **DOM XSS:** client-side JS writes unsafe data into DOM

**Impact**

* Session hijack, account takeover, data exfiltration, malicious actions as user.

**Best defenses**

* ✅ **Contextual output encoding** (HTML/attr/JS/URL contexts)
* ✅ Avoid `dangerouslySetInnerHTML` / raw HTML rendering
* ✅ Sanitize HTML if you must allow rich text (allowlist tags/attrs)
* ✅ **CSP** to reduce blast radius
* ✅ HttpOnly cookies (limits cookie theft, not full XSS protection)

---

## Command Injection

**What it is:** Attacker injects OS commands when your app builds shell commands from input.

**Example (bad)**

* `exec("convert " + filename + " out.png")`

**Impact**

* Remote code execution, server takeover, data theft.

**Best defenses**

* ✅ Avoid shell: use library APIs instead of `sh -c`
* ✅ If you must run commands:

  * use `execFile/spawn` with argument arrays (no shell)
  * strict allowlists for inputs (filenames, flags)
* ✅ Run with least privilege + sandboxing/containers

---

## Template Injection (SSTI)

**What it is:** User input is evaluated by a server-side template engine as template code.

**Example risk**

* `render("Hello " + userInput)` where engine interprets `{{...}}`, `${...}`, etc.

**Impact**

* Data exposure, SSRF, remote code execution (engine dependent).

**Best defenses**

* ✅ Never treat user input as a template
* ✅ Use safe variable interpolation only (no `eval`, no dynamic template compile)
* ✅ Sandbox template engines (if supported)
* ✅ Allowlist any “template-like” features (macros/filters) very carefully

---

# Techniques

## Parameterized queries (Recommended)

**What it does:** Separates code from data, so input can’t change query structure.

* ✅ Works for WHERE values, inserts, updates, etc.
* ⚠️ Not for identifiers (table/column names) → use allowlists for those.

---

## Escaping user input

**What it does:** Converts special characters so input is treated as data in a given context.

* ✅ Useful for **output encoding** in HTML/JS/URL contexts (anti-XSS)
* ⚠️ Do **not** rely on “SQL escaping” as primary defense—use parameterized queries
* ⚠️ Must be **context-aware** (HTML escaping ≠ JS escaping ≠ shell escaping)

---

## Content Security Policy (CSP)

**What it does:** Browser-level policy that restricts where scripts/resources can load from.

**Helps with**

* Reducing impact of XSS (blocks inline scripts, limits allowed domains)

**Good baseline ideas**

* Prefer `script-src 'self'` and **avoid** `'unsafe-inline'`
* Use nonces/hashes for necessary inline scripts
* Lock down `connect-src`, `img-src`, `frame-ancestors`, etc.

**Note:** CSP is a *mitigation*, not a substitute for proper encoding/sanitization.

---

## Quick checklist

* ✅ Validate on **server** (type, length, patterns, allowlists)
* ✅ Use parameterized queries everywhere
* ✅ Encode output by context (HTML/attr/URL/JS)
* ✅ Avoid shells; if needed, no shell + allowlists
* ✅ Never evaluate user input as templates
* ✅ Add CSP + security headers as defense-in-depth
