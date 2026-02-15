# 4️⃣ Error Handling

🎯 **Goal:**
Predictable & structured errors.

---

# Use Standard HTTP Codes

| Code | Meaning        |
| ---- | -------------- |
| 200  | OK             |
| 201  | Created        |
| 400  | Bad Request    |
| 401  | Unauthorized   |
| 403  | Forbidden      |
| 404  | Not Found      |
| 409  | Conflict       |
| 500  | Internal Error |

---

# Standard Error Response Format

```json
{
  "error": {
    "code": "ORDER_NOT_FOUND",
    "message": "Order does not exist",
    "traceId": "xyz-123"
  }
}
```

---

# Node.js Best Practice

* Centralized error middleware
* Don’t leak stack traces in production
* Add `correlationId` for observability
