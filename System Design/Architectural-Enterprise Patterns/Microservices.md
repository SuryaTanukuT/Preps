# Microservices Architecture

## Definition
**Microservices Architecture** is an architectural style that structures an application as a collection of small, autonomous, independently deployable services. Each service is:
- **Focused**: Built around a specific business capability
- **Independent**: Can be developed, deployed, and scaled independently
- **Decentralized**: Each service has its own data storage and technology stack
- **Communicating**: Services interact via well-defined APIs (usually HTTP/REST or messaging)

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │         API GATEWAY                 │
                    │  (Single Entry Point)               │
                    └──────┬──────────┬──────────┬───────┘
                           │          │          │
              ┌────────────▼──┐ ┌─────▼─────┐ ┌─▼────────────┐
              │   Order       │ │  Payment  │ │  Inventory   │
              │   Service     │ │  Service  │ │   Service    │
              │   (Node.js)   │ │  (Python) │ │   (Go)       │
              │   ┌────────┐  │ │  ┌──────┐ │ │   ┌──────┐  │
              │   │Postgres│  │ │  │Mongo │ │ │   │Redis │  │
              │   └────────┘  │ │  └──────┘ │ │   └──────┘  │
              └───────────────┘ └───────────┘ └─────────────┘
                           │          │          │
                           └──────────┼──────────┘
                                     │
                    ┌────────────────▼─────────────────────┐
                    │      Message Broker (Kafka)          │
                    │  (Event-Driven Communication)        │
                    └───────────────────────────────────────┘
```

---

## Monolith vs Microservices

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Deployment** | Single unit | Independent services |
| **Scaling** | Scale entire app | Scale individual services |
| **Development** | Single codebase | Multiple codebases |
| **Team Structure** | Feature teams | Service teams |
| **Database** | Single database | Database per service |
| **Technology** | Uniform stack | Polyglot (varied stacks) |
| **Communication** | In-method calls | Network calls |
| **Complexity** | Lower development | Higher operations |
| **Fault Isolation** | Failure affects all | Failure isolated |

---

## Good vs Bad Example

### Bad Example (Mini-Monoliths)
```javascript
// BAD: Services that aren't truly independent

// order-service/index.js - Has payment logic
app.post('/api/orders', async (req, res) => {
  // Order logic
  const order = await db.orders.create(req.body);
  
  // Payment logic (should be separate)
  if (req.body.payment.type === 'credit_card') {
    await axios.post('https://payment-processor.com/charge', {
      amount: order.total,
      card: req.body.payment.details
    });
  }
  
  // Inventory logic (should be separate)
  for (const item of order.items) {
    await db.products.updateStock(item.productId, -item.quantity);
  }
  
  // Email logic (should be separate)
  await email.send(order.userId, 'Order confirmed');
  
  res.json(order);
});

// Problems:
// - All services share same database
// - Can't scale independently
// - Changes in payment affect order service
// - One bug brings down everything
```

### Good Example (True Microservices)
```javascript
// ============ ORDER SERVICE ============
// services/order-service/src/index.js
app.post('/api/orders', async (req, res) => {
  // 1. Create order (own database)
  const order = await orderRepository.create({
    userId: req.body.userId,
    items: req.body.items,
    status: 'PENDING'
  });
  
  // 2. Emit event (not direct calls)
  await eventBus.publish('order.created', {
    orderId: order.id,
    userId: order.userId,
    items: order.items,
    total: order.total
  });
  
  res.status(202).json({ 
    orderId: order.id, 
    status: 'processing' 
  });
});

// ============ PAYMENT SERVICE ============
// services/payment-service/src/index.js
eventBus.subscribe('order.created', async (event) => {
  // Get payment details from user service
  const paymentMethod = await userService.getDefaultPayment(event.userId);
  
  // Process payment (own logic, own database)
  const payment = await paymentGateway.charge({
    amount: event.total,
    method: paymentMethod
  });
  
  // Emit result
  await eventBus.publish('payment.processed', {
    orderId: event.orderId,
    paymentId: payment.id,
    status: 'success'
  });
});

// ============ INVENTORY SERVICE ============
// services/inventory-service/src/index.js
eventBus.subscribe('order.created', async (event) => {
  // Reserve inventory (own database)
  for (const item of event.items) {
    await inventoryRepository.reserve(item.productId, item.quantity);
  }
});

// ============ EMAIL SERVICE ============
// services/email-service/src/index.js
eventBus.subscribe('payment.processed', async (event) => {
  await emailClient.send({
    userId: event.userId,
    template: 'order-confirmation',
    data: event
  });
});
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Core Banking System

```javascript
// ============ SERVICE ARCHITECTURE ============
const services = {
  'account-service': {
    port: 3001,
    database: 'postgres-accounts',
    responsibilities: ['account management', 'balances', 'statements']
  },
  'transaction-service': {
    port: 3002,
    database: 'postgres-transactions',
    responsibilities: ['payments', 'transfers', 'ledger']
  },
  'customer-service': {
    port: 3003,
    database: 'mongodb-customers',
    responsibilities: ['customer data', 'KYC', 'preferences']
  },
  'loan-service': {
    port: 3004,
    database: 'postgres-loans',
    responsibilities: ['loan applications', 'underwriting', 'amortization']
  },
  'fraud-service': {
    port: 3005,
    database: 'redis-fraud',
    responsibilities: ['fraud detection', 'risk scoring', 'alerts']
  },
  'notification-service': {
    port: 3006,
    database: 'mongodb-notifications',
    responsibilities: ['email', 'SMS', 'push']
  },
  'reporting-service': {
    port: 3007,
    database: 'clickhouse-analytics',
    responsibilities: ['reports', 'analytics', 'audit']
  }
};

// ============ ACCOUNT SERVICE ============
// services/account-service/src/index.js
const express = require('express');
const app = express();

// Own database connection
const db = require('./database');

// Own models
const Account = require('./models/Account');
const Balance = require('./models/Balance');

// Own API endpoints
app.get('/api/accounts/:id', async (req, res) => {
  const account = await Account.findById(req.params.id);
  res.json(account);
});

app.post('/api/accounts', async (req, res) => {
  const account = await Account.create(req.body);
  
  // Emit event for other services
  await eventBus.publish('account.created', {
    accountId: account.id,
    customerId: account.customerId,
    type: account.type
  });
  
  res.status(201).json(account);
});

app.get('/api/accounts/:id/balance', async (req, res) => {
  const balance = await Balance.getCurrent(req.params.id);
  res.json(balance);
});

// Listen for events from other services
eventBus.subscribe('transaction.completed', async (event) => {
  // Update account balance
  await Balance.update(event.accountId, event.amount);
});

// ============ TRANSACTION SERVICE ============
// services/transaction-service/src/index.js
class TransactionService {
  async transfer(fromAccount, toAccount, amount) {
    // Start distributed transaction
    const transactionId = uuid();
    
    // 1. Validate (call account service)
    const fromValid = await this.validateAccount(fromAccount, amount);
    const toValid = await this.validateAccount(toAccount);
    
    if (!fromValid || !toValid) {
      throw new Error('Invalid accounts');
    }
    
    // 2. Debit from account (call account service)
    await this.debitAccount(fromAccount, amount, transactionId);
    
    // 3. Credit to account (call account service)
    await this.creditAccount(toAccount, amount, transactionId);
    
    // 4. Record transaction (own database)
    const transaction = await this.recordTransaction({
      id: transactionId,
      fromAccount,
      toAccount,
      amount,
      status: 'COMPLETED'
    });
    
    // 5. Emit event
    await eventBus.publish('transaction.completed', {
      transactionId,
      fromAccount,
      toAccount,
      amount
    });
    
    // 6. Trigger fraud check (async)
    await eventBus.publish('fraud.check.required', {
      transactionId,
      fromAccount,
      amount,
      timestamp: new Date()
    });
    
    return transaction;
  }
  
  async validateAccount(accountId, amount = 0) {
    try {
      const response = await axios.get(
        `${SERVICE_URLS.account}/api/accounts/${accountId}/validate`,
        { params: { amount } }
      );
      return response.data.valid;
    } catch (error) {
      console.error('Account validation failed:', error);
      return false;
    }
  }
  
  async debitAccount(accountId, amount, transactionId) {
    await axios.post(
      `${SERVICE_URLS.account}/api/accounts/${accountId}/debit`,
      { amount, transactionId }
    );
  }
  
  async creditAccount(accountId, amount, transactionId) {
    await axios.post(
      `${SERVICE_URLS.account}/api/accounts/${accountId}/credit`,
      { amount, transactionId }
    );
  }
}

// ============ FRAUD SERVICE ============
// services/fraud-service/src/index.js
class FraudService {
  constructor() {
    this.rules = [
      new VelocityCheck(),
      new LocationAnomaly(),
      new AmountThreshold(),
      new PatternMatching()
    ];
    
    eventBus.subscribe('transaction.completed', this.analyze.bind(this));
  }
  
  async analyze(event) {
    const riskScore = await this.calculateRisk(event);
    
    if (riskScore > 80) {
      await eventBus.publish('fraud.high.risk', {
        transactionId: event.transactionId,
        riskScore,
        actions: ['BLOCK', 'FLAG', 'NOTIFY']
      });
    } else if (riskScore > 50) {
      await eventBus.publish('fraud.medium.risk', {
        transactionId: event.transactionId,
        riskScore,
        actions: ['FLAG', 'REVIEW']
      });
    }
  }
  
  async calculateRisk(event) {
    let score = 0;
    
    for (const rule of this.rules) {
      score += await rule.evaluate(event);
    }
    
    return Math.min(score, 100);
  }
}

// ============ NOTIFICATION SERVICE ============
// services/notification-service/src/index.js
class NotificationService {
  constructor() {
    eventBus.subscribe('fraud.high.risk', this.notifySecurity.bind(this));
    eventBus.subscribe('transaction.completed', this.notifyCustomer.bind(this));
    eventBus.subscribe('account.created', this.welcomeEmail.bind(this));
  }
  
  async notifySecurity(event) {
    await this.sendSMS({
      to: process.env.SECURITY_PHONE,
      message: `High risk transaction detected: ${event.transactionId}`,
      priority: 'HIGH'
    });
    
    await this.createTicket({
      system: 'fraud',
      transactionId: event.transactionId,
      riskScore: event.riskScore,
      assignedTo: 'fraud-team'
    });
  }
  
  async notifyCustomer(event) {
    const customer = await this.getCustomer(event.fromAccount);
    
    await this.sendEmail({
      to: customer.email,
      template: 'transaction-alert',
      data: {
        amount: event.amount,
        date: new Date().toLocaleString(),
        type: 'debit'
      }
    });
  }
}
```

### 2. E-commerce - Complete Platform

```javascript
// ============ SERVICE BOUNDARIES ============
const serviceBoundaries = {
  'product-catalog': {
    domain: 'Products',
    capabilities: ['CRUD products', 'categories', 'search', 'inventory views'],
    tech: 'Node.js + Elasticsearch',
    database: 'MongoDB'
  },
  'shopping-cart': {
    domain: 'Cart',
    capabilities: ['add/remove items', 'cart calculations', 'persistence'],
    tech: 'Node.js + Redis',
    database: 'Redis'
  },
  'order-management': {
    domain: 'Orders',
    capabilities: ['order creation', 'order status', 'history'],
    tech: 'Node.js + PostgreSQL',
    database: 'PostgreSQL'
  },
  'payment': {
    domain: 'Payments',
    capabilities: ['payment processing', 'refunds', 'payment methods'],
    tech: 'Python + Flask',
    database: 'PostgreSQL'
  },
  'inventory': {
    domain: 'Stock',
    capabilities: ['stock levels', 'reservations', 'reorder alerts'],
    tech: 'Go',
    database: 'PostgreSQL'
  },
  'shipping': {
    domain: 'Fulfillment',
    capabilities: ['rate calculation', 'label generation', 'tracking'],
    tech: 'Node.js',
    database: 'MongoDB'
  },
  'user': {
    domain: 'Customers',
    capabilities: ['profiles', 'addresses', 'preferences'],
    tech: 'Node.js',
    database: 'PostgreSQL'
  },
  'review': {
    domain: 'Reviews',
    capabilities: ['product reviews', 'ratings', 'moderation'],
    tech: 'Node.js',
    database: 'MongoDB'
  },
  'recommendation': {
    domain: 'Personalization',
    capabilities: ['product recommendations', 'personalized feed'],
    tech: 'Python + ML',
    database: 'Redis + ML models'
  },
  'analytics': {
    domain: 'Analytics',
    capabilities: ['user behavior', 'sales reports', 'forecasting'],
    tech: 'Python + Spark',
    database: 'ClickHouse'
  }
};

// ============ PRODUCT CATALOG SERVICE ============
// services/product-catalog/src/index.js
const express = require('express');
const elasticsearch = require('@elastic/elasticsearch');
const redis = require('redis');

class ProductCatalogService {
  constructor() {
    this.app = express();
    this.es = new elasticsearch.Client({ node: process.env.ELASTICSEARCH_URL });
    this.cache = redis.createClient({ url: process.env.REDIS_URL });
    
    this.setupRoutes();
  }
  
  setupRoutes() {
    // Get product by ID
    this.app.get('/api/products/:id', async (req, res) => {
      // Try cache first
      const cached = await this.cache.get(`product:${req.params.id}`);
      if (cached) {
        return res.json(JSON.parse(cached));
      }
      
      // Get from primary
      const product = await this.getProduct(req.params.id);
      
      // Cache for 5 minutes
      await this.cache.setex(
        `product:${req.params.id}`,
        300,
        JSON.stringify(product)
      );
      
      res.json(product);
    });
    
    // Search products
    this.app.get('/api/products/search', async (req, res) => {
      const { q, category, priceMin, priceMax, sort, page = 1 } = req.query;
      
      const results = await this.es.search({
        index: 'products',
        body: {
          query: {
            bool: {
              must: q ? { match: { name: q } } : { match_all: {} },
              filter: [
                category ? { term: { category } } : null,
                priceMin ? { range: { price: { gte: priceMin } } } : null,
                priceMax ? { range: { price: { lte: priceMax } } } : null
              ].filter(Boolean)
            }
          },
          sort: this.parseSort(sort),
          from: (page - 1) * 20,
          size: 20
        }
      });
      
      res.json({
        products: results.body.hits.hits.map(h => h._source),
        total: results.body.hits.total.value,
        page,
        totalPages: Math.ceil(results.body.hits.total.value / 20)
      });
    });
    
    // Admin: Create product
    this.app.post('/api/admin/products', async (req, res) => {
      const product = await this.createProduct(req.body);
      
      // Index in search
      await this.es.index({
        index: 'products',
        id: product.id,
        body: product
      });
      
      // Clear cache
      await this.cache.del(`product:${product.id}`);
      
      // Emit event
      await eventBus.publish('product.created', product);
      
      res.status(201).json(product);
    });
  }
  
  async getProduct(id) {
    // Get from database
    const product = await db.products.findById(id);
    
    // Get inventory from inventory service
    try {
      const inventory = await axios.get(
        `${SERVICE_URLS.inventory}/api/inventory/${id}`
      );
      product.inStock = inventory.data.available > 0;
      product.stockLevel = inventory.data.level;
    } catch (error) {
      // Inventory service down - use cached or default
      product.inStock = true;
      product.stockLevel = 'unknown';
    }
    
    // Get ratings from review service
    try {
      const ratings = await axios.get(
        `${SERVICE_URLS.review}/api/products/${id}/ratings`
      );
      product.rating = ratings.data.average;
      product.reviewCount = ratings.data.count;
    } catch (error) {
      product.rating = 0;
      product.reviewCount = 0;
    }
    
    return product;
  }
}

// ============ SHOPPING CART SERVICE ============
// services/cart-service/src/index.js
class CartService {
  constructor() {
    this.redis = redis.createClient({ url: process.env.REDIS_URL });
    this.setupRoutes();
  }
  
  setupRoutes() {
    // Get cart
    this.app.get('/api/cart/:userId', async (req, res) => {
      const cart = await this.getCart(req.params.userId);
      res.json(cart);
    });
    
    // Add to cart
    this.app.post('/api/cart/:userId/items', async (req, res) => {
      const { productId, quantity } = req.body;
      
      // Get product details from catalog service
      const product = await this.getProductDetails(productId);
      
      // Add to cart
      const cart = await this.addToCart(
        req.params.userId,
        product,
        quantity
      );
      
      // Emit event for analytics
      await eventBus.publish('cart.item.added', {
        userId: req.params.userId,
        productId,
        quantity,
        price: product.price
      });
      
      res.json(cart);
    });
    
    // Remove from cart
    this.app.delete('/api/cart/:userId/items/:productId', async (req, res) => {
      const cart = await this.removeFromCart(
        req.params.userId,
        req.params.productId
      );
      res.json(cart);
    });
  }
  
  async getCart(userId) {
    const cartKey = `cart:${userId}`;
    let cart = await this.redis.get(cartKey);
    
    if (!cart) {
      cart = { userId, items: [], total: 0 };
    } else {
      cart = JSON.parse(cart);
    }
    
    // Validate items with current prices
    for (const item of cart.items) {
      const current = await this.getProductDetails(item.productId);
      if (current.price !== item.price) {
        item.price = current.price;
        item.total = current.price * item.quantity;
      }
    }
    
    // Recalculate total
    cart.total = cart.items.reduce((sum, item) => sum + item.total, 0);
    
    return cart;
  }
  
  async addToCart(userId, product, quantity) {
    const cart = await this.getCart(userId);
    
    const existing = cart.items.find(i => i.productId === product.id);
    
    if (existing) {
      existing.quantity += quantity;
      existing.total = existing.price * existing.quantity;
    } else {
      cart.items.push({
        productId: product.id,
        name: product.name,
        price: product.price,
        quantity,
        total: product.price * quantity
      });
    }
    
    cart.total = cart.items.reduce((sum, i) => sum + i.total, 0);
    
    await this.redis.setex(`cart:${userId}`, 86400, JSON.stringify(cart));
    
    return cart;
  }
  
  async getProductDetails(productId) {
    // Call product catalog service
    const response = await axios.get(
      `${SERVICE_URLS.catalog}/api/products/${productId}`
    );
    return response.data;
  }
}

// ============ ORDER MANAGEMENT SERVICE ============
// services/order-service/src/index.js
class OrderService {
  constructor() {
    this.db = require('./database');
    this.setupRoutes();
    this.setupEventHandlers();
  }
  
  setupRoutes() {
    this.app.post('/api/orders', async (req, res) => {
      const { userId, items, shippingAddress, paymentMethod } = req.body;
      
      // Start saga
      const orderId = uuid();
      
      try {
        // 1. Create order (pending)
        const order = await this.createOrder({
          id: orderId,
          userId,
          items,
          shippingAddress,
          paymentMethod,
          status: 'PENDING'
        });
        
        // 2. Reserve inventory (call inventory service)
        await this.reserveInventory(orderId, items);
        
        // 3. Process payment (call payment service)
        const payment = await this.processPayment(orderId, paymentMethod, order.total);
        
        // 4. Update order status
        order.status = 'CONFIRMED';
        order.paymentId = payment.id;
        await this.updateOrder(order);
        
        // 5. Schedule shipping (call shipping service)
        await this.scheduleShipping(orderId, items, shippingAddress);
        
        // 6. Emit events
        await eventBus.publish('order.created', order);
        
        res.status(201).json({
          orderId,
          status: 'confirmed',
          estimatedDelivery: payment.estimatedDelivery
        });
        
      } catch (error) {
        // Compensating transactions
        await this.compensate(orderId, error);
        
        res.status(400).json({
          error: 'Order failed',
          reason: error.message
        });
      }
    });
  }
  
  async reserveInventory(orderId, items) {
    try {
      await axios.post(`${SERVICE_URLS.inventory}/api/reserve`, {
        orderId,
        items
      });
    } catch (error) {
      throw new Error('Inventory reservation failed');
    }
  }
  
  async processPayment(orderId, paymentMethod, amount) {
    try {
      const response = await axios.post(`${SERVICE_URLS.payment}/api/charge`, {
        orderId,
        method: paymentMethod,
        amount
      });
      return response.data;
    } catch (error) {
      throw new Error('Payment processing failed');
    }
  }
  
  async scheduleShipping(orderId, items, address) {
    try {
      await axios.post(`${SERVICE_URLS.shipping}/api/schedule`, {
        orderId,
        items,
        address
      });
    } catch (error) {
      // Non-critical - can retry later
      console.error('Shipping scheduling failed:', error);
      eventBus.publish('shipping.retry.required', { orderId, items, address });
    }
  }
  
  async compensate(orderId, error) {
    console.log(`Compensating order ${orderId} due to:`, error.message);
    
    // Release inventory if reserved
    await axios.post(`${SERVICE_URLS.inventory}/api/release`, { orderId }).catch(e => {
      console.error('Failed to release inventory:', e);
    });
    
    // Void payment if processed
    await axios.post(`${SERVICE_URLS.payment}/api/void`, { orderId }).catch(e => {
      console.error('Failed to void payment:', e);
    });
    
    // Update order status
    await this.updateOrderStatus(orderId, 'FAILED', error.message);
  }
}

// ============ PAYMENT SERVICE ============
// services/payment-service/src/index.js
class PaymentService {
  constructor() {
    this.gateways = {
      stripe: new StripeGateway(),
      paypal: new PayPalGateway(),
      braintree: new BraintreeGateway()
    };
  }
  
  async charge(orderId, method, amount) {
    const gateway = this.gateways[method.provider];
    
    if (!gateway) {
      throw new Error(`Unsupported payment provider: ${method.provider}`);
    }
    
    // Try primary gateway
    try {
      const result = await gateway.charge({
        amount,
        token: method.token,
        metadata: { orderId }
      });
      
      // Save transaction
      await this.saveTransaction({
        orderId,
        provider: method.provider,
        amount,
        status: 'SUCCESS',
        transactionId: result.id
      });
      
      return {
        id: result.id,
        status: 'success',
        provider: method.provider
      };
      
    } catch (error) {
      // Try fallback gateway
      console.log(`Primary gateway failed, trying fallback`);
      
      const fallbackGateway = this.getFallbackGateway(method.provider);
      
      try {
        const result = await fallbackGateway.charge({
          amount,
          token: method.token,
          metadata: { orderId }
        });
        
        return {
          id: result.id,
          status: 'success',
          provider: fallbackGateway.name
        };
      } catch (fallbackError) {
        throw new Error('All payment gateways failed');
      }
    }
  }
  
  async refund(transactionId) {
    const transaction = await this.getTransaction(transactionId);
    
    const gateway = this.gateways[transaction.provider];
    await gateway.refund(transactionId);
    
    await this.updateTransaction(transactionId, { status: 'REFUNDED' });
  }
}
```

### 3. Real Estate - MLS Platform

```javascript
// ============ SERVICE DECOMPOSITION ============
const realEstateServices = {
  'property-service': {
    responsibilities: ['property CRUD', 'listing management', 'MLS sync']
  },
  'search-service': {
    responsibilities: ['geo search', 'filtering', 'saved searches']
  },
  'user-service': {
    responsibilities: ['agents', 'buyers', 'sellers', 'preferences']
  },
  'showing-service': {
    responsibilities: ['schedule showings', 'calendar', 'confirmations']
  },
  'offer-service': {
    responsibilities: ['offers', 'counter-offers', 'negotiations']
  },
  'document-service': {
    responsibilities: ['contracts', 'disclosures', 'e-signatures']
  },
  'notification-service': {
    responsibilities: ['alerts', 'emails', 'SMS']
  },
  'analytics-service': {
    responsibilities: ['market trends', 'pricing', 'reports']
  }
};

// ============ PROPERTY SERVICE ============
// services/property-service/src/index.js
class PropertyService {
  constructor() {
    this.db = require('./database');
    this.mlsConnectors = {
      zillow: new ZillowConnector(),
      realtor: new RealtorConnector(),
      localMls: new LocalMLSConnector()
    };
  }
  
  async createListing(listingData) {
    const propertyId = uuid();
    
    // Save to own database
    const property = await this.db.properties.create({
      id: propertyId,
      ...listingData,
      status: 'ACTIVE',
      createdAt: new Date()
    });
    
    // Publish to MLS systems (async)
    for (const [mls, connector] of Object.entries(this.mlsConnectors)) {
      try {
        await connector.publish(property);
        await this.db.mlsSync.record({
          propertyId,
          mls,
          status: 'SYNCED'
        });
      } catch (error) {
        console.error(`Failed to sync with ${mls}:`, error);
        await this.db.mlsSync.record({
          propertyId,
          mls,
          status: 'FAILED',
          error: error.message
        });
      }
    }
    
    // Emit events
    await eventBus.publish('property.listed', {
      propertyId,
      address: property.address,
      price: property.price,
      features: property.features
    });
    
    return property;
  }
  
  async updatePrice(propertyId, newPrice, reason) {
    const oldPrice = await this.db.properties.getPrice(propertyId);
    
    await this.db.properties.update(propertyId, { price: newPrice });
    
    await this.db.priceHistory.create({
      propertyId,
      oldPrice,
      newPrice,
      reason,
      changedAt: new Date()
    });
    
    await eventBus.publish('property.price.changed', {
      propertyId,
      oldPrice,
      newPrice,
      reason
    });
  }
}

// ============ SEARCH SERVICE ============
// services/search-service/src/index.js
class SearchService {
  constructor() {
    this.es = new elasticsearch.Client({ node: process.env.ELASTICSEARCH_URL });
    this.setupEventHandlers();
  }
  
  setupEventHandlers() {
    eventBus.subscribe('property.listed', this.indexProperty.bind(this));
    eventBus.subscribe('property.price.changed', this.updatePrice.bind(this));
    eventBus.subscribe('property.sold', this.removeFromIndex.bind(this));
  }
  
  async search(criteria) {
    const {
      lat, lng, radius,
      minPrice, maxPrice,
      beds, baths,
      propertyType,
      features = [],
      page = 1,
      sort = 'relevance'
    } = criteria;
    
    const must = [];
    const filter = [];
    
    // Text search
    if (criteria.query) {
      must.push({
        multi_match: {
          query: criteria.query,
          fields: ['address^3', 'description', 'features']
        }
      });
    }
    
    // Geo filter
    if (lat && lng && radius) {
      filter.push({
        geo_distance: {
          distance: `${radius}km`,
          location: { lat, lon: lng }
        }
      });
    }
    
    // Price range
    if (minPrice || maxPrice) {
      const range = {};
      if (minPrice) range.gte = minPrice;
      if (maxPrice) range.lte = maxPrice;
      filter.push({ range: { price: range } });
    }
    
    // Property type
    if (propertyType) {
      filter.push({ term: { propertyType } });
    }
    
    // Features
    if (features.length > 0) {
      features.forEach(feature => {
        filter.push({ term: { features: feature } });
      });
    }
    
    const results = await this.es.search({
      index: 'properties',
      body: {
        query: {
          bool: {
            must: must.length > 0 ? must : { match_all: {} },
            filter
          }
        },
        sort: this.getSort(sort),
        from: (page - 1) * 20,
        size: 20,
        aggs: {
          price_ranges: {
            range: {
              field: 'price',
              ranges: [
                { to: 250000 },
                { from: 250000, to: 500000 },
                { from: 500000, to: 1000000 },
                { from: 1000000 }
              ]
            }
          },
          property_types: {
            terms: { field: 'propertyType' }
          },
          avg_price: {
            avg: { field: 'price' }
          }
        }
      }
    });
    
    return {
      properties: results.body.hits.hits.map(h => h._source),
      total: results.body.hits.total.value,
      page,
      totalPages: Math.ceil(results.body.hits.total.value / 20),
      aggregations: results.body.aggregations
    };
  }
  
  async indexProperty(event) {
    await this.es.index({
      index: 'properties',
      id: event.propertyId,
      body: {
        ...event,
        location: {
          lat: event.address.lat,
          lon: event.address.lng
        },
        indexedAt: new Date()
      }
    });
  }
}

// ============ SHOWING SERVICE ============
// services/showing-service/src/index.js
class ShowingService {
  constructor() {
    this.db = require('./database');
    this.calendar = new CalendarIntegration();
  }
  
  async scheduleShowing(request) {
    const { propertyId, buyerId, agentId, dateTime, duration = 60 } = request;
    
    // Check agent availability
    const agentAvailable = await this.checkAgentAvailability(agentId, dateTime, duration);
    if (!agentAvailable) {
      throw new Error('Agent not available at that time');
    }
    
    // Check property availability
    const propertyAvailable = await this.checkPropertyAvailability(propertyId, dateTime, duration);
    if (!propertyAvailable) {
      throw new Error('Property not available at that time');
    }
    
    // Create showing
    const showing = await this.db.showings.create({
      id: uuid(),
      propertyId,
      buyerId,
      agentId,
      dateTime,
      duration,
      status: 'SCHEDULED',
      createdAt: new Date()
    });
    
    // Block time in calendar
    await this.calendar.blockTime(agentId, dateTime, duration, `Showing at property ${propertyId}`);
    
    // Send notifications
    await eventBus.publish('showing.scheduled', {
      showingId: showing.id,
      propertyId,
      buyerId,
      agentId,
      dateTime
    });
    
    return showing;
  }
  
  async checkAgentAvailability(agentId, dateTime, duration) {
    const endTime = new Date(dateTime.getTime() + duration * 60000);
    
    const existing = await this.db.showings.findOne({
      agentId,
      status: 'SCHEDULED',
      dateTime: {
        $gte: dateTime,
        $lt: endTime
      }
    });
    
    return !existing;
  }
  
  async checkPropertyAvailability(propertyId, dateTime, duration) {
    const endTime = new Date(dateTime.getTime() + duration * 60000);
    
    const existing = await this.db.showings.findOne({
      propertyId,
      status: 'SCHEDULED',
      dateTime: {
        $gte: dateTime,
        $lt: endTime
      }
    });
    
    return !existing;
  }
}

// ============ API GATEWAY ============
// gateway/index.js
class APIGateway {
  constructor() {
    this.app = express();
    this.services = {
      properties: 'http://property-service:3001',
      search: 'http://search-service:3002',
      users: 'http://user-service:3003',
      showings: 'http://showing-service:3004',
      offers: 'http://offer-service:3005',
      documents: 'http://document-service:3006'
    };
    
    this.setupMiddleware();
    this.setupRoutes();
  }
  
  setupMiddleware() {
    // Authentication
    this.app.use(async (req, res, next) => {
      const token = req.headers.authorization?.split(' ')[1];
      
      if (!token && !req.path.startsWith('/public')) {
        return res.status(401).json({ error: 'Unauthorized' });
      }
      
      if (token) {
        try {
          const user = await this.verifyToken(token);
          req.user = user;
        } catch (error) {
          return res.status(401).json({ error: 'Invalid token' });
        }
      }
      
      next();
    });
    
    // Rate limiting
    this.app.use(rateLimit({
      windowMs: 15 * 60 * 1000,
      max: 100,
      keyGenerator: (req) => req.user?.id || req.ip
    }));
    
    // Logging
    this.app.use((req, res, next) => {
      console.log(`${new Date().toISOString()} - ${req.method} ${req.path}`);
      next();
    });
  }
  
  setupRoutes() {
    // Route to appropriate service
    this.app.use('/api/properties', createProxyMiddleware({
      target: this.services.properties,
      changeOrigin: true,
      pathRewrite: { '^/api/properties': '/api/properties' }
    }));
    
    this.app.use('/api/search', createProxyMiddleware({
      target: this.services.search,
      changeOrigin: true
    }));
    
    this.app.use('/api/users', createProxyMiddleware({
      target: this.services.users,
      changeOrigin: true
    }));
    
    this.app.use('/api/showings', createProxyMiddleware({
      target: this.services.showings,
      changeOrigin: true
    }));
    
    // Aggregate endpoint
    this.app.get('/api/property-details/:id', async (req, res) => {
      const { id } = req.params;
      
      try {
        // Parallel calls to multiple services
        const [property, showings, offers, documents] = await Promise.allSettled([
          axios.get(`${this.services.properties}/api/properties/${id}`),
          axios.get(`${this.services.showings}/api/showings/property/${id}`),
          axios.get(`${this.services.offers}/api/offers/property/${id}`),
          axios.get(`${this.services.documents}/api/documents/property/${id}`)
        ]);
        
        res.json({
          property: property.status === 'fulfilled' ? property.value.data : null,
          showings: showings.status === 'fulfilled' ? showings.value.data : [],
          offers: offers.status === 'fulfilled' ? offers.value.data : [],
          documents: documents.status === 'fulfilled' ? documents.value.data : [],
          _warnings: {
            property: property.status === 'rejected' ? 'Property service unavailable' : null,
            showings: showings.status === 'rejected' ? 'Showings service unavailable' : null,
            offers: offers.status === 'rejected' ? 'Offers service unavailable' : null,
            documents: documents.status === 'rejected' ? 'Documents service unavailable' : null
          }
        });
      } catch (error) {
        res.status(500).json({ error: 'Failed to fetch property details' });
      }
    });
  }
}
```

---

## Communication Patterns

### 1. Synchronous (HTTP/REST)
```javascript
// Order service calling inventory service
class OrderService {
  async checkInventory(productId, quantity) {
    try {
      const response = await axios.get(
        `http://inventory-service/api/stock/${productId}`,
        { timeout: 1000 }
      );
      return response.data.available >= quantity;
    } catch (error) {
      // Fallback logic
      return true; // Assume available if service down
    }
  }
}
```

### 2. Asynchronous (Message Queue)
```javascript
// Order service publishing event
await eventBus.publish('order.created', {
  orderId: order.id,
  items: order.items,
  userId: order.userId
});

// Inventory service consuming event
eventBus.subscribe('order.created', async (event) => {
  for (const item of event.items) {
    await inventory.reserve(item.productId, item.quantity);
  }
});
```

### 3. Event-Driven with Outbox
```javascript
class OrderService {
  async createOrder(orderData) {
    const transaction = await db.beginTransaction();
    
    try {
      // 1. Save order
      const order = await db.orders.create(orderData, { transaction });
      
      // 2. Save to outbox (same transaction)
      await db.outbox.create({
        eventType: 'order.created',
        payload: order,
        status: 'PENDING'
      }, { transaction });
      
      // 3. Commit
      await transaction.commit();
      
      return order;
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}
```

---

## Data Management Patterns

### 1. Database per Service
```javascript
// Order Service Database
const orderDB = new pg.Pool({
  database: 'order_service_db',
  user: 'order_user'
});

// Product Service Database
const productDB = new MongoClient('mongodb://product-service-db:27017/products');
```

### 2. Saga Pattern for Distributed Transactions
```javascript
class OrderSaga {
  async createOrder(orderData) {
    const sagaId = uuid();
    
    try {
      // Step 1: Create order
      const order = await this.orderService.create(orderData);
      
      // Step 2: Reserve inventory
      await this.inventoryService.reserve(order.id, order.items);
      
      // Step 3: Process payment
      await this.paymentService.charge(order.id, order.total);
      
      // Step 4: Confirm order
      await this.orderService.confirm(order.id);
      
      return { success: true, orderId: order.id };
    } catch (error) {
      // Compensate
      await this.compensate(sagaId);
      throw error;
    }
  }
  
  async compensate(sagaId) {
    await this.paymentService.refund(sagaId);
    await this.inventoryService.release(sagaId);
    await this.orderService.cancel(sagaId);
  }
}
```

### 3. CQRS for Query Optimization
```javascript
// Command side (Order Service)
app.post('/api/orders', async (req, res) => {
  const order = await orderCommand.createOrder(req.body);
  await eventBus.publish('order.created', order);
  res.json(order);
});

// Query side (Order Query Service)
eventBus.subscribe('order.created', async (event) => {
  await orderQueryDB.save({
    id: event.id,
    customerName: event.customer.name,
    total: event.total,
    status: event.status,
    // Denormalized for fast queries
    itemsCount: event.items.length,
    createdAt: event.createdAt
  });
});

app.get('/api/orders', async (req, res) => {
  const orders = await orderQueryDB.find(req.query);
  res.json(orders);
});
```

---

## Deployment Patterns

### 1. Service Discovery
```javascript
// Using Consul for service discovery
class ServiceRegistry {
  async register(service) {
    await consul.agent.service.register({
      name: service.name,
      address: service.address,
      port: service.port,
      check: {
        http: `http://${service.address}:${service.port}/health`,
        interval: '10s',
        timeout: '5s'
      }
    });
  }
  
  async discover(serviceName) {
    const services = await consul.catalog.service.nodes(serviceName);
    // Load balance
    return services[Math.floor(Math.random() * services.length)];
  }
}
```

### 2. Container Orchestration
```yaml
# docker-compose.yml
version: '3'
services:
  order-service:
    build: ./services/order
    ports:
      - "3001:3000"
    environment:
      - DATABASE_URL=postgres://order-db:5432/orders
      - REDIS_URL=redis://redis:6379
    depends_on:
      - order-db
      - redis
      
  order-db:
    image: postgres:13
    environment:
      - POSTGRES_DB=orders
      
  payment-service:
    build: ./services/payment
    ports:
      - "3002:3000"
      
  redis:
    image: redis:alpine
```

---

## Microservices vs Other Patterns

| Pattern | Scale | Team Size | Complexity | Use Case |
|---------|-------|-----------|------------|----------|
| **Monolith** | Small | 1-5 | Low | Startups, MVPs |
| **Modular Monolith** | Medium | 5-10 | Medium | Growing apps |
| **Microservices** | Large | 10+ per service | High | Large teams, complex domains |
| **Serverless** | Variable | Small | Medium | Event-driven, variable load |

---

## When to Use Microservices

### ✅ DO Use When:
- Large development teams (multiple teams)
- Complex domain with clear boundaries
- Need independent scaling
- Different technology requirements
- High availability requirements
- Frequent deployments

### ❌ DON'T Use When:
- Small team (under 10 developers)
- Simple CRUD application
- Startup/MVP (start monolith)
- No clear domain boundaries
- Low latency requirements (network overhead)
- Strong consistency requirements

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Distributed Monolith** | Ensure service independence |
| **Network Latency** | Cache, async communication |
| **Data Consistency** | Sagas, eventual consistency |
| **Service Discovery** | Use Consul, Kubernetes |
| **Monitoring** | Distributed tracing (Jaeger) |
| **Testing** | Contract testing (Pact) |
| **Versioning** | API versioning, graceful deprecation |

---

## Summary Checklist

| Aspect | Implementation |
|--------|---------------|
| **Service Boundaries** | Domain-driven design |
| **Communication** | HTTP + Messaging |
| **Data** | Database per service |
| **Discovery** | Consul, Kubernetes |
| **Gateway** | API Gateway, BFF |
| **Resilience** | Circuit breakers, retries |
| **Observability** | Logs, metrics, traces |
| **Deployment** | Containers, CI/CD |

**Bottom Line:** Microservices offer scalability and team autonomy but come with significant operational complexity. Start with a well-structured monolith and extract services only when boundaries are clear and benefits outweigh costs.