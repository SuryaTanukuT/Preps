
---

# 🔷 1️⃣ GET – Read Data

## 🎯 Purpose:

Retrieve data from server.

### Example:

```http
GET /users
GET /users/101
```

### Characteristics:

* Safe (doesn’t modify data)
* Idempotent
* Cacheable
* No request body (usually)

### Node.js Example:

```js
app.get('/users/:id', controller.getUser);
```

### Used For:

* Fetch dashboards
* Retrieve reports
* Load profile data

---

# 🔷 2️⃣ POST – Create Resource

## 🎯 Purpose:

Create a new resource.

### Example:

```http
POST /users
POST /orders
```

### Characteristics:

* Not idempotent (calling twice creates 2 records)
* Has request body
* Returns `201 Created`

### Node.js Example:

```js
app.post('/orders', controller.createOrder);
```

### Used For:

* Creating payments
* User registration
* Placing an order

---

# 🔷 3️⃣ PUT – Replace Resource

## 🎯 Purpose:

Update entire resource.

### Example:

```http
PUT /users/101
```

### Characteristics:

* Idempotent
* Replaces whole object
* Missing fields may be overwritten

### Node.js Example:

```js
app.put('/users/:id', controller.updateUser);
```

---

# 🔷 4️⃣ PATCH – Partial Update

## 🎯 Purpose:

Update specific fields only.

### Example:

```http
PATCH /users/101
```

Body:

```json
{
  "email": "new@email.com"
}
```

### Characteristics:

* Idempotent (usually)
* Partial modification
* More efficient than PUT

### Node.js Example:

```js
app.patch('/users/:id', controller.patchUser);
```

---

# 🔷 5️⃣ DELETE – Remove Resource

## 🎯 Purpose:

Delete resource.

### Example:

```http
DELETE /users/101
```

### Characteristics:

* Idempotent (deleting again doesn’t change result)
* Returns `204 No Content`

### Node.js Example:

```js
app.delete('/users/:id', controller.deleteUser);
```

---

# 🔷 6️⃣ HEAD – Metadata Only

## 🎯 Purpose:

Like GET but without body.

Used for:

* Checking if resource exists
* Checking content length

Example:

```http
HEAD /users/101
```

Rare in normal apps but used in:

* CDN
* File servers

---

# 🔷 7️⃣ OPTIONS – Allowed Methods

## 🎯 Purpose:

Tells which methods are allowed.

Used for:

* CORS preflight requests

Example:

```http
OPTIONS /users
```

Node.js handles automatically via CORS middleware.

---

# 🔷 Safe vs Idempotent vs Unsafe

| Method | Safe | Idempotent  |
| ------ | ---- | ----------- |
| GET    | ✅    | ✅           |
| POST   | ❌    | ❌           |
| PUT    | ❌    | ✅           |
| PATCH  | ❌    | ✅ (usually) |
| DELETE | ❌    | ✅           |

---

# 🔷 RESTful Best Practice

Use nouns, not verbs.

❌ `/getUsers`
✅ `/users`

Combine with method:

```
GET    /users
POST   /users
GET    /users/:id
PUT    /users/:id
PATCH  /users/:id
DELETE /users/:id
```

---

# 🔷 Real Production Example (Payments API)

```
POST   /payments
GET    /payments/:id
PATCH  /payments/:id/cancel
```

Notice:

* Resource oriented
* Clean
* Predictable

---

# 🔷 Interview-Level Answer

If interviewer asks:

> “Explain HTTP methods in REST API”

You can say:

> In RESTful API design, we use standard HTTP methods to represent operations on resources. GET retrieves data, POST creates resources, PUT replaces entire resources, PATCH updates partially, and DELETE removes resources. We ensure correct use of idempotency and HTTP status codes to make APIs predictable, scalable, and cache-friendly.

---

# 🔥 Senior-Level Insight

In high-scale systems:

* GET → Often cached
* POST → Needs idempotency key
* PUT/PATCH → Optimistic locking
* DELETE → Soft delete in production
