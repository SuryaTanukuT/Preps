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

