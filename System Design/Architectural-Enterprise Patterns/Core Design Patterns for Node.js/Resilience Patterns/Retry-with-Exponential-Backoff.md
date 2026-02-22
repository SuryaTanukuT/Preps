# Retry with Exponential Backoff Pattern

## Definition
**Retry with Exponential Backoff** is a resilience pattern that automatically retries failed operations with increasing delays between attempts. The delay grows exponentially after each failure, preventing overwhelmed systems from receiving repeated requests and giving them time to recover.

This pattern handles **transient failures** (temporary network glitches, service restarts, rate limiting) while avoiding **retry storms** that can make problems worse.

---

## Core Concept
```
                    RETRY FLOW
    ┌─────────────────────────────────────────────┐
    │  Request ──► Operation Fails                │
    │                    │                         │
    │         ┌──────────▼──────────┐             │
    │         │ Wait: 100ms (2^1×50)│             │
    │         └──────────┬──────────┘             │
    │                    │                         │
    │         ┌──────────▼──────────┐             │
    │         │ Retry 1 - Fails      │             │
    │         └──────────┬──────────┘             │
    │                    │                         │
    │         ┌──────────▼──────────┐             │
    │         │ Wait: 200ms (2^2×50)│             │
    │         └──────────┬──────────┘             │
    │                    │                         │
    │         ┌──────────▼──────────┐             │
    │         │ Retry 2 - Succeeds   │             │
    │         └──────────┬──────────┘             │
    │                    │                         │
    │              Return Success                   │
    └───────────────────────────────────────────────┘

                    BACKOFF COMPARISON
    Attempt 1: 0ms
    Attempt 2: 100ms
    Attempt 3: 200ms
    Attempt 4: 400ms
    Attempt 5: 800ms
    Attempt 6: 1600ms
```

---

## Good vs Bad Example

### Bad Example (Fixed Retry)
```javascript
// BAD: Fixed retry with no backoff
async function badRetry(fn) {
  for (let i = 0; i < 5; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === 4) throw error;
      // Always wait 1 second
      await sleep(1000);
    }
  }
}

// When service is struggling:
// - 5 requests per second regardless of service health
// - Can make overload worse
// - No jitter means synchronized retries
```

### Good Example (Exponential Backoff)
```javascript
// GOOD: Exponential backoff with jitter
async function goodRetry(fn, options = {}) {
  const maxRetries = options.maxRetries || 5;
  const baseDelay = options.baseDelay || 100;
  const maxDelay = options.maxDelay || 30000;
  
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      
      // Calculate delay: baseDelay * 2^attempt
      const delay = Math.min(
        baseDelay * Math.pow(2, i),
        maxDelay
      );
      
      // Add jitter: randomize between 80-120% of delay
      const jitter = delay * (0.8 + Math.random() * 0.4);
      
      console.log(`Retry ${i + 1} after ${Math.round(jitter)}ms`);
      await sleep(jitter);
    }
  }
}
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Payment Processing

```javascript
// ============ PAYMENT RETRY WITH EXPONENTIAL BACKOFF ============
class PaymentRetryService {
  constructor() {
    this.retryStrategies = {
      transient: {
        maxRetries: 5,
        baseDelay: 1000,
        maxDelay: 60000,
        retryableErrors: ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED', '429', '503']
      },
      critical: {
        maxRetries: 10,
        baseDelay: 2000,
        maxDelay: 300000, // 5 minutes
        retryableErrors: ['ECONNRESET', 'ETIMEDOUT', '500', '502', '503', '504']
      },
      nonCritical: {
        maxRetries: 3,
        baseDelay: 500,
        maxDelay: 10000,
        retryableErrors: ['ECONNRESET', 'ETIMEDOUT']
      }
    };
    
    this.metrics = {
      totalRetries: 0,
      successfulRetries: 0,
      failedRetries: 0,
      byErrorType: {}
    };
  }
  
  async processPayment(payment, priority = 'critical') {
    const strategy = this.retryStrategies[priority];
    let attempt = 0;
    let lastError = null;
    
    while (attempt < strategy.maxRetries) {
      try {
        const result = await this.callPaymentGateway(payment);
        
        // Log success
        if (attempt > 0) {
          this.metrics.successfulRetries++;
          console.log(`Payment succeeded after ${attempt} retries`);
        }
        
        return {
          ...result,
          retryInfo: {
            attempts: attempt,
            strategy: priority
          }
        };
        
      } catch (error) {
        lastError = error;
        attempt++;
        
        // Track metrics
        this.metrics.totalRetries++;
        const errorType = error.code || error.status || 'UNKNOWN';
        this.metrics.byErrorType[errorType] = (this.metrics.byErrorType[errorType] || 0) + 1;
        
        // Check if retryable
        if (!this.isRetryable(error, strategy)) {
          console.log(`Non-retryable error: ${error.message}`);
          break;
        }
        
        if (attempt >= strategy.maxRetries) {
          console.log(`Max retries (${strategy.maxRetries}) reached`);
          break;
        }
        
        // Calculate backoff
        const delay = this.calculateBackoff(attempt, strategy);
        
        console.log(`Payment attempt ${attempt} failed, retrying in ${delay}ms`, {
          error: error.message,
          attempt,
          nextDelay: delay
        });
        
        // Wait with backoff
        await this.sleep(delay);
      }
    }
    
    // All retries failed
    this.metrics.failedRetries++;
    
    // Queue for offline processing
    const queuedId = await this.queueForOffline(payment, lastError);
    
    return {
      status: 'queued',
      queueId: queuedId,
      message: 'Payment queued for offline processing',
      error: lastError.message,
      attempts: attempt
    };
  }
  
  calculateBackoff(attempt, strategy) {
    // Exponential backoff: baseDelay * 2^(attempt-1)
    const exponentialDelay = strategy.baseDelay * Math.pow(2, attempt - 1);
    
    // Cap at max delay
    const cappedDelay = Math.min(exponentialDelay, strategy.maxDelay);
    
    // Add jitter to prevent thundering herd
    const jitter = this.calculateJitter(cappedDelay);
    
    return Math.floor(jitter);
  }
  
  calculateJitter(delay) {
    // Full jitter: random between 0 and delay
    if (process.env.RETRY_JITTER === 'full') {
      return Math.random() * delay;
    }
    
    // Equal jitter: random between delay/2 and delay
    if (process.env.RETRY_JITTER === 'equal') {
      return delay/2 + Math.random() * (delay/2);
    }
    
    // Decorrelated jitter (more aggressive)
    if (process.env.RETRY_JITTER === 'decorrelated') {
      return delay * (0.5 + Math.random());
    }
    
    // No jitter
    return delay;
  }
  
  isRetryable(error, strategy) {
    // Check if error is in retryable list
    const errorCode = error.code || error.status?.toString() || '';
    
    return strategy.retryableErrors.some(retryable => 
      errorCode.includes(retryable) || 
      error.message.includes(retryable)
    );
  }
  
  async callPaymentGateway(payment) {
    // Simulate payment gateway call
    const response = await axios.post('https://payment-gateway.com/charge', payment, {
      timeout: 5000
    });
    
    return response.data;
  }
  
  async queueForOffline(payment, error) {
    // Store in database for batch processing
    const queuedPayment = await db.query(`
      INSERT INTO payment_queue 
      (payment_data, error, attempts, created_at)
      VALUES ($1, $2, $3, $4)
      RETURNING id
    `, [payment, error.message, error.attempts, new Date()]);
    
    return queuedPayment.id;
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
  
  getMetrics() {
    return {
      ...this.metrics,
      successRate: this.metrics.totalRetries > 0 
        ? (this.metrics.successfulRetries / this.metrics.totalRetries * 100).toFixed(2) + '%'
        : 'N/A',
      byErrorType: this.metrics.byErrorType
    };
  }
}

// ============ BANKING API WITH INTELLIGENT RETRY ============
class BankingAPI {
  constructor() {
    this.paymentRetry = new PaymentRetryService();
    this.accountRetry = new AccountRetryService();
  }
  
  async transferMoney(transfer) {
    const startTime = Date.now();
    
    try {
      const result = await this.paymentRetry.processPayment({
        fromAccount: transfer.from,
        toAccount: transfer.to,
        amount: transfer.amount,
        currency: transfer.currency
      }, 'critical');
      
      return {
        success: true,
        transactionId: result.id,
        processingTime: Date.now() - startTime,
        retryInfo: result.retryInfo
      };
    } catch (error) {
      // Log for monitoring
      logger.error('Transfer failed after retries', {
        transfer,
        error: error.message,
        duration: Date.now() - startTime
      });
      
      throw new Error('Transfer failed, please try again later');
    }
  }
  
  async getBalance(accountId) {
    // Different retry strategy for reads
    const retryStrategy = {
      maxRetries: 3,
      baseDelay: 100,
      maxDelay: 1000,
      retryableErrors: ['ECONNRESET', 'ETIMEDOUT', '503']
    };
    
    return this.retryWithBackoff(async () => {
      const balance = await this.accountService.getBalance(accountId);
      return balance;
    }, retryStrategy);
  }
  
  async retryWithBackoff(fn, strategy) {
    let lastError;
    
    for (let attempt = 1; attempt <= strategy.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;
        
        if (attempt === strategy.maxRetries) break;
        
        if (!this.isRetryableError(error, strategy)) {
          break;
        }
        
        const delay = strategy.baseDelay * Math.pow(2, attempt - 1);
        const jitteredDelay = delay * (0.9 + Math.random() * 0.2);
        
        await this.sleep(jitteredDelay);
      }
    }
    
    throw lastError;
  }
  
  isRetryableError(error, strategy) {
    const status = error.response?.status;
    const code = error.code;
    
    // Don't retry client errors (4xx except 429)
    if (status >= 400 && status < 500 && status !== 429) {
      return false;
    }
    
    return strategy.retryableErrors.includes(status?.toString()) ||
           strategy.retryableErrors.includes(code);
  }
}
```

### 2. E-commerce - Order Processing

```javascript
// ============ ORDER SERVICE WITH ADAPTIVE RETRY ============
class OrderService {
  constructor() {
    this.retryQueue = [];
    this.processing = false;
    this.stats = {
      total: 0,
      succeeded: 0,
      failed: 0,
      retries: 0
    };
  }
  
  async createOrder(orderData) {
    this.stats.total++;
    
    const retryConfig = {
      maxRetries: 5,
      baseDelay: 200,
      maxDelay: 10000,
      backoffFactor: 2,
      retryCondition: (error) => {
        // Retry on network errors and 5xx
        return error.code === 'ECONNRESET' ||
               error.code === 'ETIMEDOUT' ||
               error.response?.status >= 500;
      },
      onRetry: (attempt, delay, error) => {
        console.log(`Retrying order ${orderData.id}, attempt ${attempt}, delay ${delay}ms`);
        this.stats.retries++;
      }
    };
    
    return this.executeWithRetry(
      () => this.submitOrder(orderData),
      retryConfig
    );
  }
  
  async executeWithRetry(fn, config) {
    let attempt = 0;
    let lastError;
    
    while (attempt < config.maxRetries) {
      try {
        const result = await fn();
        
        if (attempt > 0) {
          this.stats.succeeded++;
        }
        
        return result;
      } catch (error) {
        lastError = error;
        attempt++;
        
        // Check if we should retry
        if (!config.retryCondition(error) || attempt >= config.maxRetries) {
          break;
        }
        
        // Calculate backoff with exponential factor
        const delay = Math.min(
          config.baseDelay * Math.pow(config.backoffFactor, attempt - 1),
          config.maxDelay
        );
        
        // Add jitter
        const jitteredDelay = delay * (0.8 + Math.random() * 0.4);
        
        // Callback
        if (config.onRetry) {
          config.onRetry(attempt, jitteredDelay, error);
        }
        
        // Wait
        await this.sleep(jitteredDelay);
      }
    }
    
    this.stats.failed++;
    
    // Add to dead letter queue for manual processing
    await this.addToDeadLetterQueue(fn, lastError, attempt);
    
    throw lastError;
  }
  
  async submitOrder(orderData) {
    // Try primary inventory service
    try {
      const inventory = await this.inventoryService.reserve(orderData.items);
      if (!inventory.available) {
        throw new Error('Out of stock');
      }
    } catch (error) {
      // Fallback to secondary inventory service
      console.log('Primary inventory failed, trying secondary');
      const backupInventory = await this.backupInventoryService.reserve(orderData.items);
      if (!backupInventory.available) {
        throw new Error('Out of stock');
      }
    }
    
    // Create order
    const order = await this.orderDB.create(orderData);
    
    // Try payment (with retry)
    const payment = await this.processPaymentWithRetry(order);
    
    // Send confirmation (don't fail order if email fails)
    this.sendConfirmationWithRetry(order).catch(e => {
      console.log('Confirmation email failed:', e.message);
    });
    
    return order;
  }
  
  async processPaymentWithRetry(order) {
    const paymentConfig = {
      maxRetries: 3,
      baseDelay: 1000,
      maxDelay: 5000,
      retryCondition: (error) => {
        // Only retry on gateway timeouts
        return error.code === 'ETIMEDOUT' || 
               error.response?.status === 504;
      }
    };
    
    return this.executeWithRetry(async () => {
      return await this.paymentService.charge(order);
    }, paymentConfig);
  }
  
  async sendConfirmationWithRetry(order) {
    const emailConfig = {
      maxRetries: 5,
      baseDelay: 5000, // Longer delays for email
      maxDelay: 3600000, // 1 hour
      backoffFactor: 3,
      retryCondition: () => true // Always retry emails
    };
    
    return this.executeWithRetry(async () => {
      await this.emailService.sendOrderConfirmation(order);
    }, emailConfig);
  }
  
  async addToDeadLetterQueue(operation, error, attempts) {
    await db.query(`
      INSERT INTO dead_letter_queue
      (operation_type, payload, error, attempts, created_at)
      VALUES ($1, $2, $3, $4, $5)
    `, ['order.create', JSON.stringify(operation), error.message, attempts, new Date()]);
  }
  
  async processDeadLetterQueue() {
    const items = await db.query(`
      SELECT * FROM dead_letter_queue
      WHERE processed = false
      ORDER BY created_at
      LIMIT 100
    `);
    
    for (const item of items.rows) {
      try {
        // Attempt to reprocess
        const order = await this.createOrder(JSON.parse(item.payload));
        
        // Mark as processed
        await db.query(`
          UPDATE dead_letter_queue
          SET processed = true, 
              processed_at = NOW(),
              result = $1
          WHERE id = $2
        `, [JSON.stringify(order), item.id]);
        
      } catch (error) {
        // Increment attempt count
        await db.query(`
          UPDATE dead_letter_queue
          SET attempts = attempts + 1,
              last_error = $1
          WHERE id = $2
        `, [error.message, item.id]);
      }
    }
  }
}

// ============ INVENTORY CHECK WITH RETRY ============
class InventoryService {
  async checkStock(productId, quantity) {
    const retryConfig = {
      maxRetries: 3,
      baseDelay: 100,
      maxDelay: 1000,
      retryableErrors: ['ECONNRESET', 'ETIMEDOUT'],
      
      // Circuit breaker integration
      circuitBreaker: {
        failures: 0,
        threshold: 5,
        open: false,
        openUntil: null
      }
    };
    
    return this.retryWithCircuitBreaker(async () => {
      const response = await axios.get(
        `http://inventory-service/stock/${productId}`,
        { timeout: 500 }
      );
      
      return response.data.available >= quantity;
    }, retryConfig);
  }
  
  async retryWithCircuitBreaker(fn, config) {
    // Check circuit breaker
    if (config.circuitBreaker.open) {
      if (Date.now() > config.circuitBreaker.openUntil) {
        config.circuitBreaker.open = false;
        config.circuitBreaker.failures = 0;
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    let attempt = 0;
    
    while (attempt < config.maxRetries) {
      try {
        const result = await fn();
        
        // Reset circuit breaker on success
        config.circuitBreaker.failures = 0;
        
        return result;
      } catch (error) {
        attempt++;
        
        // Update circuit breaker
        if (config.retryableErrors.includes(error.code)) {
          config.circuitBreaker.failures++;
          
          if (config.circuitBreaker.failures >= config.circuitBreaker.threshold) {
            config.circuitBreaker.open = true;
            config.circuitBreaker.openUntil = Date.now() + 30000; // 30 seconds
          }
        }
        
        if (attempt >= config.maxRetries) {
          throw error;
        }
        
        const delay = config.baseDelay * Math.pow(2, attempt - 1);
        await this.sleep(delay);
      }
    }
  }
}
```

### 3. Real Estate - MLS Data Sync

```javascript
// ============ MLS SYNC WITH ADAPTIVE BACKOFF ============
class MLSSyncService {
  constructor() {
    this.providerHealth = new Map();
    this.syncQueue = [];
  }
  
  async syncProperty(propertyId, provider) {
    const providerHealth = this.getProviderHealth(provider);
    
    // Calculate dynamic backoff based on provider health
    const baseDelay = providerHealth.failureCount > 0 
      ? 1000 * Math.pow(2, providerHealth.failureCount) 
      : 100;
    
    const retryConfig = {
      maxRetries: 5,
      baseDelay,
      maxDelay: 3600000, // 1 hour
      backoffFactor: 2.5,
      
      retryCondition: (error) => {
        // Don't retry if property not found
        if (error.response?.status === 404) {
          return false;
        }
        
        // Retry on rate limiting
        if (error.response?.status === 429) {
          providerHealth.rateLimited = true;
          providerHealth.rateLimitReset = parseInt(
            error.response.headers['retry-after'] || '60'
          );
          return true;
        }
        
        // Retry on server errors
        return error.response?.status >= 500 || 
               error.code === 'ECONNRESET';
      },
      
      onRetry: (attempt, delay, error) => {
        providerHealth.failureCount++;
        providerHealth.lastFailure = new Date();
        
        console.log(`MLS sync retry ${attempt} for ${propertyId}`, {
          provider,
          delay,
          error: error.message
        });
      }
    };
    
    return this.syncWithBackoff(
      () => this.fetchFromMLS(propertyId, provider),
      retryConfig
    );
  }
  
  async syncWithBackoff(fn, config) {
    let attempt = 0;
    let lastError;
    
    while (attempt < config.maxRetries) {
      try {
        const result = await fn();
        
        // Update health on success
        this.updateProviderHealth('success');
        
        return result;
      } catch (error) {
        lastError = error;
        attempt++;
        
        if (!config.retryCondition(error) || attempt >= config.maxRetries) {
          break;
        }
        
        // Calculate delay with exponential backoff
        let delay = config.baseDelay * Math.pow(config.backoffFactor, attempt - 1);
        
        // Respect rate limit headers
        if (error.response?.status === 429 && error.response.headers['retry-after']) {
          delay = parseInt(error.response.headers['retry-after']) * 1000;
        }
        
        // Cap at max delay
        delay = Math.min(delay, config.maxDelay);
        
        // Add jitter
        const jitter = delay * (0.8 + Math.random() * 0.4);
        
        if (config.onRetry) {
          config.onRetry(attempt, jitter, error);
        }
        
        await this.sleep(jitter);
      }
    }
    
    this.updateProviderHealth('failure');
    throw lastError;
  }
  
  async fetchFromMLS(propertyId, provider) {
    const response = await axios.get(
      `${provider.baseUrl}/properties/${propertyId}`,
      {
        headers: {
          'Authorization': `Bearer ${provider.apiKey}`,
          'X-Request-ID': uuid()
        },
        timeout: 10000
      }
    );
    
    return response.data;
  }
  
  getProviderHealth(provider) {
    if (!this.providerHealth.has(provider.name)) {
      this.providerHealth.set(provider.name, {
        successCount: 0,
        failureCount: 0,
        rateLimited: false,
        rateLimitReset: 0,
        avgLatency: 0
      });
    }
    
    return this.providerHealth.get(provider.name);
  }
  
  updateProviderHealth(result) {
    // Track health metrics for monitoring
  }
  
  async batchSync(properties) {
    const results = {
      succeeded: [],
      failed: [],
      rateLimited: [],
      retries: 0
    };
    
    // Process with concurrency limit
    const concurrency = 5;
    const chunks = this.chunkArray(properties, concurrency);
    
    for (const chunk of chunks) {
      const promises = chunk.map(property => 
        this.syncProperty(property.id, property.provider)
          .then(() => results.succeeded.push(property))
          .catch(error => {
            if (error.response?.status === 429) {
              results.rateLimited.push(property);
            } else {
              results.failed.push(property);
            }
          })
      );
      
      await Promise.allSettled(promises);
      
      // Delay between chunks to avoid rate limiting
      if (chunks.indexOf(chunk) < chunks.length - 1) {
        await this.sleep(1000);
      }
    }
    
    return results;
  }
  
  chunkArray(array, size) {
    const chunks = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}
```

---

## Implementation Patterns

### 1. Basic Exponential Backoff

```javascript
class ExponentialBackoff {
  constructor(options = {}) {
    this.baseDelay = options.baseDelay || 100;
    this.maxDelay = options.maxDelay || 30000;
    this.factor = options.factor || 2;
    this.jitter = options.jitter || true;
  }
  
  getDelay(attempt) {
    // Calculate exponential delay
    let delay = this.baseDelay * Math.pow(this.factor, attempt);
    
    // Cap at max delay
    delay = Math.min(delay, this.maxDelay);
    
    // Add jitter if enabled
    if (this.jitter) {
      delay = delay * (0.5 + Math.random());
    }
    
    return Math.floor(delay);
  }
}

// Usage
const backoff = new ExponentialBackoff({
  baseDelay: 100,
  maxDelay: 10000,
  factor: 2,
  jitter: true
});

for (let attempt = 0; attempt < 5; attempt++) {
  console.log(`Attempt ${attempt + 1}: ${backoff.getDelay(attempt)}ms`);
}
// Output:
// Attempt 1: ~50-100ms
// Attempt 2: ~100-200ms
// Attempt 3: ~200-400ms
// Attempt 4: ~400-800ms
// Attempt 5: ~800-1600ms
```

### 2. Retry with Decorrelated Jitter

```javascript
class DecorrelatedJitterBackoff {
  constructor(options = {}) {
    this.baseDelay = options.baseDelay || 100;
    this.maxDelay = options.maxDelay || 30000;
    this.cap = options.cap || 60000;
  }
  
  getDelay(attempt, previousDelay) {
    if (attempt === 0) {
      return this.baseDelay;
    }
    
    // Decorrelated jitter: random between baseDelay and previousDelay * 3
    const min = this.baseDelay;
    const max = Math.min(previousDelay * 3, this.maxDelay);
    
    return min + Math.random() * (max - min);
  }
}

// Usage for long-running processes
class LongRunningRetry {
  constructor() {
    this.backoff = new DecorrelatedJitterBackoff({
      baseDelay: 1000,
      maxDelay: 3600000 // 1 hour
    });
  }
  
  async execute(fn) {
    let attempt = 0;
    let delay = this.baseDelay;
    
    while (true) {
      try {
        return await fn();
      } catch (error) {
        attempt++;
        
        if (!this.shouldRetry(error, attempt)) {
          throw error;
        }
        
        delay = this.backoff.getDelay(attempt, delay);
        
        console.log(`Retry ${attempt} after ${delay}ms`);
        await this.sleep(delay);
      }
    }
  }
}
```

### 3. Retry with Circuit Breaker

```javascript
class RetryWithCircuitBreaker {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.baseDelay = options.baseDelay || 100;
    
    // Circuit breaker settings
    this.failureThreshold = options.failureThreshold || 5;
    this.timeout = options.timeout || 60000;
    
    this.failureCount = 0;
    this.circuitOpen = false;
    this.circuitOpenUntil = null;
  }
  
  async execute(fn) {
    // Check circuit breaker
    if (this.circuitOpen) {
      if (Date.now() > this.circuitOpenUntil) {
        this.circuitOpen = false;
        this.failureCount = 0;
      } else {
        throw new Error('Circuit breaker is open');
      }
    }
    
    let attempt = 0;
    
    while (attempt < this.maxRetries) {
      try {
        const result = await fn();
        
        // Reset on success
        this.failureCount = 0;
        
        return result;
      } catch (error) {
        attempt++;
        this.failureCount++;
        
        // Check if we should open circuit
        if (this.failureCount >= this.failureThreshold) {
          this.circuitOpen = true;
          this.circuitOpenUntil = Date.now() + this.timeout;
          throw new Error('Circuit breaker opened due to failures');
        }
        
        if (attempt >= this.maxRetries) {
          throw error;
        }
        
        // Exponential backoff
        const delay = this.baseDelay * Math.pow(2, attempt - 1);
        await this.sleep(delay);
      }
    }
  }
}
```

### 4. Retry Queue with Backoff

```javascript
class RetryQueue {
  constructor(options = {}) {
    this.maxConcurrent = options.maxConcurrent || 5;
    this.maxRetries = options.maxRetries || 3;
    this.baseDelay = options.baseDelay || 1000;
    
    this.queue = [];
    this.active = 0;
    this.retryCounts = new Map();
  }
  
  add(task, id) {
    return new Promise((resolve, reject) => {
      this.queue.push({
        task,
        id,
        resolve,
        reject,
        attempt: 0
      });
      
      this.processQueue();
    });
  }
  
  async processQueue() {
    if (this.active >= this.maxConcurrent || this.queue.length === 0) {
      return;
    }
    
    const item = this.queue.shift();
    this.active++;
    
    try {
      const result = await item.task();
      item.resolve(result);
      
      // Clear retry count on success
      this.retryCounts.delete(item.id);
    } catch (error) {
      const retryCount = (this.retryCounts.get(item.id) || 0) + 1;
      
      if (retryCount <= this.maxRetries) {
        // Requeue with backoff
        this.retryCounts.set(item.id, retryCount);
        
        const delay = this.baseDelay * Math.pow(2, retryCount - 1);
        
        setTimeout(() => {
          this.queue.push(item);
          this.processQueue();
        }, delay);
        
        console.log(`Retry ${retryCount} for ${item.id} in ${delay}ms`);
      } else {
        // Max retries exceeded
        item.reject(error);
        this.retryCounts.delete(item.id);
      }
    } finally {
      this.active--;
      this.processQueue();
    }
  }
}

// Usage
const retryQueue = new RetryQueue({
  maxConcurrent: 3,
  maxRetries: 5,
  baseDelay: 500
});

// Add tasks
for (let i = 0; i < 10; i++) {
  retryQueue.add(async () => {
    const result = await axios.get(`http://api.example.com/data/${i}`);
    return result.data;
  }, `task-${i}`);
}
```

---

## Retry Strategies Comparison

| Strategy | Formula | Best For | Pros | Cons |
|----------|---------|----------|------|------|
| **Fixed** | constant | Simple retries | Simple | Can overload |
| **Linear** | n × base | Predictable | Easy to calculate | Doesn't back off enough |
| **Exponential** | base × 2^n | Most services | Good balance | Can get too long |
| **Full Jitter** | random(0, base × 2^n) | High concurrency | Prevents thundering herd | Less predictable |
| **Equal Jitter** | base × 2^n × (0.5 + random) | General purpose | Good distribution | Slightly complex |
| **Decorrelated** | random(base, prev × 3) | Long-running | Very adaptive | Harder to implement |

---

## HTTP Status Codes Retry Decision

| Status Code | Should Retry? | Notes |
|-------------|--------------|-------|
| **408** Request Timeout | ✅ Yes | Server timed out |
| **429** Too Many Requests | ✅ Yes | Check Retry-After header |
| **500** Internal Server Error | ✅ Yes | Server error |
| **502** Bad Gateway | ✅ Yes | Network issue |
| **503** Service Unavailable | ✅ Yes | Server overloaded |
| **504** Gateway Timeout | ✅ Yes | Upstream timeout |
| **400** Bad Request | ❌ No | Client error |
| **401** Unauthorized | ❌ No | Auth issue |
| **403** Forbidden | ❌ No | Permission issue |
| **404** Not Found | ❌ No | Resource missing |
| **409** Conflict | ⚠️ Maybe | Depends on idempotency |

---

## When to Use Retry with Backoff

### ✅ DO Use When:
- Transient failures expected
- Network operations
- Database connections
- API calls to external services
- File operations
- Any operation with potential temporary failures

### ❌ DON'T Use When:
- Operation is not idempotent
- Business logic forbids retry
- Error is permanent (404, 400)
- Real-time requirements (use fail fast)
- User is waiting (consider async)

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **No Max Retries** | Always set maximum attempts |
| **No Jitter** | Add jitter to prevent thundering herd |
| **Retrying Permanent Errors** | Check error types before retry |
| **Too Many Retries** | Balance with business needs |
| **No Monitoring** | Track retry rates and success |
| **Retry Storm** | Use circuit breaker with retry |
| **Ignoring Retry-After** | Respect rate limit headers |

---

## Summary Checklist

| Aspect | Implementation |
|--------|---------------|
| **Max Retries** | 3-5 for most cases |
| **Base Delay** | 100-1000ms depending on service |
| **Backoff Factor** | 2 for exponential |
| **Jitter** | Full or equal jitter |
| **Retryable Errors** | 5xx, network errors, timeouts |
| **Idempotency** | Ensure operations can be retried |
| **Monitoring** | Track retry rates and latency |
| **Circuit Breaker** | Combine for resilience |

**Bottom Line:** Retry with exponential backoff is essential for handling transient failures. Always add jitter, respect rate limits, and combine with circuit breakers for comprehensive resilience.