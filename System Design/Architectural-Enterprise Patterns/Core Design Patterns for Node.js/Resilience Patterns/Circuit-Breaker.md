# Circuit Breaker Pattern

## Definition
The **Circuit Breaker** pattern prevents cascading failures in distributed systems by failing fast when a dependent service is unhealthy. It wraps calls to external services and monitors for failures. When failures reach a threshold, the circuit "trips" and all subsequent calls fail immediately without attempting the operation.

Named after electrical circuit breakers that protect electrical systems from damage by interrupting current flow when faults are detected.

---

## Core Concept
```
                    CLOSED STATE (Normal Operation)
    ┌─────────────────────────────────────────────────┐
    │  Request ──► Circuit Breaker ──► Service Call  │
    │                     │                           │
    │              Success/Failure Counter            │
    │                                                   │
    │  Threshold not reached → Keep calling             │
    └───────────────────────────────────────────────────┘

                    OPEN STATE (Failure Detected)
    ┌─────────────────────────────────────────────────┐
    │  Request ──► Circuit Breaker ──┐                │
    │                     │           │                │
    │              Threshold Exceeded │                │
    │                     │           ▼                │
    │              Return Error Immediately            │
    │              (No Service Call)                   │
    └───────────────────────────────────────────────────┘

                    HALF-OPEN STATE (Testing Recovery)
    ┌─────────────────────────────────────────────────┐
    │  Request ──► Circuit Breaker ──► Test Call      │
    │                     │                           │
    │              If Success → Close                 │
    │              If Failure → Open Again            │
    └───────────────────────────────────────────────────┘
```

---

## State Transitions

```
        ┌─────────────────────────────────┐
        │                                 │
        │         CLOSED                   │
        │    (Normal Operation)            │
        │                                 │
        │  Failure Threshold Exceeded      │
        └───────────────┬─────────────────┘
                        │
                        ▼
        ┌─────────────────────────────────┐
        │                                 │
        │          OPEN                     │
        │    (Failing Fast)                │
        │                                 │
        │  Timeout Expires                 │
        └───────────────┬─────────────────┘
                        │
                        ▼
        ┌─────────────────────────────────┐
        │                                 │
        │       HALF-OPEN                   │
        │    (Testing Recovery)            │
        │                                 │
        │  Success → Close                 │
        │  Failure → Open                   │
        └─────────────────────────────────┘
```

---

## Good vs Bad Example

### Bad Example (No Circuit Breaker)
```javascript
// BAD: No protection against failing services
class VulnerableService {
  async callExternalAPI() {
    // If this service is down, every call:
    // 1. Waits for timeout (30s)
    // 2. Consumes resources
    // 3. Backs up threads
    // 4. Cascades failure to callers
    return axios.get('https://api.example.com/data', {
      timeout: 30000
    });
  }
}

// When service is down:
// - 100 requests/second × 30s timeout = 3000 pending requests
// - Thread pool exhausted
// - Memory exhausted
// - Entire application crashes
```

### Good Example (With Circuit Breaker)
```javascript
// GOOD: Circuit breaker protects application
class ResilientService {
  constructor() {
    this.circuitBreaker = new CircuitBreaker({
      failureThreshold: 5,      // Open after 5 failures
      successThreshold: 3,      // Close after 3 successes in half-open
      timeout: 30000,           // Try again after 30s
      fallback: this.getFallbackData.bind(this)
    });
  }
  
  async callExternalAPI() {
    return this.circuitBreaker.execute(async () => {
      return axios.get('https://api.example.com/data');
    });
  }
  
  async getFallbackData() {
    // Return cached data when circuit is open
    return cache.get('api-data');
  }
}

// When service is down:
// - First 5 calls fail (open circuit)
// - Next calls fail immediately (no timeout)
// - Resources preserved
// - Fallback data returned
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Payment Gateway Integration

```javascript
// ============ ADVANCED CIRCUIT BREAKER FOR PAYMENTS ============
class PaymentGatewayCircuitBreaker {
  constructor() {
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.nextAttemptTime = null;
    
    // Configuration
    this.failureThreshold = 5;        // Open after 5 failures
    this.successThreshold = 3;        // Close after 3 successes
    this.timeout = 60000;              // 60 seconds in open state
    this.maxFailures = 100;            // Max failures before manual intervention
    
    // Metrics for each payment method
    this.methodMetrics = new Map();
    
    // Start health check
    setInterval(() => this.healthCheck(), 30000);
  }
  
  async processPayment(payment) {
    const method = payment.method; // 'credit_card', 'paypal', 'bank_transfer'
    
    // Get or create metrics for this payment method
    let metrics = this.methodMetrics.get(method);
    if (!metrics) {
      metrics = {
        state: 'CLOSED',
        failures: 0,
        lastFailure: null,
        totalCalls: 0,
        avgLatency: 0
      };
      this.methodMetrics.set(method, metrics);
    }
    
    // Check if this payment method circuit is open
    if (metrics.state === 'OPEN') {
      if (Date.now() - metrics.lastFailure < this.timeout) {
        console.log(`Circuit OPEN for ${method}, failing fast`);
        this.metrics.rejected++;
        throw new Error(`Payment method ${method} temporarily unavailable`);
      } else {
        // Timeout expired, move to half-open
        metrics.state = 'HALF_OPEN';
        metrics.failures = 0;
      }
    }
    
    const start = Date.now();
    
    try {
      // Process payment based on method
      let result;
      switch(method) {
        case 'credit_card':
          result = await this.processCreditCard(payment);
          break;
        case 'paypal':
          result = await this.processPayPal(payment);
          break;
        case 'bank_transfer':
          result = await this.processBankTransfer(payment);
          break;
      }
      
      // Update metrics
      metrics.totalCalls++;
      metrics.avgLatency = (metrics.avgLatency * (metrics.totalCalls - 1) + 
                           (Date.now() - start)) / metrics.totalCalls;
      
      // Handle half-open state
      if (metrics.state === 'HALF_OPEN') {
        metrics.failures = 0;
        metrics.state = 'CLOSED';
        console.log(`Circuit CLOSED for ${method} - recovered`);
      }
      
      return result;
      
    } catch (error) {
      metrics.failures++;
      metrics.lastFailure = Date.now();
      
      // Check if we need to open circuit
      if (metrics.state === 'CLOSED' && metrics.failures >= this.failureThreshold) {
        metrics.state = 'OPEN';
        console.error(`Circuit OPEN for ${method} - too many failures`);
        
        // Alert operations team
        this.alertTeam({
          method,
          failures: metrics.failures,
          error: error.message,
          time: new Date()
        });
      }
      
      // Half-open failure -> back to open
      if (metrics.state === 'HALF_OPEN') {
        metrics.state = 'OPEN';
        metrics.lastFailure = Date.now();
      }
      
      throw error;
    }
  }
  
  async processCreditCard(payment) {
    // Call credit card processor
    return axios.post('https://credit-card-processor.com/charge', {
      cardNumber: payment.cardNumber,
      amount: payment.amount,
      currency: payment.currency
    }, { timeout: 5000 });
  }
  
  async processPayPal(payment) {
    // Call PayPal API
    return axios.post('https://api.paypal.com/v1/payments/payment', {
      // PayPal specific fields
    }, { timeout: 7000 });
  }
  
  async processBankTransfer(payment) {
    // Call bank API (usually slower)
    return axios.post('https://bank-api.com/transfer', {
      // Bank transfer fields
    }, { timeout: 15000 });
  }
  
  healthCheck() {
    // Periodically check all payment methods
    for (const [method, metrics] of this.methodMetrics) {
      if (metrics.state === 'OPEN' && 
          Date.now() - metrics.lastFailure > this.timeout) {
        
        // Try a test transaction
        this.testMethod(method).then(success => {
          if (success) {
            metrics.state = 'CLOSED';
            metrics.failures = 0;
            console.log(`Health check: ${method} recovered`);
          }
        });
      }
    }
  }
  
  async testMethod(method) {
    try {
      // Attempt small test transaction
      await this.processPayment({
        method,
        amount: 1, // $1 test
        testMode: true
      });
      return true;
    } catch {
      return false;
    }
  }
  
  getStatus() {
    const status = {};
    
    for (const [method, metrics] of this.methodMetrics) {
      status[method] = {
        state: metrics.state,
        failures: metrics.failures,
        totalCalls: metrics.totalCalls,
        avgLatency: metrics.avgLatency.toFixed(0) + 'ms',
        lastFailure: metrics.lastFailure ? new Date(metrics.lastFailure).toISOString() : null
      };
    }
    
    return status;
  }
}

// ============ USAGE IN BANKING API ============
class BankingAPI {
  constructor() {
    this.paymentCircuit = new PaymentGatewayCircuitBreaker();
    this.accountCircuit = new CircuitBreaker({
      failureThreshold: 3,
      timeout: 10000,
      fallback: this.getCachedAccount.bind(this)
    });
  }
  
  async transferMoney(req) {
    const { fromAccount, toAccount, amount, method } = req;
    
    try {
      // Step 1: Check accounts (with circuit breaker)
      const accounts = await this.accountCircuit.execute(async () => {
        return await this.accountService.getAccounts([fromAccount, toAccount]);
      });
      
      // Step 2: Validate balance
      if (accounts[0].balance < amount) {
        throw new Error('Insufficient funds');
      }
      
      // Step 3: Process payment (with method-specific circuit)
      const payment = await this.paymentCircuit.processPayment({
        method,
        amount,
        fromAccount,
        toAccount
      });
      
      // Step 4: Update ledger
      await this.ledgerService.recordTransaction({
        fromAccount,
        toAccount,
        amount,
        paymentId: payment.id
      });
      
      return { success: true, transactionId: payment.id };
      
    } catch (error) {
      if (error.message.includes('circuit open')) {
        // Return degraded response
        return {
          success: false,
          message: 'Payment service temporarily unavailable',
          alternativeMethods: this.getAlternativeMethods(method)
        };
      }
      throw error;
    }
  }
  
  async getCachedAccount(accountId) {
    // Fallback when account service is down
    return cache.get(`account:${accountId}`);
  }
  
  getAlternativeMethods(failedMethod) {
    const alternatives = {
      'credit_card': ['paypal', 'bank_transfer'],
      'paypal': ['credit_card'],
      'bank_transfer': ['credit_card']
    };
    
    return alternatives[failedMethod] || [];
  }
}
```

### 2. E-commerce - Inventory and Shipping

```javascript
// ============ MULTI-SERVICE CIRCUIT BREAKER ============
class EcommerceCircuitBreaker {
  constructor() {
    // Separate circuit breakers for each dependency
    this.circuits = {
      inventory: new CircuitBreaker({
        name: 'inventory-service',
        failureThreshold: 5,
        successThreshold: 2,
        timeout: 30000,
        fallback: this.fallbackInventory.bind(this)
      }),
      
      shipping: new CircuitBreaker({
        name: 'shipping-service',
        failureThreshold: 3,
        successThreshold: 2,
        timeout: 15000,
        fallback: this.fallbackShipping.bind(this)
      }),
      
      payment: new CircuitBreaker({
        name: 'payment-service',
        failureThreshold: 5,
        successThreshold: 3,
        timeout: 45000,
        fallback: this.fallbackPayment.bind(this)
      }),
      
      recommendations: new CircuitBreaker({
        name: 'recommendation-service',
        failureThreshold: 10,
        successThreshold: 2,
        timeout: 5000,
        fallback: () => [] // Empty recommendations on failure
      })
    };
    
    // Metrics
    this.metrics = {
      totalRequests: 0,
      circuitOpen: 0,
      fallbackUsed: 0
    };
  }
  
  async checkout(order) {
    this.metrics.totalRequests++;
    
    try {
      // Execute all service calls with circuit protection
      const [inventory, shipping, payment] = await Promise.all([
        this.circuits.inventory.execute(() => 
          this.inventoryService.checkStock(order.items)
        ),
        
        this.circuits.shipping.execute(() => 
          this.shippingService.calculateRates(order.address, order.items)
        ),
        
        this.circuits.payment.execute(() => 
          this.paymentService.processPayment(order.payment)
        )
      ]);
      
      // Combine results
      return {
        success: true,
        inventory,
        shipping,
        payment,
        recommendations: await this.getRecommendations(order.userId)
      };
      
    } catch (error) {
      console.error('Checkout failed:', error);
      
      // Return partial success based on what worked
      return {
        success: false,
        partial: true,
        message: 'Some services unavailable',
        completed: {
          inventory: this.circuits.inventory.lastSuccess,
          shipping: this.circuits.shipping.lastSuccess,
          payment: this.circuits.payment.lastSuccess
        }
      };
    }
  }
  
  async getRecommendations(userId) {
    // Non-critical service - if it fails, just return empty
    try {
      return await this.circuits.recommendations.execute(async () => {
        const recs = await this.recommendationService.getForUser(userId);
        return recs;
      });
    } catch {
      return [];
    }
  }
  
  // Fallback methods
  async fallbackInventory(items) {
    console.log('Using inventory fallback');
    this.metrics.fallbackUsed++;
    
    // Return conservative estimates
    return items.map(item => ({
      productId: item.productId,
      available: true, // Assume available
      estimated: true  // Mark as estimated
    }));
  }
  
  async fallbackShipping(address, items) {
    console.log('Using shipping fallback');
    this.metrics.fallbackUsed++;
    
    // Return standard rate
    return {
      carrier: 'Standard',
      rate: 9.99,
      estimatedDays: '5-7',
      guaranteed: false
    };
  }
  
  async fallbackPayment(payment) {
    console.log('Using payment fallback');
    this.metrics.fallbackUsed++;
    
    // Cannot really fallback for payment
    throw new Error('Payment service unavailable');
  }
  
  getStatus() {
    const status = {};
    
    for (const [name, circuit] of Object.entries(this.circuits)) {
      status[name] = circuit.getState();
    }
    
    return {
      circuits: status,
      metrics: this.metrics,
      timestamp: new Date().toISOString()
    };
  }
}

// ============ FLASH SALE PROTECTION ============
class FlashSaleProtection {
  constructor() {
    this.circuit = new CircuitBreaker({
      failureThreshold: 100,      // Higher threshold for flash sales
      successThreshold: 10,
      timeout: 10000,             // Shorter timeout to try again quickly
      volumeThreshold: 1000,      // Only consider failures after 1000 calls
      monitoringWindow: 60000      // 1 minute window
    });
    
    this.stats = {
      totalAttempts: 0,
      successful: 0,
      failed: 0,
      rejected: 0
    };
  }
  
  async purchase(productId, userId) {
    this.stats.totalAttempts++;
    
    try {
      const result = await this.circuit.execute(async () => {
        // Actual purchase logic
        const product = await this.productService.get(productId);
        
        if (!product.inStock) {
          this.stats.failed++;
          throw new Error('Out of stock');
        }
        
        const order = await this.orderService.create({
          productId,
          userId,
          flashSale: true
        });
        
        this.stats.successful++;
        return order;
      });
      
      return result;
      
    } catch (error) {
      if (error.message === 'Circuit breaker is open') {
        this.stats.rejected++;
        
        // Queue for retry when circuit closes
        await this.retryQueue.add({
          productId,
          userId,
          attempt: Date.now()
        });
        
        return {
          status: 'queued',
          message: 'High traffic, your purchase is queued'
        };
      }
      
      throw error;
    }
  }
}
```

### 3. Real Estate - MLS Aggregator

```javascript
// ============ PRIORITY-BASED CIRCUIT BREAKER ============
class MLSCircuitBreaker {
  constructor() {
    this.circuits = {
      critical: new CircuitBreaker({
        name: 'critical-mls',
        failureThreshold: 3,      // Low threshold - fail fast
        timeout: 5000,             // Try again quickly
        halfOpenSuccess: 2
      }),
      
      normal: new CircuitBreaker({
        name: 'normal-mls',
        failureThreshold: 5,
        timeout: 30000,
        halfOpenSuccess: 3
      }),
      
      background: new CircuitBreaker({
        name: 'background-mls',
        failureThreshold: 10,
        timeout: 120000,           // Wait longer for background
        halfOpenSuccess: 5
      })
    };
    
    // Track provider health
    this.providerHealth = new Map();
  }
  
  async searchProperties(provider, criteria, priority = 'normal') {
    const circuit = this.circuits[priority];
    
    if (!circuit) {
      throw new Error(`Invalid priority: ${priority}`);
    }
    
    // Track provider health
    if (!this.providerHealth.has(provider)) {
      this.providerHealth.set(provider, {
        successes: 0,
        failures: 0,
        lastFailure: null,
        avgLatency: 0
      });
    }
    
    const start = Date.now();
    
    try {
      const results = await circuit.execute(async () => {
        return await this.callMLSProvider(provider, criteria);
      });
      
      // Update health metrics
      const health = this.providerHealth.get(provider);
      health.successes++;
      health.avgLatency = (health.avgLatency * (health.successes - 1) + 
                          (Date.now() - start)) / health.successes;
      
      return results;
      
    } catch (error) {
      const health = this.providerHealth.get(provider);
      health.failures++;
      health.lastFailure = new Date();
      
      // If provider is failing too much, deprioritize it
      if (health.failures > 10) {
        await this.deprioritizeProvider(provider);
      }
      
      // Try fallback provider
      return this.searchFallbackProvider(criteria, provider);
    }
  }
  
  async callMLSProvider(provider, criteria) {
    // Simulate API call
    const response = await axios.get(`${provider.baseUrl}/search`, {
      params: criteria,
      timeout: 10000
    });
    
    return response.data;
  }
  
  async searchFallbackProvider(criteria, excludeProvider) {
    // Get all providers except the failing one
    const providers = this.getHealthyProviders(excludeProvider);
    
    for (const provider of providers) {
      try {
        return await this.callMLSProvider(provider, criteria);
      } catch {
        continue; // Try next provider
      }
    }
    
    throw new Error('All MLS providers failed');
  }
  
  getHealthyProviders(exclude) {
    const healthy = [];
    
    for (const [provider, health] of this.providerHealth) {
      if (provider === exclude) continue;
      
      // Consider healthy if success rate > 80%
      const total = health.successes + health.failures;
      if (total === 0 || (health.successes / total) > 0.8) {
        healthy.push(provider);
      }
    }
    
    return healthy;
  }
  
  async deprioritizeProvider(provider) {
    console.log(`Deprioritizing failing provider: ${provider}`);
    
    // Store in configuration
    await redis.sadd('deprioritized-providers', provider);
    await redis.expire('deprioritized-providers', 3600); // 1 hour
    
    // Alert team
    await this.alertTeam({
      type: 'PROVIDER_DEGRADED',
      provider,
      time: new Date()
    });
  }
  
  getStatus() {
    const status = {};
    
    for (const [name, circuit] of Object.entries(this.circuits)) {
      status[name] = circuit.getMetrics();
    }
    
    status.providers = Array.from(this.providerHealth.entries()).map(
      ([name, health]) => ({
        name,
        ...health,
        lastFailure: health.lastFailure?.toISOString()
      })
    );
    
    return status;
  }
}
```

---

## Implementation Patterns

### 1. Basic Circuit Breaker

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 3;
    this.timeout = options.timeout || 60000;
    this.fallback = options.fallback;
    this.name = options.name || 'unnamed';
    
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = null;
    this.nextAttemptTime = null;
    
    // Metrics
    this.metrics = {
      totalCalls: 0,
      successfulCalls: 0,
      failedCalls: 0,
      rejectedCalls: 0,
      lastFailure: null
    };
  }
  
  async execute(fn) {
    this.metrics.totalCalls++;
    
    // Check if circuit is open
    if (this.state === 'OPEN') {
      if (Date.now() >= this.nextAttemptTime) {
        // Timeout expired, move to half-open
        this.transitionTo('HALF_OPEN');
      } else {
        this.metrics.rejectedCalls++;
        
        if (this.fallback) {
          return this.fallback();
        }
        
        throw new Error(`Circuit breaker ${this.name} is open`);
      }
    }
    
    try {
      const result = await fn();
      
      // Handle success
      this.onSuccess();
      
      return result;
    } catch (error) {
      this.onFailure(error);
      throw error;
    }
  }
  
  onSuccess() {
    this.metrics.successfulCalls++;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      
      if (this.successCount >= this.successThreshold) {
        this.transitionTo('CLOSED');
      }
    } else if (this.state === 'CLOSED') {
      // Reset failure count on success
      this.failureCount = 0;
    }
  }
  
  onFailure(error) {
    this.metrics.failedCalls++;
    this.metrics.lastFailure = new Date();
    
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.state === 'HALF_OPEN') {
      this.transitionTo('OPEN');
    } else if (this.state === 'CLOSED') {
      if (this.failureCount >= this.failureThreshold) {
        this.transitionTo('OPEN');
      }
    }
  }
  
  transitionTo(newState) {
    const oldState = this.state;
    this.state = newState;
    
    console.log(`Circuit ${this.name}: ${oldState} -> ${newState}`);
    
    if (newState === 'OPEN') {
      this.nextAttemptTime = Date.now() + this.timeout;
      this.successCount = 0;
      
      // Log for monitoring
      this.logStateChange(oldState, newState);
    } else if (newState === 'CLOSED') {
      this.failureCount = 0;
      this.successCount = 0;
    } else if (newState === 'HALF_OPEN') {
      this.successCount = 0;
    }
  }
  
  logStateChange(oldState, newState) {
    const log = {
      name: this.name,
      oldState,
      newState,
      failureCount: this.failureCount,
      successCount: this.successCount,
      timestamp: new Date().toISOString()
    };
    
    // Send to monitoring
    if (newState === 'OPEN') {
      console.error('Circuit opened:', log);
      // Alert team
    } else if (newState === 'CLOSED') {
      console.log('Circuit closed:', log);
    }
  }
  
  getState() {
    return {
      name: this.name,
      state: this.state,
      failureCount: this.failureCount,
      successCount: this.successCount,
      metrics: this.metrics,
      nextAttempt: this.nextAttemptTime ? new Date(this.nextAttemptTime).toISOString() : null
    };
  }
}
```

### 2. Advanced Circuit Breaker with Sliding Window

```javascript
class SlidingWindowCircuitBreaker {
  constructor(options = {}) {
    this.windowSize = options.windowSize || 60000; // 1 minute
    this.bucketSize = options.bucketSize || 10000; // 10 second buckets
    this.failureRateThreshold = options.failureRateThreshold || 0.5; // 50%
    this.minRequests = options.minRequests || 20;
    this.timeout = options.timeout || 60000;
    
    this.buckets = [];
    this.state = 'CLOSED';
    this.nextAttemptTime = null;
    
    // Create initial bucket
    this.addBucket();
    
    // Rotate buckets every bucketSize
    setInterval(() => this.rotateBuckets(), this.bucketSize);
  }
  
  addBucket() {
    this.buckets.push({
      startTime: Date.now(),
      successes: 0,
      failures: 0,
      timeouts: 0,
      rejects: 0
    });
  }
  
  rotateBuckets() {
    const now = Date.now();
    
    // Remove old buckets
    this.buckets = this.buckets.filter(b => 
      now - b.startTime < this.windowSize
    );
    
    // Add new bucket
    this.addBucket();
    
    // Check if we need to open circuit based on failure rate
    this.evaluateState();
  }
  
  getCurrentBucket() {
    return this.buckets[this.buckets.length - 1];
  }
  
  async execute(fn) {
    const bucket = this.getCurrentBucket();
    
    // Check if circuit is open
    if (this.state === 'OPEN') {
      if (Date.now() >= this.nextAttemptTime) {
        this.state = 'HALF_OPEN';
      } else {
        bucket.rejects++;
        throw new Error('Circuit breaker is open');
      }
    }
    
    const start = Date.now();
    
    try {
      const result = await Promise.race([
        fn(),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error('Timeout')), this.timeout)
        )
      ]);
      
      // Success
      bucket.successes++;
      
      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        console.log('Circuit closed after successful test');
      }
      
      return result;
      
    } catch (error) {
      if (error.message === 'Timeout') {
        bucket.timeouts++;
      } else {
        bucket.failures++;
      }
      
      // Check if we need to open circuit
      if (this.state === 'CLOSED' || this.state === 'HALF_OPEN') {
        const stats = this.getStats();
        
        if (stats.totalRequests >= this.minRequests && 
            stats.failureRate > this.failureRateThreshold) {
          this.state = 'OPEN';
          this.nextAttemptTime = Date.now() + this.timeout;
          console.log(`Circuit opened - failure rate: ${stats.failureRate}`);
        }
      }
      
      throw error;
    }
  }
  
  getStats() {
    const totalRequests = this.buckets.reduce(
      (sum, b) => sum + b.successes + b.failures + b.timeouts + b.rejects, 0
    );
    
    const totalFailures = this.buckets.reduce(
      (sum, b) => sum + b.failures + b.timeouts, 0
    );
    
    return {
      totalRequests,
      totalFailures,
      failureRate: totalRequests > 0 ? totalFailures / totalRequests : 0,
      buckets: this.buckets.length
    };
  }
}
```

### 3. Decorator-Based Circuit Breaker

```javascript
function circuitBreaker(options = {}) {
  const breaker = new CircuitBreaker(options);
  
  return function(target, propertyKey, descriptor) {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(...args) {
      return breaker.execute(() => originalMethod.apply(this, args));
    };
    
    return descriptor;
  };
}

class PaymentService {
  @circuitBreaker({
    name: 'payment-processor',
    failureThreshold: 5,
    timeout: 30000
  })
  async processPayment(amount, cardDetails) {
    // Payment processing logic
    return await axios.post('https://payment.com/charge', {
      amount,
      ...cardDetails
    });
  }
  
  @circuitBreaker({
    name: 'refund-service',
    failureThreshold: 3,
    timeout: 15000,
    fallback: async (transactionId) => {
      console.log(`Refund fallback for ${transactionId}`);
      return { status: 'queued', transactionId };
    }
  })
  async refundPayment(transactionId) {
    return await axios.post(`https://payment.com/refund/${transactionId}`);
  }
}
```

### 4. Redis-Backed Distributed Circuit Breaker

```javascript
class DistributedCircuitBreaker {
  constructor(name, redis, options = {}) {
    this.name = name;
    this.redis = redis;
    this.failureThreshold = options.failureThreshold || 5;
    this.timeout = options.timeout || 60000;
    this.localState = 'CLOSED';
    this.localFailureCount = 0;
  }
  
  async execute(fn) {
    // Check distributed state
    const state = await this.getState();
    
    if (state === 'OPEN') {
      const nextAttempt = await this.redis.get(`cb:${this.name}:nextAttempt`);
      
      if (Date.now() < parseInt(nextAttempt)) {
        throw new Error('Circuit breaker is open');
      }
    }
    
    try {
      const result = await fn();
      
      // Record success
      await this.recordSuccess();
      
      return result;
    } catch (error) {
      // Record failure
      await this.recordFailure();
      throw error;
    }
  }
  
  async getState() {
    const state = await this.redis.get(`cb:${this.name}:state`);
    return state || 'CLOSED';
  }
  
  async recordSuccess() {
    // Reset failure count
    await this.redis.del(`cb:${this.name}:failures`);
    
    // If in half-open, close circuit
    if (await this.getState() === 'HALF_OPEN') {
      await this.redis.set(`cb:${this.name}:state`, 'CLOSED');
    }
  }
  
  async recordFailure() {
    const failures = await this.redis.incr(`cb:${this.name}:failures`);
    
    if (failures >= this.failureThreshold) {
      // Open circuit
      await this.redis.set(`cb:${this.name}:state`, 'OPEN');
      await this.redis.set(
        `cb:${this.name}:nextAttempt`, 
        Date.now() + this.timeout
      );
      await this.redis.expire(`cb:${this.name}:nextAttempt`, Math.ceil(this.timeout / 1000));
      
      console.log(`Distributed circuit ${this.name} opened`);
    }
  }
}

// Usage across multiple instances
const redis = require('redis').createClient();
const paymentBreaker = new DistributedCircuitBreaker('payment-service', redis);
```

---

## Circuit Breaker vs Other Patterns

| Pattern | Purpose | State | When to Use |
|---------|---------|-------|-------------|
| **Circuit Breaker** | Prevent calls to failing service | Open/Closed/Half-Open | Service consistently failing |
| **Retry** | Handle transient failures | Attempt counter | Temporary network issues |
| **Bulkhead** | Isolate resources | Resource pools | Protect shared resources |
| **Rate Limiter** | Control request rate | Token bucket | Prevent overload |
| **Timeout** | Bound wait time | Timer | Slow responses |

**Combined Example:**
```javascript
class ResilientHttpClient {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.circuitBreaker = new CircuitBreaker({
      name: serviceName,
      failureThreshold: 5,
      timeout: 30000
    });
    
    this.bulkhead = new SemaphoreBulkhead(10);
    this.retry = new RetryStrategy(3);
    this.timeout = 5000;
  }
  
  async request(options) {
    // Bulkhead: limit concurrent requests
    return this.bulkhead.execute(async () => {
      // Circuit breaker: prevent calls to failing service
      return this.circuitBreaker.execute(async () => {
        // Retry: handle transient failures
        return this.retry.execute(async () => {
          // Timeout: don't wait forever
          return axios({
            ...options,
            timeout: this.timeout
          });
        });
      });
    });
  }
}
```

---

## When to Use Circuit Breaker

### ✅ DO Use When:
- Calling external services/APIs
- Database connections
- Microservice communication
- Third-party integrations
- Any operation that can fail consistently

### ❌ DON'T Use When:
- Local in-memory operations
- Operations that must always try
- When fallback is worse than failure
- Very low latency requirements
- Idempotent operations (retry may be better)

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Too Sensitive** | Set appropriate thresholds based on normal failure rates |
| **Not Monitoring** | Log all state changes and metrics |
| **No Fallback** | Always provide fallback for critical operations |
| **Wrong Timeout** | Base timeout on service SLAs |
| **Half-Open Testing** | Test with real requests, not just pings |
| **State Consistency** | Use distributed circuit breakers for multi-instance |

---

## Summary Checklist

| Aspect | Implementation |
|--------|---------------|
| **Thresholds** | Based on normal failure rates |
| **Timeouts** | Aligned with service SLAs |
| **Fallbacks** | Cached data, defaults, degraded mode |
| **Monitoring** | Track state changes, failure rates |
| **Half-Open** | Test with real traffic |
| **Metrics** | Success/failure counts, latency |
| **Alerting** | Notify on state changes |

**Bottom Line:** Circuit breaker is essential for building resilient distributed systems. It prevents cascading failures and allows services to degrade gracefully. Combine with retries, timeouts, and bulkheads for comprehensive resilience.