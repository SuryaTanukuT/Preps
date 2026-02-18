# 1️⃣ Access Control (Security First)

🎯 **Goal:**
Ensure only the right users/services can access the right resources.

---

# High-Level Concepts

## ✅ Authentication (Who are you?)

* JWT (stateless)
* OAuth2 / OIDC
* API Keys (for service-to-service)

### Node.js example stack:

* `jsonwebtoken`
* `passport`
* Azure AD / Keycloak integration

---

## ✅ Authorization (What can you do?)
https://dev.to/zenstack/authorize-users-like-a-pro-libraries-that-help-you-implement-access-control-with-nodejs-5109
https://medium.com/@onakoyak/how-to-implement-attribute-based-access-control-abac-in-nestjs-402245193940
* RBAC (Role-Based Access Control)
* ABAC (Attribute-Based Access Control)
* Scope-based access

### Example:

```
GET  /orders   → USER role
POST /orders   → ADMIN role
```

---

## ✅ Best Practices

* Never trust client input
* Validate tokens at middleware layer
* Use centralized auth middleware
* Use least privilege principle

---

# Architecture View

```
Client → API Gateway → Auth Middleware → Business Logic
```

# 9️⃣ Rate Limiting

Prevent abuse.

---

## Node.js

* `express-rate-limit`
* Redis-based distributed limiter

---

## Used in:

* Public APIs
* Fintech systems

---

# 🔟 Observability & Logging

Always include:

* `requestId`
* `traceId`
* Structured logs

---

## Use:

* OpenTelemetry
* ELK
* Prometheus
https://frontegg.com/guides/access-control-in-security

https://pathlock.com/learn/user-access-controls-11-best-practices-for-businesses/