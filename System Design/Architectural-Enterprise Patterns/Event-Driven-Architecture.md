# Event-Driven Architecture (EDA)

## Definition
**Event-Driven Architecture** is a software design pattern where systems communicate through the production, detection, and reaction to **events**. An event is a significant change in state that is meaningful to the system or its components.

In EDA, services are loosely coupled—they don't directly call each other. Instead, they emit events and react to events they care about. This enables asynchronous communication, scalability, and flexibility.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │         EVENT PRODUCERS             │
                    │  (Services that emit events)        │
                    └──────────────┬──────────────────────┘
                                   │
                    ┌──────────────▼──────────────────────┐
                    │         EVENT BROKER                 │
                    │  (Kafka, RabbitMQ, EventBridge)      │
                    │  ┌──────────────────────────────┐   │
                    │  │  Event 1 │ Event 2 │ Event 3 │   │
                    │  └──────────────────────────────┘   │
                    └──────────────┬──────────────────────┘
                                   │
            ┌──────────────────────┼──────────────────────┐
            │                      │                      │
    ┌───────▼───────┐      ┌───────▼───────┐      ┌───────▼───────┐
    │  CONSUMER 1   │      │  CONSUMER 2   │      │  CONSUMER 3   │
    │ (Order Service)│      │(Payment Service)│     │(Email Service)│
    │ - OrderCreated │      │ - OrderCreated│      │ - OrderCreated│
    │                │      │ - PaymentDone │      │ - OrderShipped│
    └───────────────┘      └───────────────┘      └───────────────┘
```

---

## Event-Driven vs Request-Response

| Aspect | Request-Response | Event-Driven |
|--------|-----------------|--------------|
| **Communication** | Synchronous | Asynchronous |
| **Coupling** | Tight (knows endpoint) | Loose (knows events) |
| **Scalability** | Limited by caller | Independent scaling |
| **Failure Handling** | Immediate error | Retry/DLQ |
| **Complexity** | Simple | More complex |
| **Visibility** | Easy to trace | Harder to trace |
| **Use Case** | CRUD, Queries | Workflows, Integrations |

---

## Good vs Bad Example

### Bad Example (Tight Coupling)
```javascript
// BAD: Services directly calling each other
class OrderService {
  async createOrder(orderData) {
    // Save order
    const order = await db.orders.save(orderData);
    
    // Directly call other services
    await inventoryService.updateStock(order.items); // If fails, order fails
    await paymentService.processPayment(order);      // Synchronous wait
    await emailService.sendConfirmation(order);      // Blocks response
    await analyticsService.trackOrder(order);        // Another dependency
    
    return order;
  }
}

// Problems:
// - Order service knows about all other services
// - If any service is slow, order creation is slow
// - If any service fails, order creation fails
// - Adding new service requires changing OrderService
```

### Good Example (Event-Driven)
```javascript
// GOOD: Services communicate through events
class OrderService {
  async createOrder(orderData) {
    // 1. Save order (local transaction)
    const order = await db.orders.save(orderData);
    
    // 2. Emit event (through outbox pattern)
    await this.eventBus.publish('order.created', {
      orderId: order.id,
      userId: order.userId,
      items: order.items,
      total: order.total,
      timestamp: new Date()
    });
    
    // 3. Return immediately
    return { 
      orderId: order.id, 
      status: 'processing' 
    };
  }
}

// Inventory Service (separate service)
class InventoryService {
  constructor() {
    eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
  }
  
  async handleOrderCreated(event) {
    console.log(`Processing inventory for order ${event.orderId}`);
    
    for (const item of event.items) {
      await inventory.reserve(item.productId, item.quantity);
    }
  }
}

// Payment Service (separate service)
class PaymentService {
  constructor() {
    eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
  }
  
  async handleOrderCreated(event) {
    console.log(`Processing payment for order ${event.orderId}`);
    await paymentGateway.charge(event.total);
    
    // Emit next event
    eventBus.publish('payment.processed', {
      orderId: event.orderId,
      status: 'success'
    });
  }
}

// Email Service (separate service)
class EmailService {
  constructor() {
    eventBus.subscribe('payment.processed', this.handlePaymentProcessed.bind(this));
  }
  
  async handlePaymentProcessed(event) {
    console.log(`Sending email for order ${event.orderId}`);
    await emailClient.sendConfirmation(event.orderId);
  }
}
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Fraud Detection System

```javascript
// ============ EVENT-DRIVEN FRAUD DETECTION ============
// Event Schema Registry
const eventSchema = {
  'transaction.created': {
    transactionId: 'string',
    accountId: 'string',
    amount: 'number',
    merchant: 'string',
    location: 'object',
    deviceId: 'string',
    timestamp: 'date'
  },
  
  'transaction.flagged': {
    transactionId: 'string',
    reason: 'string',
    riskScore: 'number',
    flags: ['string']
  },
  
  'account.locked': {
    accountId: 'string',
    reason: 'string',
    lockedBy: 'string',
    timestamp: 'date'
  },
  
  'customer.notified': {
    customerId: 'string',
    type: 'string',
    channel: 'string',
    timestamp: 'date'
  }
};

// ============ TRANSACTION SERVICE (PRODUCER) ============
class TransactionService {
  constructor(eventBus) {
    this.eventBus = eventBus;
  }
  
  async processTransaction(transactionData) {
    // Start saga
    const sagaId = uuid();
    
    // 1. Save transaction locally
    const transaction = await db.transactions.insert({
      ...transactionData,
      status: 'PENDING',
      createdAt: new Date()
    });
    
    // 2. Publish event for fraud detection
    await this.eventBus.publish('transaction.created', {
      transactionId: transaction.id,
      accountId: transactionData.accountId,
      amount: transactionData.amount,
      merchant: transactionData.merchant,
      location: transactionData.location,
      deviceId: transactionData.deviceId,
      timestamp: new Date(),
      metadata: {
        sagaId,
        source: 'atm'
      }
    });
    
    // 3. Return pending status
    return {
      transactionId: transaction.id,
      status: 'pending_review',
      estimatedCompletion: Date.now() + 5000
    };
  }
}

// ============ FRAUD DETECTION SERVICE (CONSUMER & PRODUCER) ============
class FraudDetectionService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.rules = [
      new VelocityCheckRule(),
      new LocationAnomalyRule(),
      new AmountThresholdRule(),
      new DeviceReputationRule()
    ];
    
    // Subscribe to events
    eventBus.subscribe('transaction.created', this.analyzeTransaction.bind(this));
    eventBus.subscribe('transaction.flagged', this.escalateFraud.bind(this));
  }
  
  async analyzeTransaction(event) {
    console.log(`Analyzing transaction ${event.transactionId}`);
    
    // Run all fraud rules in parallel
    const results = await Promise.all(
      this.rules.map(rule => rule.evaluate(event))
    );
    
    const flags = results.filter(r => r.flagged).map(r => r.reason);
    const riskScore = this.calculateRiskScore(results);
    
    if (flags.length > 0) {
      // Emit flagged event
      await this.eventBus.publish('transaction.flagged', {
        transactionId: event.transactionId,
        reason: flags.join(', '),
        riskScore,
        flags,
        transaction: event,
        timestamp: new Date()
      });
      
      // Log for metrics
      await this.logFraudEvent(event.transactionId, flags, riskScore);
    } else {
      // No fraud detected - proceed
      await this.eventBus.publish('transaction.approved', {
        transactionId: event.transactionId,
        riskScore,
        timestamp: new Date()
      });
    }
  }
  
  async escalateFraud(event) {
    // High-risk fraud requires immediate action
    
    // Lock account
    await this.eventBus.publish('account.locked', {
      accountId: event.transaction.accountId,
      reason: 'FRAUD_DETECTED',
      lockedBy: 'fraud-service',
      riskScore: event.riskScore,
      timestamp: new Date()
    });
    
    // Notify customer
    await this.eventBus.publish('customer.notified', {
      customerId: event.transaction.customerId,
      type: 'FRAUD_ALERT',
      channel: 'sms',
      data: {
        transactionId: event.transactionId,
        amount: event.transaction.amount,
        merchant: event.transaction.merchant
      },
      timestamp: new Date()
    });
    
    // Create case for investigation
    await this.eventBus.publish('fraud.case.created', {
      caseId: uuid(),
      transactionId: event.transactionId,
      riskScore: event.riskScore,
      flags: event.flags,
      assignedTo: 'fraud-team',
      timestamp: new Date()
    });
  }
}

// ============ ACCOUNT SERVICE (CONSUMER) ============
class AccountService {
  constructor(eventBus) {
    eventBus.subscribe('transaction.approved', this.updateBalance.bind(this));
    eventBus.subscribe('account.locked', this.lockAccount.bind(this));
    eventBus.subscribe('fraud.case.created', this.holdFunds.bind(this));
  }
  
  async updateBalance(event) {
    console.log(`Updating balance for transaction ${event.transactionId}`);
    
    await db.transaction(async (trx) => {
      // Update account balance
      await trx.accounts.updateBalance(
        event.accountId,
        event.amount,
        'debit'
      );
      
      // Record in ledger
      await trx.ledger.insert({
        transactionId: event.transactionId,
        accountId: event.accountId,
        amount: event.amount,
        type: 'debit',
        timestamp: event.timestamp
      });
    });
  }
  
  async lockAccount(event) {
    console.log(`Locking account ${event.accountId} due to fraud`);
    
    await db.accounts.update(event.accountId, {
      status: 'LOCKED',
      lockReason: event.reason,
      lockedAt: event.timestamp
    });
  }
  
  async holdFunds(event) {
    console.log(`Placing hold on funds for case ${event.caseId}`);
    
    await db.accounts.holdFunds(
      event.transactionId,
      event.riskScore > 80 ? 'full' : 'partial'
    );
  }
}

// ============ NOTIFICATION SERVICE (CONSUMER) ============
class NotificationService {
  constructor(eventBus) {
    eventBus.subscribe('account.locked', this.notifyAccountLocked.bind(this));
    eventBus.subscribe('transaction.approved', this.notifyTransaction.bind(this));
    eventBus.subscribe('fraud.case.created', this.notifyFraudTeam.bind(this));
  }
  
  async notifyAccountLocked(event) {
    // Send SMS
    await smsProvider.send({
      to: event.phoneNumber,
      message: `Your account has been locked due to suspicious activity. Please contact support.`
    });
    
    // Send email
    await emailProvider.send({
      to: event.email,
      subject: 'Account Security Alert',
      template: 'account-locked',
      data: event
    });
    
    // Push notification
    await pushProvider.send({
      userId: event.userId,
      title: 'Account Locked',
      body: 'Suspicious activity detected'
    });
  }
}
```

### 2. E-commerce - Order Fulfillment

```javascript
// ============ COMPLEX ORDER FULFILLMENT WORKFLOW ============
// Event definitions
const ecommerceEvents = {
  'order.placed': {
    orderId: 'string',
    customerId: 'string',
    items: [{ productId: 'string', quantity: 'number' }],
    total: 'number',
    paymentMethod: 'string',
    shippingAddress: 'object'
  },
  
  'inventory.reserved': {
    orderId: 'string',
    items: [{ productId: 'string', quantity: 'number', location: 'string' }],
    expiryTime: 'date'
  },
  
  'payment.authorized': {
    orderId: 'string',
    authorizationCode: 'string',
    amount: 'number',
    provider: 'string'
  },
  
  'payment.captured': {
    orderId: 'string',
    captureId: 'string',
    amount: 'number',
    timestamp: 'date'
  },
  
  'order.confirmed': {
    orderId: 'string',
    confirmedAt: 'date',
    estimatedDelivery: 'date'
  },
  
  'shipment.created': {
    orderId: 'string',
    trackingNumber: 'string',
    carrier: 'string',
    items: [{ productId: 'string', quantity: 'number' }]
  },
  
  'order.delivered': {
    orderId: 'string',
    deliveredAt: 'date',
    signature: 'string?'
  },
  
  'order.cancelled': {
    orderId: 'string',
    reason: 'string',
    cancelledBy: 'string',
    refundIssued: 'boolean'
  }
};

// ============ ORDER SERVICE (PRODUCER) ============
class OrderService {
  constructor(eventBus) {
    this.eventBus = eventBus;
  }
  
  async placeOrder(orderData) {
    // Start distributed transaction
    const orderId = uuid();
    
    // Save order locally
    await db.orders.insert({
      id: orderId,
      ...orderData,
      status: 'PENDING',
      createdAt: new Date()
    });
    
    // Emit event to start workflow
    await this.eventBus.publish('order.placed', {
      orderId,
      customerId: orderData.customerId,
      items: orderData.items,
      total: orderData.total,
      paymentMethod: orderData.paymentMethod,
      shippingAddress: orderData.shippingAddress,
      timestamp: new Date()
    });
    
    return { orderId, status: 'processing' };
  }
  
  async cancelOrder(orderId, reason) {
    const order = await db.orders.findById(orderId);
    
    if (order.status === 'SHIPPED') {
      throw new Error('Cannot cancel shipped order');
    }
    
    await db.orders.update(orderId, { status: 'CANCELLING' });
    
    await this.eventBus.publish('order.cancellation.requested', {
      orderId,
      reason,
      requestedBy: 'customer',
      timestamp: new Date()
    });
  }
}

// ============ INVENTORY SERVICE ============
class InventoryService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    eventBus.subscribe('order.placed', this.reserveInventory.bind(this));
    eventBus.subscribe('order.cancelled', this.releaseInventory.bind(this));
    eventBus.subscribe('payment.failed', this.releaseInventory.bind(this));
  }
  
  async reserveInventory(event) {
    console.log(`Reserving inventory for order ${event.orderId}`);
    
    try {
      const reservations = [];
      
      for (const item of event.items) {
        // Find best warehouse
        const warehouse = await this.findNearestWarehouse(
          item.productId,
          event.shippingAddress
        );
        
        // Reserve stock
        const reservation = await this.reserveStock(
          item.productId,
          item.quantity,
          warehouse.id,
          event.orderId
        );
        
        reservations.push({
          productId: item.productId,
          quantity: item.quantity,
          location: warehouse.code,
          expiryTime: new Date(Date.now() + 3600000) // 1 hour hold
        });
      }
      
      // Emit success event
      await this.eventBus.publish('inventory.reserved', {
        orderId: event.orderId,
        items: reservations,
        timestamp: new Date()
      });
      
    } catch (error) {
      // Emit failure event
      await this.eventBus.publish('inventory.reservation.failed', {
        orderId: event.orderId,
        reason: error.message,
        items: event.items,
        timestamp: new Date()
      });
    }
  }
  
  async releaseInventory(event) {
    console.log(`Releasing inventory for cancelled order ${event.orderId}`);
    
    await db.inventory.releaseByOrderId(event.orderId);
    
    await this.eventBus.publish('inventory.released', {
      orderId: event.orderId,
      timestamp: new Date()
    });
  }
}

// ============ PAYMENT SERVICE ============
class PaymentService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    eventBus.subscribe('inventory.reserved', this.authorizePayment.bind(this));
    eventBus.subscribe('order.shipped', this.capturePayment.bind(this));
  }
  
  async authorizePayment(event) {
    console.log(`Authorizing payment for order ${event.orderId}`);
    
    // Get order details
    const order = await db.orders.findById(event.orderId);
    
    try {
      const auth = await paymentGateway.authorize({
        amount: order.total,
        method: order.paymentMethod,
        orderId: event.orderId
      });
      
      await this.eventBus.publish('payment.authorized', {
        orderId: event.orderId,
        authorizationCode: auth.code,
        amount: order.total,
        provider: auth.provider,
        expiresAt: new Date(Date.now() + 7 * 86400000), // 7 days
        timestamp: new Date()
      });
      
    } catch (error) {
      await this.eventBus.publish('payment.failed', {
        orderId: event.orderId,
        reason: error.message,
        timestamp: new Date()
      });
    }
  }
  
  async capturePayment(event) {
    console.log(`Capturing payment for shipped order ${event.orderId}`);
    
    const payment = await db.payments.findByOrderId(event.orderId);
    
    const capture = await paymentGateway.capture(payment.authorizationCode);
    
    await this.eventBus.publish('payment.captured', {
      orderId: event.orderId,
      captureId: capture.id,
      amount: capture.amount,
      timestamp: new Date()
    });
  }
}

// ============ SHIPPING SERVICE ============
class ShippingService {
  constructor(eventBus) {
    this.eventBus = eventBus;
    eventBus.subscribe('payment.authorized', this.createShipment.bind(this));
    eventBus.subscribe('order.cancelled', this.cancelShipment.bind(this));
  }
  
  async createShipment(event) {
    console.log(`Creating shipment for order ${event.orderId}`);
    
    const order = await db.orders.findById(event.orderId);
    
    // Get inventory locations
    const inventory = await db.inventory.findByOrderId(event.orderId);
    
    // Optimize shipping based on item locations
    const shipment = await shippingProvider.createShipment({
      orderId: event.orderId,
      items: inventory.items,
      address: order.shippingAddress,
      carrier: this.selectCarrier(inventory.items)
    });
    
    await this.eventBus.publish('shipment.created', {
      orderId: event.orderId,
      trackingNumber: shipment.trackingNumber,
      carrier: shipment.carrier,
      items: inventory.items,
      estimatedDelivery: shipment.estimatedDelivery,
      timestamp: new Date()
    });
    
    // Update order status
    await this.eventBus.publish('order.shipped', {
      orderId: event.orderId,
      shippedAt: new Date(),
      trackingNumber: shipment.trackingNumber
    });
  }
  
  async cancelShipment(event) {
    if (event.orderId) {
      await shippingProvider.cancelShipment(event.orderId);
    }
  }
}

// ============ NOTIFICATION SERVICE ============
class NotificationService {
  constructor(eventBus) {
    // Subscribe to multiple events
    const events = [
      'order.confirmed',
      'order.shipped',
      'order.delivered',
      'payment.failed',
      'inventory.reservation.failed'
    ];
    
    events.forEach(event => 
      eventBus.subscribe(event, this.handleNotification.bind(this))
    );
  }
  
  async handleNotification(event) {
    const templates = {
      'order.confirmed': 'Your order #{{orderId}} has been confirmed',
      'order.shipped': 'Your order #{{orderId}} has been shipped',
      'order.delivered': 'Your order #{{orderId}} has been delivered',
      'payment.failed': 'Payment failed for order #{{orderId}}',
      'inventory.reservation.failed': 'Some items in your order are unavailable'
    };
    
    const message = this.renderTemplate(
      templates[event.type],
      event.data
    );
    
    await this.sendNotification(event.customerId, message);
  }
}

// ============ ANALYTICS SERVICE ============
class AnalyticsService {
  constructor(eventBus) {
    // Subscribe to all events for analytics
    eventBus.subscribe('*', this.trackEvent.bind(this));
    
    // Real-time dashboards
    this.dashboards = {
      orders: new OrdersDashboard(),
      payments: new PaymentsDashboard(),
      inventory: new InventoryDashboard()
    };
  }
  
  async trackEvent(event) {
    // Store for analytics
    await db.analytics.insert({
      eventType: event.type,
      data: event.data,
      timestamp: event.timestamp || new Date()
    });
    
    // Update real-time metrics
    switch(event.type) {
      case 'order.placed':
        this.dashboards.orders.incrementPlaced();
        break;
      case 'payment.failed':
        this.dashboards.payments.incrementFailed();
        break;
      case 'inventory.reserved':
        this.dashboards.inventory.updateReservations(event.data.items);
        break;
    }
  }
}
```

### 3. Real Estate - Property Listing Platform

```javascript
// ============ PROPERTY LISTING EVENTS ============
const propertyEvents = {
  'property.listed': {
    propertyId: 'string',
    address: 'object',
    price: 'number',
    bedrooms: 'number',
    bathrooms: 'number',
    sqft: 'number',
    features: ['string'],
    images: ['string'],
    agentId: 'string',
    mlsId: 'string'
  },
  
  'property.price.changed': {
    propertyId: 'string',
    oldPrice: 'number',
    newPrice: 'number',
    reason: 'string',
    changedBy: 'string'
  },
  
  'property.sold': {
    propertyId: 'string',
    finalPrice: 'number',
    soldDate: 'date',
    buyerId: 'string',
    sellerId: 'string'
  },
  
  'property.viewed': {
    propertyId: 'string',
    userId: 'string?',
    sessionId: 'string',
    timestamp: 'date',
    source: 'web|mobile'
  },
  
  'offer.submitted': {
    offerId: 'string',
    propertyId: 'string',
    buyerId: 'string',
    amount: 'number',
    contingencies: ['string'],
    expiresAt: 'date'
  }
};

// ============ PROPERTY SERVICE ============
class PropertyService {
  constructor(eventBus) {
    this.eventBus = eventBus;
  }
  
  async listProperty(listingData) {
    const propertyId = uuid();
    
    // Save to database
    await db.properties.insert({
      id: propertyId,
      ...listingData,
      status: 'ACTIVE',
      listedAt: new Date()
    });
    
    // Emit events for downstream systems
    await Promise.all([
      this.eventBus.publish('property.listed', {
        propertyId,
        ...listingData,
        timestamp: new Date()
      }),
      
      this.eventBus.publish('search.index.required', {
        propertyId,
        operation: 'index',
        data: listingData
      }),
      
      this.eventBus.publish('image.processing.required', {
        propertyId,
        images: listingData.images
      })
    ]);
    
    return propertyId;
  }
  
  async updatePrice(propertyId, newPrice, reason, agentId) {
    const property = await db.properties.findById(propertyId);
    
    await db.properties.update(propertyId, {
      price: newPrice,
      updatedAt: new Date()
    });
    
    await db.priceHistory.insert({
      propertyId,
      oldPrice: property.price,
      newPrice,
      reason,
      changedBy: agentId,
      changedAt: new Date()
    });
    
    await this.eventBus.publish('property.price.changed', {
      propertyId,
      oldPrice: property.price,
      newPrice,
      reason,
      changedBy: agentId,
      timestamp: new Date()
    });
  }
}

// ============ SEARCH INDEX SERVICE ============
class SearchIndexService {
  constructor(eventBus) {
    eventBus.subscribe('property.listed', this.indexProperty.bind(this));
    eventBus.subscribe('property.price.changed', this.updatePrice.bind(this));
    eventBus.subscribe('property.sold', this.removeFromIndex.bind(this));
  }
  
  async indexProperty(event) {
    console.log(`Indexing property ${event.propertyId} for search`);
    
    await elasticsearch.index({
      index: 'properties',
      id: event.propertyId,
      body: {
        address: event.address,
        price: event.price,
        bedrooms: event.bedrooms,
        bathrooms: event.bathrooms,
        sqft: event.sqft,
        features: event.features,
        location: {
          lat: event.address.lat,
          lon: event.address.lng
        },
        listedAt: event.timestamp
      }
    });
  }
  
  async updatePrice(event) {
    await elasticsearch.update({
      index: 'properties',
      id: event.propertyId,
      body: {
        doc: {
          price: event.newPrice,
          updatedAt: event.timestamp
        }
      }
    });
  }
}

// ============ IMAGE PROCESSING SERVICE ============
class ImageProcessingService {
  constructor(eventBus) {
    eventBus.subscribe('image.processing.required', this.processImages.bind(this));
  }
  
  async processImages(event) {
    console.log(`Processing ${event.images.length} images for property ${event.propertyId}`);
    
    const versions = [];
    
    for (const image of event.images) {
      // Generate multiple sizes
      const processed = await this.generateImageVersions(image);
      versions.push(processed);
    }
    
    await db.propertyImages.insert({
      propertyId: event.propertyId,
      versions,
      processedAt: new Date()
    });
    
    await this.eventBus.publish('image.processing.completed', {
      propertyId: event.propertyId,
      imageCount: versions.length,
      timestamp: new Date()
    });
  }
  
  async generateImageVersions(imageUrl) {
    // Generate thumbnail, medium, large versions
    return {
      thumbnail: await this.resize(imageUrl, 150, 150),
      medium: await this.resize(imageUrl, 600, 400),
      large: await this.resize(imageUrl, 1200, 800),
      original: imageUrl
    };
  }
}

// ============ NOTIFICATION HUB ============
class NotificationHub {
  constructor(eventBus) {
    // Track user preferences
    this.userPreferences = new Map();
    
    eventBus.subscribe('property.listed', this.notifyInterestedBuyers.bind(this));
    eventBus.subscribe('property.price.changed', this.notifyPriceWatchers.bind(this));
    eventBus.subscribe('offer.submitted', this.notifyAgent.bind(this));
    eventBus.subscribe('property.viewed', this.trackInterest.bind(this));
  }
  
  async notifyInterestedBuyers(event) {
    // Find buyers interested in this area/price range
    const interested = await db.query(`
      SELECT user_id, email, phone 
      FROM buyer_preferences 
      WHERE price_range @> $1
        AND ST_DWithin(
          preferred_location,
          ST_MakePoint($2, $3),
          $4
        )
    `, [event.price, event.address.lng, event.address.lat, 10000]);
    
    for (const buyer of interested.rows) {
      await this.sendNotification(buyer, {
        type: 'NEW_LISTING',
        propertyId: event.propertyId,
        address: event.address,
        price: event.price,
        matchScore: this.calculateMatchScore(buyer, event)
      });
    }
  }
  
  async notifyPriceWatchers(event) {
    const watchers = await db.query(`
      SELECT user_id, email, threshold 
      FROM price_alerts 
      WHERE property_id = $1
    `, [event.propertyId]);
    
    for (const watcher of watchers.rows) {
      if (event.newPrice <= watcher.threshold) {
        await this.sendNotification(watcher, {
          type: 'PRICE_DROP',
          propertyId: event.propertyId,
          oldPrice: event.oldPrice,
          newPrice: event.newPrice,
          savings: event.oldPrice - event.newPrice
        });
      }
    }
  }
  
  async notifyAgent(event) {
    const property = await db.properties.findById(event.propertyId);
    
    await this.sendNotification(
      { userId: property.agentId },
      {
        type: 'OFFER_RECEIVED',
        offerId: event.offerId,
        propertyId: event.propertyId,
        amount: event.amount,
        expiresAt: event.expiresAt
      }
    );
  }
  
  async trackInterest(event) {
    // Update property popularity score
    await redis.zincrby(
      'property:views',
      1,
      event.propertyId
    );
    
    // Track user interest for recommendations
    if (event.userId) {
      await redis.sadd(
        `user:${event.userId}:viewed`,
        event.propertyId
      );
    }
  }
}

// ============ MARKET ANALYTICS SERVICE ============
class MarketAnalyticsService {
  constructor(eventBus) {
    eventBus.subscribe('property.listed', this.updateInventory.bind(this));
    eventBus.subscribe('property.sold', this.updateSales.bind(this));
    eventBus.subscribe('property.price.changed', this.updatePriceTrends.bind(this));
    
    // Materialized views for analytics
    this.views = {
      inventory: new InventoryView(),
      sales: new SalesView(),
      trends: new TrendsView()
    };
  }
  
  async updateInventory(event) {
    await this.views.inventory.add({
      propertyId: event.propertyId,
      price: event.price,
      bedrooms: event.bedrooms,
      location: event.address,
      listedAt: event.timestamp
    });
    
    // Update neighborhood stats
    await this.updateNeighborhoodStats(event.address.zipCode);
  }
  
  async updateSales(event) {
    await this.views.sales.add({
      propertyId: event.propertyId,
      finalPrice: event.finalPrice,
      soldDate: event.soldDate,
      daysOnMarket: this.calculateDaysOnMarket(event.propertyId)
    });
  }
  
  async updatePriceTrends(event) {
    await this.views.trends.recordPriceChange(
      event.propertyId,
      event.oldPrice,
      event.newPrice,
      event.timestamp
    );
  }
  
  async updateNeighborhoodStats(zipCode) {
    // Recalculate neighborhood aggregates
    const stats = await db.query(`
      SELECT 
        AVG(price) as avg_price,
        MEDIAN(price) as median_price,
        COUNT(*) as inventory_count,
        AVG(days_on_market) as avg_dom
      FROM properties
      WHERE zip_code = $1 AND status = 'ACTIVE'
    `, [zipCode]);
    
    await this.eventBus.publish('neighborhood.stats.updated', {
      zipCode,
      stats: stats.rows[0],
      timestamp: new Date()
    });
  }
}
```

---

## Event-Driven Architecture Patterns

### 1. Event Sourcing

```javascript
// Store all events as source of truth
class EventStore {
  constructor() {
    this.events = [];
  }
  
  async append(streamId, event) {
    await db.query(`
      INSERT INTO event_store 
      (stream_id, event_type, data, version, created_at)
      VALUES ($1, $2, $3, $4, $5)
    `, [streamId, event.type, event.data, event.version, new Date()]);
  }
  
  async readStream(streamId) {
    const result = await db.query(`
      SELECT * FROM event_store 
      WHERE stream_id = $1 
      ORDER BY version ASC
    `, [streamId]);
    
    return result.rows;
  }
  
  async replay(streamId, handlers) {
    const events = await this.readStream(streamId);
    
    for (const event of events) {
      const handler = handlers[event.event_type];
      if (handler) {
        await handler(event.data);
      }
    }
  }
}
```

### 2. CQRS with Events

```javascript
// Command side - produces events
class OrderCommand {
  async createOrder(data) {
    const orderId = uuid();
    
    await eventStore.append(`order-${orderId}`, {
      type: 'ORDER_CREATED',
      data: data
    });
    
    return orderId;
  }
}

// Query side - consumes events to build read models
class OrderQuery {
  constructor() {
    eventBus.subscribe('ORDER_CREATED', this.handleOrderCreated);
    eventBus.subscribe('ORDER_SHIPPED', this.handleOrderShipped);
  }
  
  async handleOrderCreated(event) {
    await db.query(`
      INSERT INTO order_read_model 
      (id, customer_id, total, status, created_at)
      VALUES ($1, $2, $3, $4, $5)
    `, [event.orderId, event.customerId, event.total, 'PENDING', event.timestamp]);
  }
  
  async handleOrderShipped(event) {
    await db.query(`
      UPDATE order_read_model 
      SET status = 'SHIPPED', shipped_at = $2
      WHERE id = $1
    `, [event.orderId, event.timestamp]);
  }
}
```

### 3. Saga Pattern with Events

```javascript
class OrderSaga {
  constructor(eventBus) {
    this.sagas = new Map();
    
    eventBus.subscribe('ORDER_CREATED', this.startSaga.bind(this));
    eventBus.subscribe('PAYMENT_PROCESSED', this.handlePayment.bind(this));
    eventBus.subscribe('INVENTORY_RESERVED', this.handleInventory.bind(this));
    eventBus.subscribe('SHIPMENT_CREATED', this.completeSaga.bind(this));
    eventBus.subscribe('PAYMENT_FAILED', this.compensate.bind(this));
  }
  
  async startSaga(event) {
    const saga = {
      id: event.orderId,
      state: 'STARTED',
      data: event.data,
      steps: ['PAYMENT', 'INVENTORY', 'SHIPMENT']
    };
    
    this.sagas.set(event.orderId, saga);
    
    // Start first step
    await eventBus.publish('PROCESS_PAYMENT', {
      orderId: event.orderId,
      amount: event.data.total
    });
  }
  
  async handlePayment(event) {
    const saga = this.sagas.get(event.orderId);
    saga.state = 'PAYMENT_COMPLETED';
    
    await eventBus.publish('RESERVE_INVENTORY', {
      orderId: event.orderId,
      items: saga.data.items
    });
  }
  
  async compensate(event) {
    const saga = this.sagas.get(event.orderId);
    
    // Compensate in reverse order
    if (saga.state === 'PAYMENT_COMPLETED') {
      await eventBus.publish('REFUND_PAYMENT', {
        orderId: event.orderId
      });
    }
    
    saga.state = 'COMPENSATED';
  }
}
```

### 4. Dead Letter Queue for Failed Events

```javascript
class DeadLetterQueue {
  constructor() {
    this.failedEvents = [];
  }
  
  async handleFailure(event, error) {
    console.error(`Event ${event.type}