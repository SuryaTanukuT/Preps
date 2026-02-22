# Fallback Pattern

## Definition
The **Fallback pattern** provides an alternative response or behavior when a primary operation fails. It's a graceful degradation strategy that ensures the system can still function—even if in a reduced capacity—when dependencies are unavailable or operations fail.

Fallbacks can return default values, cached data, simplified responses, or trigger alternative flows.

---

## Core Concept
```
                    NORMAL FLOW
    ┌─────────────────────────────────────┐
    │  Request ──► Primary Operation      │
    │                 │                    │
    │                 ▼                    │
    │            Success ──► Return Result │
    └───────────────────────────────────────┘

                    FALLBACK FLOW
    ┌─────────────────────────────────────┐
    │  Request ──► Primary Operation      │
    │                 │                    │
    │                 ▼                    │
    │            Failure ──► Fallback      │
    │                      │               │
    │                      ▼               │
    │              Return Alternative      │
    └───────────────────────────────────────┘
```

---

## Types of Fallbacks

| Type | Description | Example |
|------|-------------|---------|
| **Static Fallback** | Return hardcoded default | Empty array, "Service Unavailable" |
| **Cached Fallback** | Return previously cached data | Stale product details |
| **Degraded Fallback** | Return simplified response | Basic info instead of rich data |
| **Alternative Service** | Call backup service | Secondary payment gateway |
| **Queued Fallback** | Queue for later processing | "We'll process this offline" |
| **Null Fallback** | Return null/empty | Optional feature disabled |

---

## Good vs Bad Example

### Bad Example (No Fallback)
```javascript
// BAD: No fallback, complete failure
app.get('/api/products/:id', async (req, res) => {
  try {
    const product = await productService.getProduct(req.params.id);
    res.json(product);
  } catch (error) {
    // Just fail - user sees 500 error
    res.status(500).json({ error: 'Failed to get product' });
  }
});

// When product service is down:
// - User sees error page
// - No product information at all
// - Bad user experience
// - Lost sales opportunity
```

### Good Example (With Fallback)
```javascript
// GOOD: Multiple fallback layers
app.get('/api/products/:id', async (req, res) => {
  let product = null;
  let source = 'primary';
  
  try {
    // Try primary source
    product = await productService.getProduct(req.params.id);
  } catch (primaryError) {
    console.log('Primary failed, trying cache');
    
    try {
      // Fallback to cache
      product = await cache.get(`product:${req.params.id}`);
      source = 'cache';
    } catch (cacheError) {
      console.log('Cache failed, using default');
      
      // Final fallback - return basic info
      product = {
        id: req.params.id,
        name: 'Product temporarily unavailable',
        price: null,
        inStock: false,
        fallback: true
      };
      source = 'default';
    }
  }
  
  // Return with metadata
  res.json({
    ...product,
    _meta: {
      source,
      timestamp: new Date().toISOString()
    }
  });
});
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Account Balance Display

```javascript
// ============ MULTI-LAYER FALLBACK ============
class BalanceService {
  constructor() {
    this.cache = new RedisCache();
    this.db = new Database();
    this.fallbackService = new FallbackBalanceService();
  }
  
  async getAccountBalance(accountId, userId) {
    const startTime = Date.now();
    const context = { accountId, userId, attempt: 1 };
    
    try {
      // Layer 1: Real-time balance from core banking
      return await this.getRealtimeBalance(accountId, userId);
    } catch (error) {
      console.log(`Layer 1 failed: ${error.message}`);
      
      // Layer 2: Cached balance from 5 minutes ago
      try {
        const cached = await this.getCachedBalance(accountId);
        if (cached) {
          return this.enhanceWithWarning(cached, 'stale');
        }
      } catch (cacheError) {
        console.log(`Layer 2 failed: ${cacheError.message}`);
      }
      
      // Layer 3: Previous day's closing balance
      try {
        const previousDay = await this.getPreviousDayBalance(accountId);
        if (previousDay) {
          return this.enhanceWithWarning(previousDay, 'yesterday');
        }
      } catch (historicalError) {
        console.log(`Layer 3 failed: ${historicalError.message}`);
      }
      
      // Layer 4: Default minimum balance
      return this.getMinimumBalanceFallback(accountId, userId);
    }
  }
  
  async getRealtimeBalance(accountId, userId) {
    // Primary: Call core banking system
    const response = await axios.post(`${CORE_BANKING_URL}/balance`, {
      accountId,
      userId,
      requestId: uuid()
    }, { timeout: 2000 });
    
    // Update cache
    await this.cache.setex(
      `balance:${accountId}`, 
      300, // 5 minutes
      JSON.stringify(response.data)
    );
    
    return {
      balance: response.data.available,
      ledger: response.data.ledger,
      currency: response.data.currency,
      lastUpdated: new Date().toISOString(),
      source: 'realtime'
    };
  }
  
  async getCachedBalance(accountId) {
    const cached = await this.cache.get(`balance:${accountId}`);
    
    if (!cached) {
      return null;
    }
    
    const data = JSON.parse(cached);
    
    return {
      balance: data.balance,
      ledger: data.ledger,
      currency: data.currency,
      lastUpdated: data.timestamp,
      source: 'cache',
      warning: 'Balance may be up to 5 minutes old'
    };
  }
  
  async getPreviousDayBalance(accountId) {
    // Query data warehouse for yesterday's closing balance
    const result = await this.dataWarehouse.query(`
      SELECT closing_balance, currency, balance_date
      FROM daily_balances
      WHERE account_id = $1
        AND balance_date = CURRENT_DATE - 1
    `, [accountId]);
    
    if (result.rows.length === 0) {
      return null;
    }
    
    return {
      balance: result.rows[0].closing_balance,
      currency: result.rows[0].currency,
      asOf: result.rows[0].balance_date,
      source: 'historical',
      warning: 'Showing yesterday\'s closing balance'
    };
  }
  
  getMinimumBalanceFallback(accountId, userId) {
    // Ultimate fallback - show minimum guaranteed amount
    return {
      balance: 100.00, // Minimum guaranteed balance
      currency: 'USD',
      source: 'minimum-guarantee',
      warning: 'Unable to retrieve current balance. Showing minimum guaranteed amount.',
      accountId,
      timestamp: new Date().toISOString(),
      helpText: 'Please try again later or contact support'
    };
  }
  
  enhanceWithWarning(data, type) {
    const warnings = {
      stale: 'Balance may not reflect recent transactions',
      yesterday: 'Showing yesterday\'s closing balance'
    };
    
    return {
      ...data,
      warning: warnings[type],
      timestamp: new Date().toISOString()
    };
  }
}

// ============ FALLBACK FOR TRANSACTIONS ============
class TransactionService {
  async processTransaction(transaction) {
    const strategies = [
      this.processRealtime.bind(this),
      this.processBatch.bind(this),
      this.queueForLater.bind(this)
    ];
    
    for (const strategy of strategies) {
      try {
        return await strategy(transaction);
      } catch (error) {
        console.log(`Strategy failed: ${error.message}`);
        // Continue to next fallback
      }
    }
    
    // All strategies failed
    throw new Error('Unable to process transaction');
  }
  
  async processRealtime(transaction) {
    // Try real-time processing
    return await axios.post(`${PAYMENT_GATEWAY}/process`, transaction, {
      timeout: 3000
    });
  }
  
  async processBatch(transaction) {
    // Fallback to batch processing
    return await axios.post(`${BATCH_PROCESSOR}/queue`, transaction);
  }
  
  async queueForLater(transaction) {
    // Last resort - queue for manual processing
    const queueId = await this.transactionQueue.add(transaction);
    
    return {
      status: 'queued',
      queueId,
      message: 'Transaction queued for processing',
      estimatedCompletion: Date.now() + 3600000 // 1 hour
    };
  }
}
```

### 2. E-commerce - Product Catalog

```javascript
// ============ PRODUCT DETAILS WITH FALLBACK ============
class ProductCatalogService {
  constructor() {
    this.cache = new RedisCache();
    this.searchService = new SearchService();
    this.recommendationService = new RecommendationService();
    this.reviewService = new ReviewService();
  }
  
  async getProductDetails(productId, userId) {
    const result = {
      product: null,
      recommendations: [],
      reviews: [],
      sources: {}
    };
    
    // Parallel execution with fallbacks
    const promises = [
      this.getProductWithFallback(productId).then(data => {
        result.product = data;
        result.sources.product = data.source;
      }),
      
      this.getRecommendationsWithFallback(userId, productId).then(data => {
        result.recommendations = data;
        result.sources.recommendations = data.source;
      }),
      
      this.getReviewsWithFallback(productId).then(data => {
        result.reviews = data;
        result.sources.reviews = data.source;
      })
    ];
    
    await Promise.allSettled(promises);
    
    // Ensure we always have something
    if (!result.product) {
      result.product = this.getBasicProductFallback(productId);
    }
    
    return result;
  }
  
  async getProductWithFallback(productId) {
    // Level 1: Primary product service
    try {
      const product = await this.searchService.getProduct(productId);
      
      // Update cache
      await this.cache.setex(`product:${productId}`, 3600, JSON.stringify(product));
      
      return { ...product, source: 'primary' };
    } catch (primaryError) {
      console.log('Primary product service failed');
      
      // Level 2: Cache
      try {
        const cached = await this.cache.get(`product:${productId}`);
        if (cached) {
          const product = JSON.parse(cached);
          return { 
            ...product, 
            source: 'cache',
            warning: 'Product information may be out of date'
          };
        }
      } catch (cacheError) {
        console.log('Cache failed');
      }
      
      // Level 3: Search index
      try {
        const searchResult = await this.searchService.search({
          filters: { productId },
          fields: ['id', 'name', 'price', 'category']
        });
        
        if (searchResult.length > 0) {
          return { 
            ...searchResult[0], 
            source: 'search-index',
            warning: 'Limited product information available'
          };
        }
      } catch (searchError) {
        console.log('Search index failed');
      }
      
      // Level 4: Default fallback
      throw new Error('All product sources failed');
    }
  }
  
  async getRecommendationsWithFallback(userId, productId) {
    const fallbackStrategies = [
      async () => {
        // Primary: Personalized recommendations
        return {
          items: await this.recommendationService.getForUser(userId),
          source: 'personalized'
        };
      },
      
      async () => {
        // Fallback 1: Product-based recommendations
        return {
          items: await this.recommendationService.getForProduct(productId),
          source: 'product-based'
        };
      },
      
      async () => {
        // Fallback 2: Popular items
        return {
          items: await this.searchService.getPopular(10),
          source: 'popular'
        };
      },
      
      async () => {
        // Fallback 3: Cached recommendations
        const cached = await this.cache.get(`recs:${userId}:fallback`);
        if (cached) {
          return {
            items: JSON.parse(cached),
            source: 'cached'
          };
        }
        throw new Error('No cached recommendations');
      },
      
      async () => {
        // Final fallback: Empty array
        return {
          items: [],
          source: 'none'
        };
      }
    ];
    
    for (const strategy of fallbackStrategies) {
      try {
        return await strategy();
      } catch {
        continue;
      }
    }
  }
  
  async getReviewsWithFallback(productId) {
    try {
      // Primary review service
      const reviews = await this.reviewService.getReviews(productId);
      return { items: reviews, source: 'primary' };
    } catch {
      // Fallback: Aggregate rating only
      const rating = await this.cache.get(`rating:${productId}`);
      
      if (rating) {
        return {
          items: [],
          summary: {
            average: JSON.parse(rating),
            count: 0,
            source: 'aggregate'
          },
          source: 'aggregate'
        };
      }
      
      // Ultimate fallback
      return {
        items: [],
        summary: null,
        source: 'none'
      };
    }
  }
  
  getBasicProductFallback(productId) {
    // Absolute minimum product info
    return {
      id: productId,
      name: 'Product Details Unavailable',
      price: null,
      inStock: false,
      description: 'We are currently unable to load product details.',
      fallback: true,
      source: 'default'
    };
  }
}

// ============ CHECKOUT WITH FALLBACK ============
class CheckoutService {
  async processCheckout(order) {
    const fallbackHandlers = {
      inventory: this.inventoryFallback.bind(this),
      payment: this.paymentFallback.bind(this),
      shipping: this.shippingFallback.bind(this),
      email: this.emailFallback.bind(this)
    };
    
    const results = {};
    const failures = [];
    
    // Try each step with fallback
    for (const [step, handler] of Object.entries(fallbackHandlers)) {
      try {
        results[step] = await handler(order);
      } catch (error) {
        failures.push({ step, error: error.message });
        results[step] = { status: 'failed', fallback: true };
      }
    }
    
    // Determine overall status
    const criticalSteps = ['inventory', 'payment'];
    const criticalFailed = criticalSteps.some(step => 
      failures.some(f => f.step === step)
    );
    
    if (criticalFailed) {
      return {
        status: 'failed',
        failures,
        message: 'Unable to complete checkout',
        results
      };
    }
    
    // Partial success
    return {
      status: 'partial',
      failures,
      message: failures.length > 0 
        ? 'Checkout completed with some issues' 
        : 'Checkout completed',
      results
    };
  }
  
  async inventoryFallback(order) {
    const strategies = [
      // Strategy 1: Real-time check
      async () => {
        const inventory = await this.inventoryService.check(order.items);
        if (!inventory.available) {
          throw new Error('Out of stock');
        }
        return { status: 'available', method: 'realtime' };
      },
      
      // Strategy 2: Allow backorder
      async () => {
        const backorder = await this.inventoryService.backorder(order.items);
        return { 
          status: 'backordered', 
          method: 'backorder',
          estimatedRestock: backorder.estimatedDate
        };
      },
      
      // Strategy 3: Split shipment
      async () => {
        const available = await this.inventoryService.getAvailableItems(order.items);
        const unavailable = order.items.filter(i => !available.includes(i.id));
        
        return {
          status: 'partial',
          method: 'split',
          available,
          unavailable,
          message: `${unavailable.length} items on backorder`
        };
      },
      
      // Strategy 4: Queue for fulfillment
      async () => {
        await this.fulfillmentQueue.add(order);
        return {
          status: 'queued',
          method: 'async',
          message: 'Order queued for processing'
        };
      }
    ];
    
    for (const strategy of strategies) {
      try {
        return await strategy();
      } catch {
        continue;
      }
    }
    
    throw new Error('All inventory strategies failed');
  }
  
  async paymentFallback(order) {
    const gateways = ['stripe', 'paypal', 'braintree', 'authorize.net'];
    
    for (const gateway of gateways) {
      try {
        const result = await this.tryPaymentGateway(gateway, order);
        return {
          status: 'success',
          gateway,
          transactionId: result.id
        };
      } catch {
        console.log(`Gateway ${gateway} failed, trying next`);
      }
    }
    
    // Final fallback - store payment method for later
    await this.paymentQueue.add({
      orderId: order.id,
      amount: order.total,
      method: order.payment.method,
      retryCount: 0
    });
    
    return {
      status: 'queued',
      message: 'Payment will be processed offline'
    };
  }
  
  async shippingFallback(order) {
    try {
      // Try primary carrier
      return await this.shippingService.createLabel(order);
    } catch {
      // Fallback to different carrier
      const alternatives = ['UPS', 'FedEx', 'USPS', 'DHL'];
      
      for (const carrier of alternatives) {
        try {
          return await this.alternateShipping.createLabel(order, carrier);
        } catch {
          continue;
        }
      }
      
      // Manual processing
      return {
        status: 'manual',
        message: 'Shipping label will be created manually'
      };
    }
  }
  
  async emailFallback(order) {
    try {
      await this.emailService.sendOrderConfirmation(order);
    } catch {
      // Queue for retry
      await this.emailQueue.add({
        type: 'order-confirmation',
        orderId: order.id,
        userId: order.userId,
        attempts: 0
      });
    }
  }
}
```

### 3. Real Estate - Property Search

```javascript
// ============ SEARCH WITH FALLBACK ============
class PropertySearchService {
  async search(criteria) {
    const searchStrategies = [
      this.fullTextSearch.bind(this),
      this.geoSearch.bind(this),
      this.cachedSearch.bind(this),
      this.basicSearch.bind(this),
      this.defaultResults.bind(this)
    ];
    
    let lastError = null;
    
    for (const strategy of searchStrategies) {
      try {
        const results = await strategy(criteria);
        
        if (results && results.length > 0) {
          return {
            properties: results,
            strategy: strategy.name,
            warning: lastError ? 'Using fallback search' : null
          };
        }
      } catch (error) {
        lastError = error;
        console.log(`Strategy ${strategy.name} failed:`, error.message);
      }
    }
    
    // Ultimate fallback
    return {
      properties: [],
      strategy: 'none',
      message: 'No properties found matching criteria'
    };
  }
  
  async fullTextSearch(criteria) {
    // Primary: Elasticsearch full-text search
    return await this.elasticsearch.search({
      index: 'properties',
      body: {
        query: {
          multi_match: {
            query: criteria.query,
            fields: ['title^3', 'description', 'address']
          }
        },
        filter: {
          range: {
            price: { gte: criteria.minPrice, lte: criteria.maxPrice }
          }
        }
      }
    });
  }
  
  async geoSearch(criteria) {
    // Fallback 1: Geo-spatial search
    return await this.postgres.query(`
      SELECT *, 
             ST_Distance(location, ST_MakePoint($1, $2)) as distance
      FROM properties
      WHERE ST_DWithin(location, ST_MakePoint($1, $2), $3)
        AND price BETWEEN $4 AND $5
      ORDER BY distance
      LIMIT 50
    `, [criteria.lng, criteria.lat, criteria.radius, criteria.minPrice, criteria.maxPrice]);
  }
  
  async cachedSearch(criteria) {
    // Fallback 2: Cached results
    const cacheKey = `search:${JSON.stringify(criteria)}`;
    const cached = await this.redis.get(cacheKey);
    
    if (!cached) {
      throw new Error('No cached results');
    }
    
    return JSON.parse(cached);
  }
  
  async basicSearch(criteria) {
    // Fallback 3: Basic database query
    return await this.postgres.query(`
      SELECT id, address, price, bedrooms, bathrooms
      FROM properties
      WHERE price BETWEEN $1 AND $2
        AND bedrooms >= $3
      LIMIT 50
    `, [criteria.minPrice, criteria.maxPrice, criteria.minBeds || 0]);
  }
  
  async defaultResults(criteria) {
    // Ultimate fallback: Show recent listings
    return await this.postgres.query(`
      SELECT id, address, price, bedrooms, bathrooms
      FROM properties
      ORDER BY listed_at DESC
      LIMIT 20
    `);
  }
}

// ============ MAP TILES WITH FALLBACK ============
class MapTileService {
  async getTile(z, x, y) {
    const providers = [
      { name: 'mapbox', url: process.env.MAPBOX_URL },
      { name: 'google', url: process.env.GOOGLE_MAPS_URL },
      { name: 'osm', url: process.env.OSM_URL },
      { name: 'local', url: process.env.LOCAL_TILE_URL }
    ];
    
    for (const provider of providers) {
      try {
        const tile = await this.fetchTile(provider.url, z, x, y);
        
        // Cache successful tile
        await this.cacheTile(z, x, y, tile, provider.name);
        
        return tile;
      } catch (error) {
        console.log(`Provider ${provider.name} failed`);
        
        // Try cached version
        const cached = await this.getCachedTile(z, x, y);
        if (cached) {
          return {
            ...cached,
            warning: 'Using cached map tile'
          };
        }
      }
    }
    
    // All providers failed, return placeholder
    return this.getPlaceholderTile(z, x, y);
  }
  
  async getPlaceholderTile(z, x, y) {
    // Return a simple grid pattern
    return {
      type: 'placeholder',
      content: this.generateGridTile(z, x, y),
      warning: 'Map temporarily unavailable'
    };
  }
  
  generateGridTile(z, x, y) {
    // Generate simple grid representation
    const canvas = createCanvas(256, 256);
    const ctx = canvas.getContext('2d');
    
    ctx.fillStyle = '#f0f0f0';
    ctx.fillRect(0, 0, 256, 256);
    
    ctx.strokeStyle = '#ccc';
    ctx.strokeRect(0, 0, 256, 256);
    
    ctx.fillStyle = '#666';
    ctx.font = '12px Arial';
    ctx.fillText(`${z}/${x}/${y}`, 10, 20);
    ctx.fillText('Map Unavailable', 10, 40);
    
    return canvas.toBuffer();
  }
}
```

---

## Implementation Patterns

### 1. Hierarchical Fallback Pattern

```javascript
class HierarchicalFallback {
  constructor() {
    this.fallbacks = [];
  }
  
  addFallback(priority, name, handler) {
    this.fallbacks.push({ priority, name, handler });
    // Sort by priority (lower number = higher priority)
    this.fallbacks.sort((a, b) => a.priority - b.priority);
  }
  
  async execute(context) {
    const errors = [];
    
    for (const fallback of this.fallbacks) {
      try {
        console.log(`Trying fallback: ${fallback.name}`);
        
        const result = await fallback.handler(context);
        
        return {
          ...result,
          _fallback: {
            used: fallback.name,
            attempted: this.fallbacks.slice(0, this.fallbacks.indexOf(fallback) + 1)
              .map(f => f.name),
            errors: errors.length > 0 ? errors : undefined
          }
        };
      } catch (error) {
        errors.push({ fallback: fallback.name, error: error.message });
        console.log(`Fallback ${fallback.name} failed:`, error.message);
      }
    }
    
    throw new Error('All fallbacks exhausted', { cause: errors });
  }
}

// Usage
const productFallback = new HierarchicalFallback();
productFallback.addFallback(1, 'primary-cache', getFromCache);
productFallback.addFallback(2, 'database', getFromDatabase);
productFallback.addFallback(3, 'search-index', getFromSearch);
productFallback.addFallback(4, 'default', getDefaultProduct);
```

### 2. Circuit Breaker with Fallback

```javascript
class CircuitBreakerWithFallback {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.timeout = options.timeout || 60000;
    this.fallbacks = options.fallbacks || [];
    this.state = 'CLOSED';
    this.failureCount = 0;
    this.nextAttempt = null;
  }
  
  async execute(primaryFn, context) {
    // Check circuit state
    if (this.state === 'OPEN') {
      if (Date.now() >= this.nextAttempt) {
        this.state = 'HALF_OPEN';
      } else {
        // Circuit open, use fallback directly
        return this.tryFallbacks(context);
      }
    }
    
    try {
      const result = await primaryFn();
      
      // Success - reset circuit
      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }
      
      return result;
    } catch (primaryError) {
      // Primary failed, increment counter
      this.failureCount++;
      
      if (this.state === 'CLOSED' && this.failureCount >= this.failureThreshold) {
        this.state = 'OPEN';
        this.nextAttempt = Date.now() + this.timeout;
      } else if (this.state === 'HALF_OPEN') {
        this.state = 'OPEN';
        this.nextAttempt = Date.now() + this.timeout;
      }
      
      // Try fallbacks
      return this.tryFallbacks(context, primaryError);
    }
  }
  
  async tryFallbacks(context, originalError = null) {
    for (const fallback of this.fallbacks) {
      try {
        const result = await fallback.handler(context);
        return {
          ...result,
          _fallback: {
            used: fallback.name,
            originalError: originalError?.message
          }
        };
      } catch (fallbackError) {
        console.log(`Fallback ${fallback.name} failed:`, fallbackError.message);
      }
    }
    
    throw new Error('All fallbacks failed', { cause: originalError });
  }
}
```

### 3. Stale-While-Revalidate Pattern

```javascript
class StaleWhileRevalidate {
  constructor(fetchFn, options = {}) {
    this.fetchFn = fetchFn;
    this.staleTTL = options.staleTTL || 3600000; // 1 hour
    this.maxStale = options.maxStale || 86400000; // 24 hours
    this.cache = new Map();
  }
  
  async get(key) {
    const cached = this.cache.get(key);
    const now = Date.now();
    
    // No cache - fetch fresh
    if (!cached) {
      return this.fetchAndCache(key);
    }
    
    // Cache is fresh - return immediately, revalidate in background
    if (now - cached.timestamp < this.staleTTL) {
      // Background revalidation
      if (now - cached.lastRevalidated > this.staleTTL / 2) {
        this.revalidateInBackground(key);
      }
      
      return {
        ...cached.data,
        _source: 'cache-fresh'
      };
    }
    
    // Cache is stale but within maxStale - return stale, revalidate
    if (now - cached.timestamp < this.maxStale) {
      // Trigger revalidation
      this.revalidateInBackground(key);
      
      return {
        ...cached.data,
        _source: 'cache-stale',
        _warning: 'Data may be outdated'
      };
    }
    
    // Cache too old - fetch fresh
    return this.fetchAndCache(key);
  }
  
  async fetchAndCache(key) {
    try {
      const data = await this.fetchFn(key);
      
      this.cache.set(key, {
        data,
        timestamp: Date.now(),
        lastRevalidated: Date.now()
      });
      
      return { ...data, _source: 'fresh' };
    } catch (error) {
      // If fetch fails, try to return any cached data
      const cached = this.cache.get(key);
      
      if (cached) {
        return {
          ...cached.data,
          _source: 'cache-emergency',
          _warning: 'Using cached data due to service outage'
        };
      }
      
      throw error;
    }
  }
  
  async revalidateInBackground(key) {
    if (this.revalidationInProgress?.has(key)) {
      return;
    }
    
    this.revalidationInProgress = this.revalidationInProgress || new Set();
    this.revalidationInProgress.add(key);
    
    try {
      const data = await this.fetchFn(key);
      
      this.cache.set(key, {
        data,
        timestamp: Date.now(),
        lastRevalidated: Date.now()
      });
    } catch (error) {
      console.log(`Background revalidation failed for ${key}:`, error.message);
    } finally {
      this.revalidationInProgress.delete(key);
    }
  }
}

// Usage
const productCache = new StaleWhileRevalidate(
  async (productId) => {
    const response = await axios.get(`/api/products/${productId}`);
    return response.data;
  },
  { staleTTL: 300000, maxStale: 86400000 }
);
```

### 4. Fallback Chain with Timeout

```javascript
class FallbackChain {
  constructor(options = {}) {
    this.timeout = options.timeout || 5000;
    this.strategies = [];
  }
  
  addStrategy(name, fn, options = {}) {
    this.strategies.push({
      name,
      fn,
      timeout: options.timeout || this.timeout
    });
    
    return this;
  }
  
  async execute(context) {
    const errors = [];
    
    for (const strategy of this.strategies) {
      try {
        const result = await Promise.race([
          strategy.fn(context),
          new Promise((_, reject) => 
            setTimeout(() => reject(new Error('Timeout')), strategy.timeout)
          )
        ]);
        
        return {
          ...result,
          _fallback: {
            strategy: strategy.name,
            attempted: this.strategies
              .slice(0, this.strategies.indexOf(strategy) + 1)
              .map(s => s.name),
            errors: errors.length > 0 ? errors : undefined
          }
        };
      } catch (error) {
        errors.push({
          strategy: strategy.name,
          error: error.message,
          time: new Date().toISOString()
        });
        
        console.log(`Strategy ${strategy.name} failed:`, error.message);
      }
    }
    
    throw new Error('All fallback strategies failed', { cause: errors });
  }
}

// Usage
const paymentFallback = new FallbackChain({ timeout: 3000 })
  .addStrategy('stripe', async (payment) => {
    return await stripe.charges.create(payment);
  })
  .addStrategy('paypal', async (payment) => {
    return await paypal.payment.create(payment);
  })
  .addStrategy('braintree', async (payment) => {
    return await braintree.transaction.sale(payment);
  })
  .addStrategy('queue', async (payment) => {
    const jobId = await paymentQueue.add(payment);
    return { status: 'queued', jobId, message: 'Payment queued for processing' };
  });
```

---

## Fallback Strategies Matrix

| Strategy | Speed | Reliability | Use Case |
|----------|-------|-------------|----------|
| **Static Default** | Instant | 100% | Non-critical data |
| **Cache** | Very Fast | Medium | Frequently accessed data |
| **Stale Cache** | Fast | Medium | When freshness isn't critical |
| **Alternative Service** | Variable | High | Critical operations |
| **Degraded Mode** | Fast | High | Complex features |
| **Queued Processing** | Slow | High | Async operations |
| **Manual Processing** | Very Slow | Highest | Last resort |

---

## When to Use Fallback

### ✅ DO Use When:
- External services may fail
- User experience is critical
- Data can be stale
- Operations can be degraded
- Multiple service options exist
- Async processing is acceptable

### ❌ DON'T Use When:
- Data must be real-time
- Financial transactions require certainty
- Security validation fails
- Legal/compliance requires fresh data
- Fallback is worse than failure

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Masking Critical Failures** | Log all fallback usage, alert on patterns |
| **Performance Impact** | Cache fallback results, use timeouts |
| **Infinite Loops** | Limit fallback chain depth |
| **Stale Data Problems** | Add TTL, clear warning |
| **Silent Degradation** | Notify users when using fallback |
| **Cost of Fallbacks** | Monitor fallback usage costs |

---

## Summary Checklist

| Aspect | Implementation |
|--------|---------------|
| **Multiple Layers** | 3-5 fallback strategies |
| **Timeouts** | Per-strategy timeouts |
| **Logging** | Log all fallback usage |
| **User Communication** | Clear warnings when degraded |
| **Metrics** | Track fallback rate by type |
| **Testing** | Test each fallback path |
| **Recovery** | Retry primary when possible |

**Bottom Line:** Fallbacks are essential for graceful degradation. Design them in layers, communicate clearly when they're used, and always monitor fallback rates to detect systemic issues.