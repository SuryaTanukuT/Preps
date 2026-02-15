# 5️⃣ Caching

🎯 **Goal:**
Improve performance & reduce DB load.

---

# Types

## 1️⃣ Client Caching

* ETag
* Cache-Control headers

---

## 2️⃣ API Level

* Redis
* In-memory (short lived)

---

## 3️⃣ CDN Level

* CloudFront
* Akamai

---

# Node.js Pattern

```
Check Redis →
If exists → return
Else → fetch DB → store in Redis → return
```

---

⚠ Never cache:

* Sensitive data
* Highly dynamic payment states
