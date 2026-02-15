# 7️⃣ Versioning

🎯 **Goal:**
Avoid breaking clients.

---

# Methods

| Method            | Example                  |
| ----------------- | ------------------------ |
| URL Versioning    | `/v1/users`              |
| Header Versioning | `Accept: application/v2` |
| Query Param       | `/users?version=2`       |

---

# Recommended

URL versioning (most common)

```
/api/v1/orders
/api/v2/orders
```

---

# Node.js Structure

```
routes/
 ├── v1/
 └── v2/
```


# 8️⃣ Consistency & Naming

---

## Use nouns, not verbs

❌ `/getUsers`
✅ `/users`

---

## Use proper HTTP methods

| Method | Meaning        |
| ------ | -------------- |
| GET    | Read           |
| POST   | Create         |
| PUT    | Replace        |
| PATCH  | Partial update |
| DELETE | Remove         |
