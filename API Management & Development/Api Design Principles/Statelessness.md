# 2️⃣ Statelessness

🎯 **Goal:**
Each request should contain all required information.

---

# Why?

* Enables horizontal scaling
* Works perfectly with load balancers
* No server memory dependency

---

## ❌ Bad

* Storing session in memory

---

## ✅ Good

JWT carries:

* `userId`
* `roles`
* `expiry`

---

## Node.js

* Don’t store user session in memory
* Use Redis only if absolutely needed

---

Stateless = Cloud friendly ☁️
