# Bulkhead Pattern

## Definition
The **Bulkhead pattern** is a resilience design pattern that isolates elements of an application into pools so that if one fails, the others continue to function. Named after the compartments (bulkheads) in a ship's hull that prevent water from flooding the entire vessel if one section is breached.

In software, bulkheads limit the impact of failures by partitioning resources (threads, connections, memory) so that failure in one part doesn't cascade to others.

---

## Core Concept
```
                    WITHOUT BULKHEAD
    ┌─────────────────────────────────────────┐
    │           APPLICATION                    │
    │  ┌─────────────────────────────────┐    │
    │  │      SHARED RESOURCE POOL       │    │
    │  │ [Threads, Connections, Memory]  │    │
    │  └─────────────────────────────────┘    │
    │           ▲              ▲               │
    │           │              │               │
    │  ┌────────┴──────┐ ┌─────┴────────┐     │
    │  │  Service A    │ │  Service B   │     │
    │  │   (Fails)     │ │   (Blocked)  │     │
    │  └───────────────┘ └──────────────┘     │
    │                                          │
    │  FAILURE CASCADES TO ALL SERVICES        │
    └──────────────────────────────────────────┘

                    WITH BULKHEAD
    ┌─────────────────────────────────────────┐
    │           APPLICATION                    │
    │  ┌─────────────────┐  ┌─────────────────┐│
    │  │   BULKHEAD 1    │  │   BULKHEAD 2    ││
    │  │  [10 Threads]   │  │  [10 Threads]   ││
    │  │  Service A      │  │  Service B      ││
    │  │   (Fails)       │  │   (Works)       ││
    │  └─────────────────┘  └─────────────────┘│
    │                                          │
    │  FAILURE CONTAINED, SERVICE B UNAFFECTED │
    └──────────────────────────────────────────┘
```

---

## Good vs Bad Example

### Bad Example (No Bulkhead)
```javascript
// SINGLE THREAD POOL FOR EVERYTHING
const axios = require('axios');

class BadAPIClient {
  constructor() {
    // Single connection pool for all services
    this.httpClient = axios.create({
      baseURL: 'https://api.example.com',
      maxSockets: 100 // Shared pool
    });
  }
  
  async callServiceA() {
    // If this service slows down, consumes all connections
    return this.httpClient.get('/service-a/slow-endpoint');
  }
  
  async callServiceB() {
    // Gets blocked when Service A consumes all connections
    return this.httpClient.get('/service-b/fast-endpoint');
  }
}

// PROBLEM: Service A failing/slow blocks Service B
```

### Good Example (With Bulkhead)
```javascript
// SEPARATE POOLS FOR EACH SERVICE
const axios = require('axios');
const http = require('http');
const https = require('https');

class GoodAPIClient {
  constructor() {
    // Separate connection pools for each service
    this.serviceAPool = new http.Agent({ 
      maxSockets: 10,  // Service A limited to 10 connections
      keepAlive: true,
      timeout: 5000
    });
    
    this.serviceBPool = new http.Agent({ 
      maxSockets: 20,  // Service B gets 20 connections
      keepAlive: true,
      timeout: 5000
    });
    
    this.serviceAPool = axios.create({
      baseURL: 'https://service-a.example.com',
      httpAgent: this.serviceAAgent,
      httpsAgent: this.serviceAAgent
    });
    
    this.serviceBClient = axios.create({
      baseURL: 'https://service-b.example.com',
      httpAgent: this.serviceBAgent,
      httpsAgent: this.serviceBAgent
    });
  }
  
  async callServiceA() {
    // Uses its own pool - failures here don't affect Service B
    return this.serviceAClient.get('/slow-endpoint');
  }
  
  async callServiceB() {
    // Separate pool, works even if Service A is down
    return this.serviceBClient.get('/fast-endpoint');
  }
}
```

---

## Types of Bulkheads

### 1. Thread-Based Bulkhead
Separate thread pools for different services or operations.

### 2. Connection-Based Bulkhead
Separate connection pools for different downstream services.

### 3. Semaphore-Based Bulkhead
Limit concurrent calls using semaphores (lighter than threads).

### 4. Queue-Based Bulkhead
Separate queues for different request types.

---

## Industry Use Cases

### 1. BFSI (Banking) - Payment Processing

```javascript
// ============ THREAD-BASED BULKHEAD ============
const { Bulkhead } = require('cockatiel');
const async = require('async');

class PaymentProcessor {
  constructor() {
    // Separate bulkheads for different payment types
    this.domesticBulkhead = new Bulkhead(20); // 20 concurrent domestic payments
    this.internationalBulkhead = new Bulkhead(5); // 5 concurrent international
    this.highValueBulkhead = new Bulkhead(2); // Only 2 high-value at a time
    
    // Separate queues
    this.domesticQueue = async.queue(this.processDomestic.bind(this), 20);
    this.internationalQueue = async.queue(this.processInternational.bind(this), 5);
    this.hvQueue = async.queue(this.processHighValue.bind(this), 2);
    
    // Monitoring
    this.metrics = {
      domestic: { active: 0, queued: 0 },
      international: { active: 0, queued: 0 },
      highValue: { active: 0, queued: 0 }
    };
  }
  
  async processPayment(payment) {
    const { type, amount } = payment;
    
    // Route to appropriate bulkhead based on payment characteristics
    if (amount > 1000000) {
      return this.processHighValuePayment(payment);
    } else if (payment.isInternational) {
      return this.processInternationalPayment(payment);
    } else {
      return this.processDomesticPayment(payment);
    }
  }
  
  async processDomesticPayment(payment) {
    return this.domesticBulkhead.execute(async () => {
      this.metrics.domestic.active++;
      
      try {
        console.log(`[Domestic] Processing payment ${payment.id}`);
        
        // Call domestic payment gateway
        const result = await this.callDomesticGateway(payment);
        
        // Update metrics
        this.metrics.domestic.success++;
        
        return result;
      } catch (error) {
        this.metrics.domestic.failures++;
        throw error;
      } finally {
        this.metrics.domestic.active--;
      }
    });
  }
  
  async processInternationalPayment(payment) {
    return this.internationalBulkhead.execute(async () => {
      this.metrics.international.active++;
      
      try {
        console.log(`[International] Processing payment ${payment.id}`);
        
        // International payments are slower, but isolated
        const result = await this.callInternationalGateway(payment);
        
        return result;
      } finally {
        this.metrics.international.active--;
      }
    });
  }
  
  async processHighValuePayment(payment) {
    return this.highValueBulkhead.execute(async () => {
      this.metrics.highValue.active++;
      
      try {
        console.log(`[HighValue] Processing payment ${payment.id}`);
        
        // High-value needs extra compliance checks
        await this.complianceCheck(payment);
        const result = await this.callHighValueGateway(payment);
        
        return result;
      } finally {
        this.metrics.highValue.active--;
      }
    });
  }
  
  // Health check endpoint
  getStatus() {
    return {
      domestic: {
        active: this.metrics.domestic.active,
        queued: this.domesticQueue.length(),
        utilization: this.metrics.domestic.active / 20 * 100
      },
      international: {
        active: this.metrics.international.active,
        queued: this.internationalQueue.length(),
        utilization: this.metrics.international.active / 5 * 100
      },
      highValue: {
        active: this.metrics.highValue.active,
        queued: this.hvQueue.length(),
        utilization: this.metrics.highValue.active / 2 * 100
      }
    };
  }
}

// ============ CONNECTION-BASED BULKHEAD ============
class DatabaseBulkhead {
  constructor() {
    // Separate connection pools for different operations
    this.pools = {
      reads: new pg.Pool({
        max: 50,        // 50 connections for reads
        idleTimeoutMillis: 30000
      }),
      writes: new pg.Pool({
        max: 20,        // 20 connections for writes
        idleTimeoutMillis: 30000
      }),
      reports: new pg.Pool({
        max: 5,         // Only 5 for heavy reports
        idleTimeoutMillis: 60000
      }),
      admin: new pg.Pool({
        max: 2,         // Minimal for admin queries
        idleTimeoutMillis: 30000
      })
    };
    
    // Monitor pool usage
    setInterval(() => this.logPoolStats(), 60000);
  }
  
  async query(type, sql, params) {
    const pool = this.pools[type];
    
    if (!pool) {
      throw new Error(`Unknown pool type: ${type}`);
    }
    
    const start = Date.now();
    
    try {
      const result = await pool.query(sql, params);
      
      // Track metrics
      this.recordMetrics(type, Date.now() - start, 'success');
      
      return result;
    } catch (error) {
      this.recordMetrics(type, Date.now() - start, 'failure');
      throw error;
    }
  }
  
  // Read operations
  async getAccount(accountId) {
    return this.query('reads', 
      'SELECT * FROM accounts WHERE id = $1', 
      [accountId]
    );
  }
  
  // Write operations  
  async updateBalance(accountId, amount) {
    return this.query('writes',
      'UPDATE accounts SET balance = $1 WHERE id = $2',
      [amount, accountId]
    );
  }
  
  // Heavy reporting
  async generateMonthlyReport(month) {
    return this.query('reports',
      `SELECT * FROM transactions 
       WHERE date_trunc('month', created_at) = $1`,
      [month]
    );
  }
  
  logPoolStats() {
    for (const [name, pool] of Object.entries(this.pools)) {
      console.log(`Pool ${name}:`, {
        total: pool.totalCount,
        idle: pool.idleCount,
        waiting: pool.waitingCount
      });
    }
  }
}

// ============ SEMAPHORE-BASED BULKHEAD ============
const { Semaphore } = require('async-mutex');

class APIBulkhead {
  constructor() {
    // Semaphores are lighter than thread pools
    this.semaphores = {
      critical: new Semaphore(5),     // Only 5 critical operations
      normal: new Semaphore(20),       // 20 normal operations
      background: new Semaphore(10)    // 10 background jobs
    };
    
    this.metrics = {
      critical: { acquired: 0, rejected: 0 },
      normal: { acquired: 0, rejected: 0 },
      background: { acquired: 0, rejected: 0 }
    };
  }
  
  async executeWithBulkhead(priority, operation) {
    const semaphore = this.semaphores[priority];
    
    if (!semaphore) {
      throw new Error(`Unknown priority: ${priority}`);
    }
    
    // Try to acquire semaphore
    const [value, release] = await semaphore.acquire();
    
    try {
      this.metrics[priority].acquired++;
      return await operation();
    } catch (error) {
      this.metrics[priority].failures = (this.metrics[priority].failures || 0) + 1;
      throw error;
    } finally {
      release();
    }
  }
  
  // Try without waiting (immediate rejection)
  async tryExecuteWithBulkhead(priority, operation) {
    const semaphore = this.semaphores[priority];
    
    if (semaphore.isLocked()) {
      this.metrics[priority].rejected++;
      throw new Error(`Bulkhead ${priority} is full, try later`);
    }
    
    return this.executeWithBulkhead(priority, operation);
  }
}

// Usage in banking
class BankingAPI {
  constructor() {
    this.bulkhead = new APIBulkhead();
  }
  
  async processCriticalTransaction(transaction) {
    // Critical operations get dedicated capacity
    return this.bulkhead.executeWithBulkhead('critical', async () => {
      // Fund transfer, payment processing
      await this.validateTransaction(transaction);
      await this.updateLedger(transaction);
      await this.notifyParties(transaction);
      
      return { status: 'completed', id: transaction.id };
    });
  }
  
  async getAccountHistory(accountId) {
    // Normal operations share capacity
    return this.bulkhead.executeWithBulkhead('normal', async () => {
      return await this.db.query(
        'SELECT * FROM transactions WHERE account_id = $1 LIMIT 100',
        [accountId]
      );
    });
  }
  
  async generateStatement(accountId) {
    // Background jobs have separate capacity
    return this.bulkhead.tryExecuteWithBulkhead('background', async () => {
      const statement = await this.db.query(
        `SELECT * FROM transactions 
         WHERE account_id = $1 
         AND created_at > NOW() - INTERVAL '30 days'`,
        [accountId]
      );
      
      await this.emailService.sendStatement(accountId, statement);
      
      return { queued: true };
    });
  }
}
```

### 2. E-commerce - Checkout System

```javascript
// ============ QUEUE-BASED BULKHEAD ============
class CheckoutBulkhead {
  constructor() {
    // Separate queues for different checkout paths
    this.queues = {
      standard: new Queue('standard', { 
        concurrency: 50,    // 50 concurrent standard checkouts
        timeout: 30000      // 30 second timeout
      }),
      premium: new Queue('premium', { 
        concurrency: 20,    // 20 concurrent premium
        timeout: 15000      // 15 second timeout (faster)
      }),
      guest: new Queue('guest', { 
        concurrency: 30,    // 30 guest checkouts
        timeout: 60000      // 60 second timeout (slower)
      })
    };
    
    // Monitor queue sizes
    setInterval(() => this.reportMetrics(), 5000);
  }
  
  async processCheckout(checkoutData) {
    const { userId, isPremium, isGuest } = checkoutData;
    
    // Route to appropriate queue
    let queue;
    if (isPremium) {
      queue = this.queues.premium;
    } else if (isGuest) {
      queue = this.queues.guest;
    } else {
      queue = this.queues.standard;
    }
    
    // Add to queue with timeout
    return queue.add(async () => {
      console.log(`Processing checkout for ${userId || 'guest'}`);
      
      // Checkout steps
      const inventory = await this.checkInventory(checkoutData.items);
      if (!inventory.available) {
        throw new Error('Out of stock');
      }
      
      const payment = await this.processPayment(checkoutData.payment);
      const order = await this.createOrder(checkoutData);
      
      await this.sendConfirmation(order);
      
      return order;
    });
  }
  
  reportMetrics() {
    for (const [name, queue] of Object.entries(this.queues)) {
      console.log(`Queue ${name}:`, {
        size: queue.size(),
        pending: queue.pending,
        completed: queue.completed,
        failed: queue.failed
      });
    }
  }
}

// ============ TENANT-BASED BULKHEAD ============
class MultiTenantBulkhead {
  constructor() {
    // Each tenant gets its own pool
    this.tenantPools = new Map();
    
    // Default limits
    this.defaultLimits = {
      connections: 10,
      requests: 100,
      memory: 256 // MB
    };
  }
  
  getTenantPool(tenantId) {
    if (!this.tenantPools.has(tenantId)) {
      this.tenantPools.set(tenantId, this.createTenantPool(tenantId));
    }
    
    return this.tenantPools.get(tenantId);
  }
  
  createTenantPool(tenantId) {
    const limits = this.getTenantLimits(tenantId);
    
    return {
      connections: new http.Agent({ 
        maxSockets: limits.connections,
        keepAlive: true
      }),
      requestCounter: 0,
      requestLimit: limits.requests,
      memoryUsed: 0,
      memoryLimit: limits.memory * 1024 * 1024,
      
      async execute(request) {
        // Check limits
        if (this.requestCounter >= this.requestLimit) {
          throw new Error(`Tenant ${tenantId} exceeded request limit`);
        }
        
        this.requestCounter++;
        
        try {
          return await request();
        } finally {
          // Decrement after completion
          setTimeout(() => this.requestCounter--, 1000);
        }
      }
    };
  }
  
  async handleRequest(tenantId, req, handler) {
    const pool = this.getTenantPool(tenantId);
    
    // Check memory (approximate)
    const memoryUsage = process.memoryUsage().heapUsed;
    if (memoryUsage > pool.memoryLimit) {
      throw new Error(`Tenant ${tenantId} exceeded memory limit`);
    }
    
    // Execute with tenant isolation
    return pool.execute(async () => {
      const start = Date.now();
      
      try {
        const result = await handler(req);
        
        // Track tenant metrics
        this.recordMetrics(tenantId, 'success', Date.now() - start);
        
        return result;
      } catch (error) {
        this.recordMetrics(tenantId, 'failure', Date.now() - start);
        throw error;
      }
    });
  }
}

// ============ FLASH SALE BULKHEAD ============
class FlashSaleBulkhead {
  constructor() {
    // Separate pools for flash sale vs regular traffic
    this.pools = {
      flashSale: {
        maxConcurrent: 100,
        queueSize: 1000,
        timeout: 5000,
        active: 0,
        queue: []
      },
      regular: {
        maxConcurrent: 500,
        queueSize: 5000,
        timeout: 10000,
        active: 0,
        queue: []
      }
    };
    
    // Process queues
    setInterval(() => this.processQueue('flashSale'), 10);
    setInterval(() => this.processQueue('regular'), 10);
  }
  
  async execute(type, operation) {
    const pool = this.pools[type];
    
    // Check queue size
    if (pool.queue.length >= pool.queueSize) {
      throw new Error(`Queue full for ${type}, try later`);
    }
    
    // If under limit, execute immediately
    if (pool.active < pool.maxConcurrent) {
      return this.runImmediately(pool, operation);
    }
    
    // Otherwise queue
    return new Promise((resolve, reject) => {
      pool.queue.push({
        operation,
        resolve,
        reject,
        timeout: setTimeout(() => {
          const index = pool.queue.findIndex(q => q.operation === operation);
          if (index !== -1) {
            pool.queue.splice(index, 1);
            reject(new Error(`Timeout in ${type} queue`));
          }
        }, pool.timeout)
      });
    });
  }
  
  async runImmediately(pool, operation) {
    pool.active++;
    
    try {
      return await operation();
    } finally {
      pool.active--;
    }
  }
  
  processQueue(type) {
    const pool = this.pools[type];
    
    while (pool.active < pool.maxConcurrent && pool.queue.length > 0) {
      const next = pool.queue.shift();
      clearTimeout(next.timeout);
      
      this.runImmediately(pool, next.operation)
        .then(next.resolve)
        .catch(next.reject);
    }
  }
}

// Usage in flash sale
class FlashSaleAPI {
  constructor() {
    this.bulkhead = new FlashSaleBulkhead();
  }
  
  async purchaseFlashSaleItem(userId, productId) {
    // Flash sale purchases get dedicated capacity
    return this.bulkhead.execute('flashSale', async () => {
      const product = await this.getProduct(productId);
      
      if (!product.inStock) {
        throw new Error('Sold out');
      }
      
      const order = await this.createOrder(userId, productId);
      
      return {
        orderId: order.id,
        message: 'Purchase successful!'
      };
    });
  }
  
  async browseCatalog() {
    // Regular browsing goes to normal pool
    return this.bulkhead.execute('regular', async () => {
      return await this.getCatalog();
    });
  }
}
```

### 3. Real Estate - MLS Integration

```javascript
// ============ SERVICE-BASED BULKHEAD ============
class MLSBulkhead {
  constructor() {
    // Different MLS providers have separate pools
    this.providers = {
      zillow: {
        client: new ZillowClient(),
        semaphore: new Semaphore(10), // Zillow rate limit
        failures: 0,
        circuitOpen: false
      },
      redfin: {
        client: new RedfinClient(),
        semaphore: new Semaphore(5),  // Redfin is stricter
        failures: 0,
        circuitOpen: false
      },
      localMLS: {
        client: new LocalMLSClient(),
        semaphore: new Semaphore(20), // Local has higher limits
        failures: 0,
        circuitOpen: false
      }
    };
  }
  
  async searchProperties(provider, criteria) {
    const providerPool = this.providers[provider];
    
    if (!providerPool) {
      throw new Error(`Unknown provider: ${provider}`);
    }
    
    // Circuit breaker
    if (providerPool.circuitOpen) {
      throw new Error(`${provider} circuit is open`);
    }
    
    // Try to acquire semaphore
    return providerPool.semaphore.runExclusive(async () => {
      try {
        const results = await providerPool.client.search(criteria);
        
        // Reset failures on success
        providerPool.failures = 0;
        
        return results;
      } catch (error) {
        providerPool.failures++;
        
        // Open circuit after 5 failures
        if (providerPool.failures >= 5) {
          providerPool.circuitOpen = true;
          
          // Reset after 30 seconds
          setTimeout(() => {
            providerPool.circuitOpen = false;
            providerPool.failures = 0;
          }, 30000);
        }
        
        throw error;
      }
    });
  }
  
  async aggregateSearch(criteria) {
    // Search all providers in parallel, but isolated
    const results = await Promise.allSettled([
      this.searchProperties('zillow', criteria),
      this.searchProperties('redfin', criteria),
      this.searchProperties('localMLS', criteria)
    ]);
    
    // Combine successful results
    const properties = results
      .filter(r => r.status === 'fulfilled')
      .flatMap(r => r.value);
    
    return {
      properties,
      failedProviders: results
        .filter(r => r.status === 'rejected')
        .map(r => r.reason.message)
    };
  }
}
```

---

## Implementation Patterns

### 1. Thread Pool Bulkhead

```javascript
const { Worker } = require('worker_threads');

class ThreadPoolBulkhead {
  constructor(name, size) {
    this.name = name;
    this.size = size;
    this.queue = [];
    this.active = 0;
    this.workers = [];
    
    // Create worker threads
    for (let i = 0; i < size; i++) {
      this.workers.push(new Worker('./worker.js'));
    }
  }
  
  async execute(task) {
    if (this.active >= this.size) {
      // Queue the task
      return new Promise((resolve, reject) => {
        this.queue.push({ task, resolve, reject });
      });
    }
    
    return this.runTask(task);
  }
  
  async runTask(task) {
    this.active++;
    
    try {
      // Find available worker
      const worker = this.workers.find(w => !w.busy);
      worker.busy = true;
      
      const result = await this.runInWorker(worker, task);
      
      worker.busy = false;
      this.active--;
      
      // Process next in queue
      this.processQueue();
      
      return result;
    } catch (error) {
      this.active--;
      this.processQueue();
      throw error;
    }
  }
  
  processQueue() {
    if (this.queue.length > 0 && this.active < this.size) {
      const next = this.queue.shift();
      this.runTask(next.task).then(next.resolve).catch(next.reject);
    }
  }
}
```

### 2. Semaphore Bulkhead with Timeout

```javascript
class SemaphoreBulkhead {
  constructor(limit, timeout = 30000) {
    this.limit = limit;
    this.timeout = timeout;
    this.active = 0;
    this.queue = [];
  }
  
  async acquire() {
    if (this.active < this.limit) {
      this.active++;
      return { release: this.release.bind(this) };
    }
    
    // Wait with timeout
    return new Promise((resolve, reject) => {
      const timeoutId = setTimeout(() => {
        const index = this.queue.findIndex(item => item.resolve === resolve);
        if (index !== -1) {
          this.queue.splice(index, 1);
          reject(new Error('Bulkhead timeout'));
        }
      }, this.timeout);
      
      this.queue.push({
        resolve: () => {
          clearTimeout(timeoutId);
          this.active++;
          resolve({ release: this.release.bind(this) });
        },
        reject
      });
    });
  }
  
  release() {
    this.active--;
    
    if (this.queue.length > 0) {
      const next = this.queue.shift();
      next.resolve();
    }
  }
  
  async execute(fn) {
    const { release } = await this.acquire();
    
    try {
      return await fn();
    } finally {
      release();
    }
  }
}
```

### 3. Dynamic Bulkhead

```javascript
class DynamicBulkhead {
  constructor(options = {}) {
    this.minSize = options.minSize || 5;
    this.maxSize = options.maxSize || 100;
    this.currentSize = this.minSize;
    this.targetUtilization = options.targetUtilization || 0.7;
    
    this.active = 0;
    this.queue = [];
    this.metrics = {
      totalRequests: 0,
      rejectedRequests: 0,
      queueLengths: []
    };
    
    // Adjust size periodically
    setInterval(() => this.adjustSize(), 60000);
  }
  
  async execute(fn) {
    this.metrics.totalRequests++;
    
    if (this.active >= this.currentSize) {
      // Reject or queue based on configuration
      if (this.queue.length > this.currentSize) {
        this.metrics.rejectedRequests++;
        throw new Error('Bulkhead full');
      }
      
      // Queue the request
      return new Promise((resolve, reject) => {
        this.queue.push({ fn, resolve, reject });
      });
    }
    
    return this.run(fn);
  }
  
  async run(fn) {
    this.active++;
    
    try {
      const result = await fn();
      return result;
    } finally {
      this.active--;
      this.processQueue();
    }
  }
  
  processQueue() {
    while (this.queue.length > 0 && this.active < this.currentSize) {
      const next = this.queue.shift();
      this.run(next.fn).then(next.resolve).catch(next.reject);
    }
  }
  
  adjustSize() {
    const utilization = this.active / this.currentSize;
    const queuePressure = this.queue.length / this.currentSize;
    
    // Record metrics
    this.metrics.queueLengths.push(this.queue.length);
    
    // Adjust based on utilization
    if (utilization > this.targetUtilization && this.currentSize < this.maxSize) {
      // Need more capacity
      this.currentSize = Math.min(
        this.currentSize + 5,
        this.maxSize
      );
      console.log(`Increasing bulkhead to ${this.currentSize}`);
    } else if (utilization < 0.3 && this.currentSize > this.minSize) {
      // Too much capacity
      this.currentSize = Math.max(
        this.currentSize - 5,
        this.minSize
      );
      console.log(`Decreasing bulkhead to ${this.currentSize}`);
    }
    
    // Log metrics
    console.log('Bulkhead metrics:', {
      size: this.currentSize,
      active: this.active,
      queueLength: this.queue.length,
      utilization: (utilization * 100).toFixed(1) + '%',
      rejected: this.metrics.rejectedRequests,
      avgQueue: this.metrics.queueLengths.reduce((a, b) => a + b, 0) / 
                 this.metrics.queueLengths.length
    });
    
    // Reset metrics
    this.metrics.queueLengths = [];
  }
}
```

### 4. Bulkhead with Fallback

```javascript
class BulkheadWithFallback {
  constructor(primaryBulkhead, fallbackBulkhead) {
    this.primary = primaryBulkhead;
    this.fallback = fallbackBulkhead;
  }
  
  async execute(primaryFn, fallbackFn) {
    try {
      // Try primary with its limits
      return await this.primary.execute(primaryFn);
    } catch (error) {
      // If primary fails or is overloaded, try fallback
      console.log('Primary bulkhead failed, trying fallback');
      
      try {
        return await this.fallback.execute(fallbackFn);
      } catch (fallbackError) {
        // Both failed
        throw new Error('Both primary and fallback failed');
      }
    }
  }
}

// Usage
const criticalPath = new BulkheadWithFallback(
  new SemaphoreBulkhead(10),  // Primary: 10 concurrent
  new SemaphoreBulkhead(5)     // Fallback: 5 concurrent
);

// In API
app.get('/api/critical-data', async (req, res) => {
  try {
    const data = await criticalPath.execute(
      // Primary: fast cache
      () => cache.get('critical-data'),
      // Fallback: slow database
      () => db.query('SELECT * FROM critical_data')
    );
    
    res.json(data);
  } catch (error) {
    res.status(503).json({ error: 'Service unavailable' });
  }
});
```

---

## Bulkhead vs Other Patterns

| Pattern | Purpose | Scope |
|---------|---------|-------|
| **Bulkhead** | Isolate failures | Resource pools |
| **Circuit Breaker** | Stop calling failing services | Request level |
| **Rate Limiter** | Control request rate | Time-based |
| **Retry** | Handle transient failures | Request level |
| **Timeout** | Bound wait time | Request level |

**Combined Example:**
```javascript
class ResilientService {
  constructor() {
    // Bulkhead: isolate resources
    this.bulkhead = new SemaphoreBulkhead(10);
    
    // Circuit breaker: stop calling failing service
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,
      timeout: 30000
    });
    
    // Rate limiter: control request rate
    this.rateLimiter = new RateLimiter({
      maxRequests: 100,
      perSecond: 60
    });
  }
  
  async callService(request) {
    // Rate limit first
    await this.rateLimiter.acquire();
    
    // Then circuit breaker
    return this.circuitBreaker.execute(async () => {
      // Then bulkhead for resource isolation
      return this.bulkhead.execute(async () => {
        return await axios.post('https://api.example.com', request);
      });
    });
  }
}
```

---

## When to Use Bulkhead

### ✅ DO Use When:
- Calling multiple downstream services
- Different priorities for requests
- Need to isolate tenant workloads
- Protecting shared resources (DB connections)
- Preventing cascade failures
- Guaranteeing resources for critical paths

### ❌ DON'T Use When:
- Simple applications with one dependency
- Resources are abundant and cheap
- Overhead outweighs benefits
- Already using circuit breakers (combine them)

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Too Many Bulkheads** | Complexity overhead, group similar services |
| **Wrong Sizing** | Monitor and adjust dynamically |
| **No Monitoring** | Track utilization, queue lengths |
| **Forgotten Timeouts** | Always set timeouts on queue |
| **Thread Starvation** | Monitor for deadlocks |
| **Resource Leak** | Always release in finally block |

---

## Summary Checklist

| Aspect | Implementation |
|--------|---------------|
| **Isolation Units** | By service, tenant, priority |
| **Resource Limits** | Threads, connections, memory |
| **Queuing** | Bounded queues with timeouts |
| **Monitoring** | Active count, queue length, rejections |
| **Dynamic Adjustment** | Scale based on load |
| **Fallback** | Alternative paths when full |
| **Testing** | Chaos engineering to validate |

**Bottom Line:** Bulkhead is essential for resilience. Isolate critical resources to prevent failures from cascading. Start with simple semaphore-based bulkheads and add complexity as needed. Always monitor and adjust limits based on real traffic.