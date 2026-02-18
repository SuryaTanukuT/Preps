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


https://www.mulesoft.com/api/design/api-pagination-patterns
https://apisyouwonthate.com/blog/api-design-basics-pagination/
https://dev.to/pragativerma18/unlocking-the-power-of-api-pagination-best-practices-and-strategies-4b49
https://medium.com/@khanshahid9283/mastering-api-pagination-best-practices-for-performance-scalability-ca16980bc8f0