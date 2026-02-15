# 3️⃣ Idempotency

🎯 **Goal:**
Multiple identical requests should produce the same result.

---

## Critical for:

* Payments
* Order creation
* Distributed systems

---

## Example

```
POST /payments
Idempotency-Key: abc123
```

If retried → should not charge twice.

---

## Node.js Strategy

* Store idempotency key in DB or Redis
* Return stored response for duplicates

---

Used heavily in:

* BFSI systems
* Stripe-like APIs
