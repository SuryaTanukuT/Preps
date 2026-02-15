# 6️⃣ Pagination

🎯 **Goal:**
Avoid returning massive datasets.

---

## ❌ Bad

```
GET /users
```

Returns 50k records

---

## ✅ Good

```
GET /users?page=1&limit=20
```

---

# Types of Pagination

| Type         | When Used          |
| ------------ | ------------------ |
| Offset-based | Simple apps        |
| Cursor-based | High-scale systems |
| Keyset       | Better performance |

---

# Recommended Response Format

```json
{
  "data": [...],
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 300
  }
}
```

---

High-scale systems prefer:

```
GET /users?cursor=abc123
```

---

Used in:

* Twitter
* LinkedIn
* Banking dashboards
