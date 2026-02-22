# Request-Response Pattern

## Definition
The **Request-Response** is a fundamental message exchange pattern where a client sends a request message to a server, and the server processes it and sends back a response. It's the most common pattern in distributed systems, forming the basis of web services, RPC, and database queries.

This pattern is inherently **synchronous** and **tightly coupled** in time—the client waits for the response.

---

## Core Concept
```
Client                  Server
  |                       |
  |----REQUEST----------->|
  |                       |-- Process
  |<---RESPONSE-----------|
  |                       |
```

**Characteristics:**
- **Synchronous:** Client blocks until response received
- **Addressable:** Client knows server endpoint
- **Correlated:** Response matches specific request
- **Stateless or Stateful:** Depends on implementation

---

## Good vs Bad Example

### Good Example
**RESTful API with proper status codes:**
```http
POST /api/orders HTTP/1.1
Content-Type: application/json

{
  "productId": "123",
  "quantity": 2
}

---
HTTP/1.1 201 Created
Location: /api/orders/456
Content-Type: application/json

{
  "orderId": "456",
  "status": "confirmed",
  "total": 49.99
}
```
- Clear semantics (POST creates)
- Proper status code (201 Created)
- Location header for new resource
- Consistent response structure

### Bad Example
**RPC-style with mixed semantics:**
```http
GET /api/createOrder?product=123&qty=2 HTTP/1.1

---
HTTP/1.1 200 OK
Content-Type: text/html

Order created successfully with ID 456
```
- GET used for state change (violates HTTP semantics)
- No proper status code (should be 201)
- HTML response for API (should be JSON)
- No resource location returned

---

## Industry Use Cases

### 1. BFSI (Banking)

**Common Request-Response Flows:**

| Endpoint | Request | Response | Characteristics |
|----------|---------|----------|-----------------|
| `POST /api/accounts/verify` | Account number, routing number | Verified status, account holder name | Idempotent, cached |
| `POST /api/transfers` | From account, to account, amount | Transfer ID, status, reference number | Idempotency key required |
| `GET /api/balance/{accountId}` | Account ID | Current balance, available balance | Cached short-term |
| `POST /api/auth/login` | Username, password, MFA token | Session token, expiry, permissions | Rate limited |

**Banking-Specific Requirements:**
```javascript
// Idempotency for money transfers
app.post('/api/transfers', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  
  // Check if already processed
  const existing = await redis.get(`transfer:${idempotencyKey}`);
  if (existing) {
    return res.status(200).json(JSON.parse(existing));
  }
  
  // Process transfer (database transaction)
  const result = await processTransfer(req.body);
  
  // Store result with idempotency key
  await redis.setex(`transfer:${idempotencyKey}`, 86400, JSON.stringify(result));
  
  res.status(201).json(result);
});

// Balance check with caching
app.get('/api/balance/:accountId', async (req, res) => {
  const cacheKey = `balance:${req.params.accountId}`;
  
  // Try cache first (5 second TTL for balance)
  const cached = await redis.get(cacheKey);
  if (cached) {
    res.set('X-Cache', 'HIT');
    return res.json(JSON.parse(cached));
  }
  
  // Fresh from database
  const balance = await db.accounts.getBalance(req.params.accountId);
  await redis.setex(cacheKey, 5, JSON.stringify(balance));
  
  res.set('X-Cache', 'MISS');
  res.json(balance);
});
```

**Critical Considerations for BFSI:**
- **Idempotency:** Prevent duplicate transactions
- **Audit Trail:** Every request logged
- **Rate Limiting:** Prevent abuse
- **Encryption:** TLS everywhere
- **Timeouts:** Banking systems may take seconds

### 2. Real Estate

**Common Request-Response Flows:**

| Endpoint | Request | Response | Characteristics |
|----------|---------|----------|-----------------|
| `GET /api/properties/search` | Lat, lng, radius, filters | Paginated property list | Cached, complex queries |
| `GET /api/properties/{id}` | Property ID | Full property details | Rich media included |
| `POST /api/showings/request` | Property ID, time, agent | Confirmation, calendar invite | Email confirmation async |
| `GET /api/market-stats/{zip}` | Zip code | Median price, inventory, days on market | Pre-computed, cached |

**Real Estate Implementation:**
```javascript
// Complex search with filtering
app.get('/api/properties/search', async (req, res) => {
  const {
    lat, lng, radius = 5,
    minPrice, maxPrice,
    beds, baths,
    propertyType,
    page = 1, limit = 20
  } = req.query;
  
  // Generate cache key from all params
  const cacheKey = `search:${hash(req.query)}`;
  
  // Check cache (5 minute TTL for searches)
  const cached = await redis.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }
  
  // Geospatial query
  const properties = await db.query(`
    SELECT *, ST_Distance(location, ST_MakePoint($1, $2)) as distance
    FROM properties
    WHERE ST_DWithin(location, ST_MakePoint($1, $2), $3)
    AND price BETWEEN $4 AND $5
    AND bedrooms >= $6
    LIMIT $7 OFFSET $8
  `, [lng, lat, radius * 1000, minPrice, maxPrice, beds, limit, (page-1)*limit]);
  
  const result = {
    properties: properties.rows,
    total: properties.count,
    page,
    totalPages: Math.ceil(properties.count / limit)
  };
  
  // Cache search results
  await redis.setex(cacheKey, 300, JSON.stringify(result));
  
  res.json(result);
});

// Property details with aggregation
app.get('/api/properties/:id', async (req, res) => {
  const { id } = req.params;
  const { include = 'basic' } = req.query;
  
  // Parallel fetching of property data
  const promises = {
    basic: db.properties.findById(id)
  };
  
  if (include.includes('images')) {
    promises.images = db.images.findByProperty(id);
  }
  
  if (include.includes('similar')) {
    promises.similar = db.properties.findSimilar(id, 5);
  }
  
  if (include.includes('history')) {
    promises.history = db.priceHistory.findByProperty(id);
  }
  
  const result = await Promise.props(promises);
  
  res.json(result);
});
```

**Considerations for Real Estate:**
- **Geospatial Indexing:** Fast location queries
- **Image Optimization:** Return different sizes based on client
- **Partial Responses:** Client requests only needed fields
- **Caching Strategy:** Popular searches cached longer

### 3. E-commerce

**Common Request-Response Flows:**

| Endpoint | Request | Response | Characteristics |
|----------|---------|----------|-----------------|
| `GET /api/products` | Category, sort, page | Product list with thumbnails | Heavy caching |
| `GET /api/products/{id}` | Product ID | Full product details | Cache with invalidation |
| `POST /api/cart` | Product ID, quantity | Cart summary | Session-based |
| `POST /api/checkout` | Cart ID, payment info | Order confirmation | Idempotent, async steps |
| `GET /api/orders/{id}` | Order ID | Order status, tracking | Real-time updates |

**E-commerce Implementation:**
```javascript
// Product listing with caching
app.get('/api/products', async (req, res) => {
  const { category, sort = 'popular', page = 1 } = req.query;
  
  // Cache product lists aggressively
  const cacheKey = `products:${category}:${sort}:${page}`;
  const cached = await redis.get(cacheKey);
  
  if (cached) {
    res.set('X-Cache', 'HIT');
    return res.json(JSON.parse(cached));
  }
  
  const products = await db.products.findByCategory(category, { sort, page });
  
  // Cache for 5 minutes (products don't change often)
  await redis.setex(cacheKey, 300, JSON.stringify(products));
  
  res.set('X-Cache', 'MISS');
  res.json(products);
});

// Cart operations (session-based)
app.post('/api/cart', async (req, res) => {
  const sessionId = req.cookies.sessionId;
  const { productId, quantity } = req.body;
  
  // Get or create cart from Redis (fast!)
  let cart = await redis.get(`cart:${sessionId}`) || { items: [] };
  
  // Update cart
  const existingItem = cart.items.find(i => i.productId === productId);
  if (existingItem) {
    existingItem.quantity += quantity;
  } else {
    cart.items.push({ productId, quantity });
  }
  
  // Get current prices from cache or DB
  cart.items = await enrichWithPrices(cart.items);
  cart.total = cart.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  
  // Store back in Redis
  await redis.setex(`cart:${sessionId}`, 3600, JSON.stringify(cart));
  
  res.json(cart);
});

// Checkout with idempotency
app.post('/api/checkout', async (req, res) => {
  const idempotencyKey = req.headers['idempotency-key'];
  const sessionId = req.cookies.sessionId;
  
  // Check for duplicate
  const existing = await redis.get(`checkout:${idempotencyKey}`);
  if (existing) {
    return res.json(JSON.parse(existing));
  }
  
  // Get cart
  const cart = await redis.get(`cart:${sessionId}`);
  if (!cart || cart.items.length === 0) {
    return res.status(400).json({ error: 'Cart is empty' });
  }
  
  // Process order (this might take time)
  const order = await processOrder(cart, req.body);
  
  // Clear cart
  await redis.del(`cart:${sessionId}`);
  
  // Store result with idempotency key
  await redis.setex(`checkout:${idempotencyKey}`, 86400, JSON.stringify(order));
  
  // Return immediately, further processing async
  res.status(202).json({
    orderId: order.id,
    status: 'processing',
    estimatedCompletion: Date.now() + 30000
  });
});
```

**Considerations for E-commerce:**
- **Read-Heavy:** Cache product catalogs aggressively
- **Write Consistency:** Cart updates need atomic operations
- **Session Management:** Redis for fast cart access
- **Inventory Check:** Real-time stock verification
- **Partial Success:** Handle payment success but inventory failure

---

## Request-Response Protocols

### 1. HTTP/REST
```javascript
// Express.js REST API
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await db.users.findById(req.params.id);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }
    
    // HATEOAS links
    user._links = {
      self: { href: `/api/users/${user.id}` },
      orders: { href: `/api/users/${user.id}/orders` },
      update: { href: `/api/users/${user.id}`, method: 'PUT' }
    };
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

### 2. GraphQL
```javascript
// Apollo Server
const typeDefs = `
  type Query {
    user(id: ID!): User
    users(limit: Int): [User]
  }
  
  type User {
    id: ID!
    name: String!
    email: String!
    orders(limit: Int): [Order]
  }
`;

const resolvers = {
  Query: {
    user: async (_, { id }) => {
      return db.users.findById(id);
    }
  },
  User: {
    orders: async (user, { limit }) => {
      return db.orders.findByUser(user.id, { limit });
    }
  }
};
```

### 3. gRPC
```protobuf
// user.proto
service UserService {
  rpc GetUser (GetUserRequest) returns (User) {}
  rpc CreateUser (CreateUserRequest) returns (User) {}
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse) {}
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
}
```

```javascript
// Node.js gRPC server
const grpc = require('@grpc/grpc-js');

function getUser(call, callback) {
  const userId = call.request.user_id;
  
  db.users.findById(userId)
    .then(user => callback(null, user))
    .catch(err => callback({
      code: grpc.status.NOT_FOUND,
      message: 'User not found'
    }));
}
```

---

## Advanced Patterns

### 1. Request-Response with Async Processing
```javascript
// Pattern: Return immediately, process async
app.post('/api/reports/generate', async (req, res) => {
  const reportId = uuid.v4();
  
  // Queue for processing
  await queue.add('report-generation', {
    reportId,
    params: req.body,
    userId: req.user.id
  });
  
  // Return 202 Accepted with status URL
  res.status(202).json({
    reportId,
    status: 'processing',
    statusUrl: `/api/reports/status/${reportId}`,
    estimatedCompletion: Date.now() + 60000
  });
});

// Status check endpoint
app.get('/api/reports/status/:reportId', async (req, res) => {
  const status = await redis.get(`report:${req.params.reportId}:status`);
  
  if (status === 'completed') {
    res.json({
      status: 'completed',
      downloadUrl: `/api/reports/download/${req.params.reportId}`
    });
  } else if (status === 'failed') {
    res.json({
      status: 'failed',
      error: 'Report generation failed'
    });
  } else {
    res.json({
      status: 'processing',
      progress: await redis.get(`report:${req.params.reportId}:progress`)
    });
  }
});
```

### 2. Long Polling
```javascript
// Client waits for server to complete
app.get('/api/long-poll/:jobId', async (req, res) => {
  const jobId = req.params.jobId;
  const startTime = Date.now();
  const timeout = 30000; // 30 seconds
  
  // Poll for completion
  while (Date.now() - startTime < timeout) {
    const result = await redis.get(`job:${jobId}:result`);
    
    if (result) {
      return res.json(JSON.parse(result));
    }
    
    // Wait 1 second before next check
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
  
  // Timeout - client should try again
  res.status(204).send();
});
```

### 3. Request Pipeline
```javascript
// Chain multiple requests
app.get('/api/user-dashboard/:userId', async (req, res) => {
  // Pipeline: user -> orders -> recommendations
  const user = await db.users.findById(req.params.userId);
  
  const orders = await db.orders.findByUser(user.id, { limit: 5 });
  
  const productIds = orders.flatMap(o => o.items.map(i => i.productId));
  const recommendations = await db.products.findRecommendations(productIds);
  
  res.json({
    user,
    recentOrders: orders,
    recommendations
  });
});
```

### 4. Conditional Requests
```javascript
// HTTP conditional requests for caching
app.get('/api/products/:id', async (req, res) => {
  const product = await db.products.findById(req.params.id);
  
  // Check If-Modified-Since
  const ifModifiedSince = req.headers['if-modified-since'];
  if (ifModifiedSince && new Date(product.updatedAt) <= new Date(ifModifiedSince)) {
    return res.status(304).send(); // Not Modified
  }
  
  // Check ETag
  const etag = `"${hash(JSON.stringify(product))}"`;
  if (req.headers['if-none-match'] === etag) {
    return res.status(304).send();
  }
  
  res.set({
    'Last-Modified': product.updatedAt.toUTCString(),
    'ETag': etag,
    'Cache-Control': 'private, max-age=60'
  });
  
  res.json(product);
});
```

---

## Best Practices

### 1. Consistent Response Structure
```javascript
// Standard API response wrapper
class ApiResponse {
  static success(data, meta = {}) {
    return {
      success: true,
      data,
      meta: {
        timestamp: new Date().toISOString(),
        ...meta
      },
      error: null
    };
  }
  
  static error(message, code = 'INTERNAL_ERROR', details = null) {
    return {
      success: false,
      data: null,
      error: {
        code,
        message,
        details,
        timestamp: new Date().toISOString()
      }
    };
  }
}

// Usage
app.get('/api/users/:id', async (req, res) => {
  try {
    const user = await db.users.findById(req.params.id);
    res.json(ApiResponse.success(user));
  } catch (error) {
    res.status(404).json(ApiResponse.error(
      'User not found',
      'USER_NOT_FOUND',
      { userId: req.params.id }
    ));
  }
});
```

### 2. Proper HTTP Status Codes
```javascript
// 200: Success
// 201: Created (POST)
// 202: Accepted (async processing)
// 204: No Content (DELETE success)
// 304: Not Modified (conditional GET)
// 400: Bad Request (validation error)
// 401: Unauthorized (no auth)
// 403: Forbidden (auth but no permission)
// 404: Not Found
// 409: Conflict (duplicate, version mismatch)
// 422: Unprocessable Entity (semantic error)
// 429: Too Many Requests
// 500: Internal Server Error
// 503: Service Unavailable
```

### 3. Request Validation
```javascript
const Joi = require('joi');

const createUserSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
  name: Joi.string().required(),
  age: Joi.number().min(18).max(120)
});

function validate(schema) {
  return (req, res, next) => {
    const { error } = schema.validate(req.body);
    
    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }
    
    next();
  };
}

app.post('/api/users', validate(createUserSchema), createUser);
```

### 4. Pagination
```javascript
app.get('/api/products', async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit;
  
  const [items, total] = await Promise.all([
    db.products.find({ limit, offset }),
    db.products.count()
  ]);
  
  res.json({
    items,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit),
      hasNext: offset + limit < total,
      hasPrev: page > 1
    },
    links: {
      next: page < Math.ceil(total / limit) ? `/api/products?page=${page + 1}&limit=${limit}` : null,
      prev: page > 1 ? `/api/products?page=${page - 1}&limit=${limit}` : null
    }
  });
});
```

---

## Performance Optimization

### 1. Response Compression
```javascript
const compression = require('compression');
app.use(compression({
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  },
  level: 6 // balance between speed and compression
}));
```

### 2. Partial Responses (Fields Filtering)
```javascript
app.get('/api/users/:id', async (req, res) => {
  const user = await db.users.findById(req.params.id);
  
  // Fields filtering: ?fields=id,name,email
  if (req.query.fields) {
    const fields = req.query.fields.split(',');
    const filtered = {};
    fields.forEach(field => {
      if (user[field] !== undefined) {
        filtered[field] = user[field];
      }
    });
    return res.json(filtered);
  }
  
  res.json(user);
});
```

### 3. Response Caching Headers
```javascript
app.get('/api/products/:id', async (req, res) => {
  const product = await db.products.findById(req.params.id);
  
  // Cache public products for 5 minutes
  res.set({
    'Cache-Control': 'public, max-age=300',
    'ETag': `"${product.version}"`,
    'Last-Modified': product.updatedAt.toUTCString()
  });
  
  res.json(product);
});
```

---

## Request-Response vs Other Patterns

| Pattern | Communication | Use Case |
|---------|--------------|----------|
| **Request-Response** | Synchronous | CRUD APIs, queries |
| **Pub/Sub** | Asynchronous, one-to-many | Events, notifications |
| **Streaming** | Continuous | Real-time data |
| **Callback** | Async with callback URL | Long-running processes |
| **Fire-and-Forget** | One-way | Logging, analytics |

---

## When to Use Request-Response

### ✅ DO Use When:
- Immediate response needed
- Querying data
- CRUD operations
- Simple service-to-service calls
- User-initiated actions

### ❌ DON'T Use When:
- Multiple consumers needed (use Pub/Sub)
- Long-running processes (use async with status)
- Real-time streaming (use WebSockets)
- One-way notifications (use fire-and-forget)

---

## Summary Checklist

| Aspect | Best Practice |
|--------|--------------|
| **Method** | REST for web, gRPC for internal |
| **Status Codes** | Use appropriate HTTP status codes |
| **Idempotency** | Required for mutating operations |
| **Validation** | Validate all inputs |
| **Error Handling** | Consistent error structure |
| **Caching** | Cache where appropriate |
| **Pagination** | Always paginate lists |
| **Rate Limiting** | Protect against abuse |
| **Monitoring** | Track latency and errors |

**Bottom Line:** Request-Response is the foundation of web APIs. Keep responses predictable, use proper status codes, and optimize for the 90% case while handling errors gracefully.