# API Gateway Pattern

## Definition
An **API Gateway** is a server that acts as a single entry point for all client requests in a microservices architecture. It routes requests, aggregates responses, handles cross-cutting concerns (authentication, rate limiting, logging), and translates protocols between clients and services.

Unlike BFF (which is client-specific), an API Gateway serves all clients but provides a unified facade to backend services.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │           INTERNET/CLIENTS          │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼───────────────────┐
                    │           API GATEWAY               │
                    │  (Auth, Rate Limit, Routing, Logs)  │
                    └───┬──────────┬──────────┬──────────┘
                        │          │          │
            ┌───────────▼──┐ ┌─────▼─────┐ ┌─▼───────────┐
            │  User Service│ │Order Serv.│ │Product Serv.│
            └──────────────┘ └───────────┘ └─────────────┘
```

---

## Good vs Bad Example

### Good Example
**API Gateway** handles:
- All requests hit `api.example.com`
- Gateway authenticates JWT tokens
- Routes `/users/*` to user service, `/orders/*` to order service
- Rate limits 1000 requests/min per API key
- Logs all requests for auditing
- Returns 504 if service times out

### Bad Example
Clients call services directly:
- Mobile app calls `user.service.example.com`
- Web app calls `user.service.example.com:3001`
- Need to implement auth in every service
- CORS issues everywhere
- Can't track all traffic in one place
- Services exposed directly to internet

---

## Industry Use Cases

### 1. BFSI (Banking)

**Gateway Responsibilities:**
```
https://api.bank.com
├── /v1/accounts/*    → Account Service
├── /v1/transactions/* → Transaction Service  
├── /v1/loans/*       → Loan Service
└── /v1/reports/*     → Reporting Service (rate limited)
```

**Specifics:**
- **PCI Compliance:** All payment routes go through extra encryption
- **Audit Logging:** Every request logged for regulatory compliance
- **IP Whitelisting:** Internal banking tools have IP restrictions
- **Fraud Detection:** Gateway analyzes patterns before routing
- **Request/Response Encryption:** Automatic TLS termination and re-encryption

**Example Flow:**
```
Mobile App → API Gateway (auth + rate limit) → Transaction Service
                ↓
           Fraud Check
                ↓
           Audit Log
```

### 2. Real Estate

**Gateway Responsibilities:**
```
https://api.realestate.com
├── /public/search/*    → Search Service (public, low rate limit)
├── /agent/properties/* → Agent Service (JWT required)
├── /images/*          → Image Service (cached at gateway)
└── /admin/*           → Admin Service (IP whitelisted)
```

**Specifics:**
- **Image Resizing:** Gateway can request different sizes from image service
- **Geocoding Cache:** Cache lat/lng conversions at gateway level
- **MLS Compliance:** Certain endpoints require special headers
- **Public vs Agent:** Different rate limits and response formats
- **WebSocket Upgrade:** Handle real-time viewing connections

**Example Flow:**
```
Agent Web App → API Gateway (JWT + MLS header check) → Property Service
                    ↓
              Cache Check (popular searches)
                    ↓
              Response Transformation (add agent commission data)
```

### 3. E-commerce

**Gateway Responsibilities:**
```
https://api.shop.com
├── /v1/products/*     → Product Service (cached 5min)
├── /v1/cart/*         → Cart Service (session affinity)
├── /v1/checkout/*     → Checkout Service (idempotency keys)
├── /v1/inventory/*    → Inventory Service (real-time)
└── /v1/reviews/*      → Review Service (public write, slow)
```

**Specifics:**
- **Flash Sale Mode:** Dynamic rate limiting during peak events
- **A/B Testing:** Route splitting for experiment groups
- **Mobile vs Desktop:** Response transformation based on User-Agent
- **Idempotency:** Ensure checkout isn't processed twice
- **Circuit Breaking:** If inventory service fails, serve stale cache

**Example Flow:**
```
Mobile App → API Gateway (check rate limit + A/B test group) → Checkout Service
                  ↓
            Idempotency Check (Redis)
                  ↓
            Parallel Calls (inventory + payment + shipping)
                  ↓
            Response Aggregation
```

---

## API Gateway vs BFF

| Aspect | API Gateway | BFF |
|--------|------------|-----|
| **Purpose** | Single entry for all clients | Dedicated backend per client |
| **Clients** | All clients (web, mobile, 3rd party) | One specific client type |
| **Granularity** | Coarse, service-level routing | Fine, UI-specific data shaping |
| **Ownership** | Central platform team | Frontend team |
| **Logic** | Cross-cutting concerns (auth, routing) | UI-specific orchestration |
| **Example** | Kong, AWS API Gateway, Nginx | Custom Node.js service per app |

**Can they coexist? YES:**
```
Client → API Gateway (auth, rate limit) → BFF (data shaping) → Microservices
```

---

## Core Features in Node.js

### 1. Routing & Proxy
```javascript
// Express Gateway Example
app.use('/api/users', authenticate, createProxyMiddleware({
  target: 'http://user-service:3001',
  timeout: 5000
}));

app.use('/api/orders', authenticate, rateLimit('orders'), createProxyMiddleware({
  target: 'http://order-service:3002'
}));
```

### 2. Authentication
```javascript
// JWT Validation at Gateway
const jwt = require('jsonwebtoken');

async function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Add user info to request for downstream services
    req.headers['x-user-id'] = decoded.userId;
    req.headers['x-user-roles'] = decoded.roles.join(',');
    
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}
```

### 3. Rate Limiting
```javascript
// Different limits per route/client
const rateLimit = require('express-rate-limit');

const publicLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 min
  max: 100, // 100 requests
  keyGenerator: (req) => req.ip
});

const authenticatedLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 1000,
  keyGenerator: (req) => req.headers['x-user-id'] // Track by user
});

app.use('/api/public/*', publicLimiter);
app.use('/api/users/*', authenticatedLimiter);
```

### 4. Circuit Breaking
```javascript
// Prevent cascading failures
const circuitBreaker = require('opossum');

const options = {
  timeout: 3000, // If service takes >3s, trigger failure
  errorThresholdPercentage: 50, // Open circuit when 50% fail
  resetTimeout: 30000 // Try again after 30s
};

const userServiceBreaker = new circuitBreaker(
  (req) => proxyToService('user-service', req),
  options
);

app.use('/api/users', async (req, res) => {
  try {
    const result = await userServiceBreaker.fire(req);
    res.json(result);
  } catch (error) {
    // Fallback response
    res.status(503).json({ 
      error: 'User service unavailable',
      fallback: true,
      data: cache.get('popular-users')
    });
  }
});
```

### 5. Request/Response Transformation
```javascript
// Add common headers, remove sensitive data
app.use((req, res, next) => {
  // Add request ID for tracing
  req.headers['x-request-id'] = uuid.v4();
  
  // Remove internal headers before forwarding
  delete req.headers['x-internal-key'];
  delete req.headers['x-debug-info'];
  
  next();
});

// Response transformation
app.use('/api/*', (req, res, next) => {
  const originalJson = res.json;
  
  res.json = function(data) {
    // Add metadata to all responses
    const response = {
      data,
      meta: {
        requestId: req.headers['x-request-id'],
        timestamp: new Date().toISOString(),
        version: 'v1'
      }
    };
    
    // Remove sensitive fields
    if (data.password) delete data.password;
    if (data.ssn) delete data.ssn;
    
    originalJson.call(this, response);
  };
  
  next();
});
```

### 6. Request Aggregation
```javascript
// Combine multiple service calls
app.get('/api/dashboard/:userId', async (req, res) => {
  try {
    // Parallel calls to multiple services
    const [user, orders, recommendations] = await Promise.all([
      fetch('http://user-service/users/' + req.params.userId),
      fetch('http://order-service/orders?userId=' + req.params.userId + '&limit=5'),
      fetch('http://ml-service/recommendations/' + req.params.userId)
    ]);
    
    // Aggregate responses
    res.json({
      profile: user.data,
      recentOrders: orders.data,
      recommendedForYou: recommendations.data
    });
  } catch (error) {
    // Partial response if some services fail
    res.json({
      profile: user?.data || null,
      recentOrders: orders?.data || [],
      recommendations: [],
      partial: true
    });
  }
});
```

### 7. Protocol Translation
```javascript
// REST to GraphQL translation
app.post('/api/graphql', async (req, res) => {
  // Accept REST-like request, convert to GraphQL
  const { query, variables } = restToGraphQL(req.body);
  
  const result = await fetch('http://gql-service/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables })
  });
  
  res.json(result.data);
});

// WebSocket upgrade
app.ws('/api/realtime/orders', (ws, req) => {
  // Authenticate WebSocket connection
  if (!authenticateWs(req)) {
    ws.close(1008, 'Unauthorized');
    return;
  }
  
  // Proxy to appropriate service
  const orderWs = new WebSocket('ws://order-service:3002/realtime');
  
  ws.on('message', (msg) => orderWs.send(msg));
  orderWs.on('message', (msg) => ws.send(msg));
});
```

---

## Popular Node.js Gateway Solutions

| Solution | Type | Best For |
|----------|------|----------|
| **Express Gateway** | Node.js lib | Small-medium apps, custom logic |
| **Kong** | NGINX + Lua | Enterprise, high performance |
| **AWS API Gateway** | Managed | Serverless, AWS ecosystem |
| **Nginx** | Web server | Simple routing, load balancing |
| **Traefik** | Reverse proxy | Kubernetes, dynamic config |
| **GraphQL Mesh** | Node.js lib | GraphQL federation |

---

## When to Use API Gateway

### ✅ DO Use When:
- Multiple microservices behind single domain
- Need centralized authentication/authorization
- Cross-cutting concerns (logging, metrics, rate limiting)
- Multiple client types (mobile, web, third-party)
- Protocol translation needed (REST ↔ GraphQL)
- Need to hide internal service structure

### ❌ DON'T Use When:
- Single monolithic application
- Direct service-to-service communication only
- Small application (< 3 services)
- Development environment
- Heavy real-time bidirectional streaming (use WebSocket gateway instead)

---

## Common Pitfalls

1. **Single Point of Failure** → Deploy multiple gateway instances + load balancer
2. **Performance Bottleneck** → Keep gateway logic minimal, use async
3. **Latency Addition** → Each hop adds 1-5ms, optimize critical paths
4. **Configuration Drift** → Infrastructure as code for gateway config
5. **Debugging Complexity** → Add request IDs, structured logging

---

## Summary Checklist

| Feature | Implementation |
|---------|---------------|
| **Single Entry Point** | All APIs behind `api.example.com` |
| **Authentication** | JWT/OAuth validation at edge |
| **Rate Limiting** | Per user/IP/endpoint |
| **Routing** | Path-based to services |
| **Load Balancing** | Round-robin or least-connections |
| **Caching** | Redis for frequent queries |
| **Circuit Breaking** | Fail fast, fallback responses |
| **Request Tracing** | Correlation IDs |
| **Monitoring** | Metrics, logs, alerts |
| **SSL Termination** | HTTPS at gateway |

**Bottom Line:** API Gateway is the traffic cop of microservices—essential for security, observability, and management, but must be kept lean to avoid becoming a bottleneck.