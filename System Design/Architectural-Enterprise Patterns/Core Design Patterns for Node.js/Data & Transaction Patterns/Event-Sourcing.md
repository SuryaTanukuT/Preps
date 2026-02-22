# Event Sourcing

## Definition
**Event Sourcing** is an architectural pattern where state changes are stored as a sequence of events, rather than storing just the current state. The current state is derived by replaying these events. Every change to the application state is captured in an event object, and these events are stored in the order they occurred.

Instead of "UPDATE users SET balance = 500 WHERE id = 1", you store "AccountDeposited" and "AccountWithdrawn" events, and calculate the balance by summing them.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │           COMMANDS                  │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼───────────────────┐
                    │         EVENT STORE                 │
                    │  [Event1, Event2, Event3, ...]      │
                    └───┬──────────────────────┬──────────┘
                        │                      │
            ┌───────────▼───────────┐  ┌───────▼───────────┐
            │     PROJECTIONS       │  │    QUERIES        │
            │  (Current State)      │  │  (Read Models)    │
            └───────────────────────┘  └───────────────────┘
```

**Traditional vs Event Sourcing:**

| Traditional | Event Sourcing |
|------------|----------------|
| `users` table with current balance | `events` table with all transactions |
| `UPDATE users SET balance = 500` | `INSERT INTO events (type, data) VALUES ('DEPOSIT', {...})` |
| Lost history | Full audit trail |
| Current state only | Can replay to any point |

---

## Good vs Bad Example

### Good Example
**Bank Account with Event Sourcing:**
```javascript
// Events are stored, not current state
const events = [
  { type: 'ACCOUNT_OPENED', data: { accountId: '123', owner: 'John' }, timestamp: '2024-01-01' },
  { type: 'DEPOSIT', data: { accountId: '123', amount: 1000 }, timestamp: '2024-01-02' },
  { type: 'WITHDRAWAL', data: { accountId: '123', amount: 200 }, timestamp: '2024-01-03' },
  { type: 'DEPOSIT', data: { accountId: '123', amount: 500 }, timestamp: '2024-01-04' }
];

// Current state calculated from events
function getBalance(accountId, events) {
  return events
    .filter(e => e.data.accountId === accountId)
    .reduce((balance, event) => {
      if (event.type === 'DEPOSIT') return balance + event.data.amount;
      if (event.type === 'WITHDRAWAL') return balance - event.data.amount;
      return balance;
    }, 0);
}

// Balance = 1000 - 200 + 500 = 1300
```

### Bad Example
**Traditional Update:**
```javascript
// Only current state stored
await db.query(`
  UPDATE accounts 
  SET balance = balance + 500 
  WHERE id = 123
`);

// Can't answer:
// - What was the balance yesterday?
// - Who made this change?
// - Why did we add 500?
// - Can we undo this change?
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Account Ledger

**Event Store Implementation:**
```javascript
// Event Store Schema
const eventStoreSchema = `
  CREATE TABLE event_store (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB NOT NULL,
    version INT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    UNIQUE(aggregate_id, version)
  );

  CREATE INDEX idx_event_store_aggregate 
    ON event_store(aggregate_id, version);
  
  CREATE INDEX idx_event_store_time 
    ON event_store(created_at);
`;

// Banking Events
const bankEvents = {
  ACCOUNT_OPENED: {
    accountId: 'uuid',
    ownerName: 'string',
    accountType: 'CHECKING|SAVINGS',
    initialDeposit: 'number',
    openedBy: 'employeeId'
  },
  
  DEPOSIT_MADE: {
    accountId: 'uuid',
    amount: 'number',
    method: 'CASH|CHECK|TRANSFER',
    checkNumber: 'string?',
    depositedBy: 'userId',
    location: 'branchId?'
  },
  
  WITHDRAWAL_MADE: {
    accountId: 'uuid',
    amount: 'number',
    method: 'ATM|TELLER|ONLINE',
    atmLocation: 'string?',
    tellerId: 'string?',
    remainingBalance: 'number' // Snapshot for quick access
  },
  
  FUNDS_TRANSFERRED: {
    fromAccount: 'uuid',
    toAccount: 'uuid',
    amount: 'number',
    reference: 'string',
    initiatedBy: 'userId'
  },
  
  ACCOUNT_FROZEN: {
    accountId: 'uuid',
    reason: 'string',
    frozenBy: 'employeeId',
    legalReference: 'string?'
  },
  
  MONTHLY_FEE_CHARGED: {
    accountId: 'uuid',
    amount: 'number',
    feeType: 'MAINTENANCE|OVERDRAFT|WIRE',
    statementMonth: 'date'
  }
};

// Command Handler
class BankAccount {
  constructor(events = []) {
    this.events = events;
    this.applyEvents(events);
  }
  
  // Apply events to build state
  applyEvents(events) {
    for (const event of events) {
      switch(event.type) {
        case 'ACCOUNT_OPENED':
          this.id = event.data.accountId;
          this.owner = event.data.ownerName;
          this.type = event.data.accountType;
          this.balance = event.data.initialDeposit || 0;
          this.status = 'ACTIVE';
          break;
          
        case 'DEPOSIT_MADE':
          this.balance += event.data.amount;
          this.lastTransaction = event.created_at;
          break;
          
        case 'WITHDRAWAL_MADE':
          this.balance -= event.data.amount;
          this.lastTransaction = event.created_at;
          break;
          
        case 'ACCOUNT_FROZEN':
          this.status = 'FROZEN';
          this.frozenReason = event.data.reason;
          break;
      }
    }
  }
  
  // Commands that produce events
  deposit(amount, method, userId) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Account not active');
    }
    
    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }
    
    // For large deposits, flag for compliance
    if (amount > 10000) {
      this.events.push({
        type: 'LARGE_DEPOSIT_FLAGGED',
        data: { amount, reviewer: 'COMPLIANCE_AUTO' }
      });
    }
    
    // Create deposit event
    return {
      type: 'DEPOSIT_MADE',
      data: {
        accountId: this.id,
        amount,
        method,
        depositedBy: userId,
        remainingBalance: this.balance + amount
      }
    };
  }
  
  withdraw(amount, method, userId) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Account not active');
    }
    
    if (this.balance - amount < -this.overdraftLimit) {
      throw new Error('Insufficient funds');
    }
    
    return {
      type: 'WITHDRAWAL_MADE',
      data: {
        accountId: this.id,
        amount,
        method,
        withdrawnBy: userId,
        remainingBalance: this.balance - amount
      }
    };
  }
}

// Event Store Repository
class EventStore {
  async saveEvents(aggregateId, events, expectedVersion) {
    const client = await pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Check concurrency
      const currentVersion = await client.query(
        'SELECT MAX(version) as version FROM event_store WHERE aggregate_id = $1',
        [aggregateId]
      );
      
      if (currentVersion.rows[0].version !== expectedVersion) {
        throw new ConcurrencyError('Aggregate was modified');
      }
      
      // Save events with version numbers
      for (let i = 0; i < events.length; i++) {
        await client.query(
          `INSERT INTO event_store 
           (aggregate_id, aggregate_type, event_type, event_data, metadata, version)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [
            aggregateId,
            'BANK_ACCOUNT',
            events[i].type,
            events[i].data,
            { userId: events[i].userId, ip: events[i].ip },
            expectedVersion + i + 1
          ]
        );
      }
      
      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }
  
  async getEvents(aggregateId, fromVersion = 0) {
    const result = await pool.query(
      `SELECT * FROM event_store 
       WHERE aggregate_id = $1 AND version > $2
       ORDER BY version ASC`,
      [aggregateId, fromVersion]
    );
    
    return result.rows;
  }
  
  async getEventsByType(eventType, fromDate, toDate) {
    // For compliance and reporting
    return await pool.query(
      `SELECT * FROM event_store 
       WHERE event_type = $1 
         AND created_at BETWEEN $2 AND $3
       ORDER BY created_at DESC`,
      [eventType, fromDate, toDate]
    );
  }
}
```

**Benefits for BFSI:**
- **Complete Audit Trail:** Every transaction recorded immutably
- **Compliance:** Can prove all changes for regulators
- **Fraud Detection:** Pattern analysis on event stream
- **Point-in-time Queries:** Balance as of any date
- **Dispute Resolution:** Full history for investigations

### 2. Real Estate - Property History

**Property Events:**
```javascript
const propertyEvents = {
  PROPERTY_LISTED: {
    propertyId: 'uuid',
    address: 'string',
    price: 'number',
    bedrooms: 'number',
    bathrooms: 'number',
    sqft: 'number',
    listedBy: 'agentId',
    mlsNumber: 'string'
  },
  
  PRICE_CHANGED: {
    propertyId: 'uuid',
    oldPrice: 'number',
    newPrice: 'number',
    reason: 'MARKET|OWNER_REQUEST|EXPIRING',
    changedBy: 'agentId',
    effectiveDate: 'date'
  },
  
  OFFER_RECEIVED: {
    propertyId: 'uuid',
    offerId: 'uuid',
    amount: 'number',
    buyerId: 'uuid',
    terms: 'string',
    contingencies: ['string'],
    expiresAt: 'date'
  },
  
  OFFER_ACCEPTED: {
    propertyId: 'uuid',
    offerId: 'uuid',
    acceptedBy: 'sellerId',
    acceptanceDate: 'date'
  },
  
  UNDER_CONTRACT: {
    propertyId: 'uuid',
    contractDate: 'date',
    expectedClosing: 'date'
  },
  
  SOLD: {
    propertyId: 'uuid',
    finalPrice: 'number',
    closingDate: 'date',
    buyerId: 'uuid',
    sellerId: 'uuid'
  },
  
  OPEN_HOUSE_SCHEDULED: {
    propertyId: 'uuid',
    date: 'date',
    startTime: 'time',
    endTime: 'time',
    agentId: 'uuid'
  }
};

// Property Aggregate
class Property {
  constructor(events) {
    this.events = events;
    this.apply(events);
  }
  
  apply(events) {
    for (const event of events) {
      switch(event.type) {
        case 'PROPERTY_LISTED':
          this.id = event.data.propertyId;
          this.address = event.data.address;
          this.price = event.data.price;
          this.status = 'ACTIVE';
          this.listedDate = event.created_at;
          break;
          
        case 'PRICE_CHANGED':
          this.priceHistory = this.priceHistory || [];
          this.priceHistory.push({
            oldPrice: this.price,
            newPrice: event.data.newPrice,
            date: event.created_at,
            reason: event.data.reason
          });
          this.price = event.data.newPrice;
          break;
          
        case 'OFFER_ACCEPTED':
          this.status = 'UNDER_CONTRACT';
          this.acceptedOfferId = event.data.offerId;
          break;
          
        case 'SOLD':
          this.status = 'SOLD';
          this.soldPrice = event.data.finalPrice;
          this.closingDate = event.data.closingDate;
          break;
      }
    }
  }
  
  // Commands
  changePrice(newPrice, reason, agentId) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Can only change price on active listings');
    }
    
    return {
      type: 'PRICE_CHANGED',
      data: {
        propertyId: this.id,
        oldPrice: this.price,
        newPrice,
        reason,
        changedBy: agentId
      }
    };
  }
  
  acceptOffer(offerId, sellerId) {
    if (this.status !== 'ACTIVE') {
      throw new Error('Property not available');
    }
    
    return {
      type: 'OFFER_ACCEPTED',
      data: {
        propertyId: this.id,
        offerId,
        acceptedBy: sellerId,
        acceptanceDate: new Date()
      }
    };
  }
}

// Market Analysis from Events
class MarketAnalytics {
  async getPriceHistory(propertyId) {
    const events = await eventStore.getEvents(propertyId);
    
    return events
      .filter(e => e.type === 'PRICE_CHANGED')
      .map(e => ({
        date: e.created_at,
        price: e.data.newPrice,
        reason: e.data.reason
      }));
  }
  
  async getNeighborhoodTrends(zipCode, months = 12) {
    const since = new Date();
    since.setMonth(since.getMonth() - months);
    
    const sales = await eventStore.getEventsByType('SOLD', since, new Date());
    
    const neighborhoodSales = sales.filter(s => 
      s.data.zipCode === zipCode
    );
    
    return {
      averagePrice: this.average(neighborhoodSales.map(s => s.data.finalPrice)),
      medianPrice: this.median(neighborhoodSales.map(s => s.data.finalPrice)),
      daysOnMarket: this.average(neighborhoodSales.map(s => s.data.daysOnMarket)),
      totalSales: neighborhoodSales.length,
      pricePerSqft: this.average(neighborhoodSales.map(s => s.data.pricePerSqft)),
      monthlyBreakdown: this.groupByMonth(neighborhoodSales)
    };
  }
}
```

**Benefits for Real Estate:**
- **Price History:** Track all changes with reasons
- **Market Trends:** Analyze from actual events
- **Offer Tracking:** Complete negotiation history
- **MLS Compliance:** Record every listing change

### 3. E-commerce - Order Management

**Order Events:**
```javascript
const orderEvents = {
  ORDER_CREATED: {
    orderId: 'uuid',
    customerId: 'uuid',
    items: [{ productId: 'uuid', quantity: 'number', price: 'number' }],
    shippingAddress: 'object',
    paymentMethod: 'string',
    total: 'number'
  },
  
  PAYMENT_AUTHORIZED: {
    orderId: 'uuid',
    amount: 'number',
    authorizationCode: 'string',
    paymentMethod: 'string',
    timestamp: 'date'
  },
  
  PAYMENT_CAPTURED: {
    orderId: 'uuid',
    amount: 'number',
    captureId: 'string'
  },
  
  INVENTORY_RESERVED: {
    orderId: 'uuid',
    items: [{ productId: 'uuid', quantity: 'number' }],
    warehouseId: 'string'
  },
  
  ORDER_SHIPPED: {
    orderId: 'uuid',
    trackingNumber: 'string',
    carrier: 'string',
    shippedAt: 'date',
    items: [{ productId: 'uuid', quantity: 'number' }]
  },
  
  ITEM_RETURNED: {
    orderId: 'uuid',
    itemId: 'uuid',
    reason: 'string',
    refundAmount: 'number',
    returnedAt: 'date'
  },
  
  ORDER_CANCELLED: {
    orderId: 'uuid',
    reason: 'string',
    cancelledBy: 'USER|ADMIN',
    refundIssued: 'boolean'
  }
};

// Order Aggregate
class Order {
  constructor(events) {
    this.state = {
      status: 'PENDING',
      items: [],
      payments: [],
      shipments: [],
      returns: []
    };
    
    this.apply(events);
  }
  
  apply(events) {
    for (const event of events) {
      switch(event.type) {
        case 'ORDER_CREATED':
          this.state.id = event.data.orderId;
          this.state.customerId = event.data.customerId;
          this.state.items = event.data.items;
          this.state.shippingAddress = event.data.shippingAddress;
          this.state.total = event.data.total;
          this.state.status = 'PENDING_PAYMENT';
          break;
          
        case 'PAYMENT_AUTHORIZED':
          this.state.payments.push({
            type: 'AUTHORIZATION',
            amount: event.data.amount,
            code: event.data.authorizationCode,
            timestamp: event.data.timestamp
          });
          this.state.status = 'PAYMENT_AUTHORIZED';
          break;
          
        case 'PAYMENT_CAPTURED':
          this.state.payments.push({
            type: 'CAPTURE',
            amount: event.data.amount,
            captureId: event.data.captureId
          });
          this.state.status = 'PAID';
          break;
          
        case 'ORDER_SHIPPED':
          this.state.shipments.push({
            trackingNumber: event.data.trackingNumber,
            carrier: event.data.carrier,
            shippedAt: event.data.shippedAt,
            items: event.data.items
          });
          this.state.status = 'SHIPPED';
          break;
          
        case 'ITEM_RETURNED':
          this.state.returns.push({
            itemId: event.data.itemId,
            reason: event.data.reason,
            refundAmount: event.data.refundAmount,
            returnedAt: event.data.returnedAt
          });
          break;
          
        case 'ORDER_CANCELLED':
          this.state.status = 'CANCELLED';
          this.state.cancellationReason = event.data.reason;
          break;
      }
    }
  }
  
  // Commands
  cancel(reason, userId) {
    if (['SHIPPED', 'DELIVERED'].includes(this.state.status)) {
      throw new Error('Cannot cancel shipped order');
    }
    
    return {
      type: 'ORDER_CANCELLED',
      data: {
        orderId: this.state.id,
        reason,
        cancelledBy: userId,
        refundIssued: this.state.status === 'PAID'
      }
    };
  }
  
  returnItem(itemId, reason, condition) {
    const item = this.state.items.find(i => i.id === itemId);
    
    if (!item) {
      throw new Error('Item not found');
    }
    
    // Check return policy (30 days)
    const orderDate = this.state.createdAt;
    const daysSince = (new Date() - orderDate) / (1000 * 60 * 60 * 24);
    
    if (daysSince > 30) {
      throw new Error('Return period expired');
    }
    
    return {
      type: 'ITEM_RETURNED',
      data: {
        orderId: this.state.id,
        itemId,
        reason,
        condition,
        refundAmount: item.price,
        returnedAt: new Date()
      }
    };
  }
}

// Read Models from Events
class OrderReadModel {
  constructor(eventStore) {
    this.eventStore = eventStore;
    
    // Subscribe to events to build read models
    this.eventStore.subscribe('ORDER_CREATED', this.handleOrderCreated);
    this.eventStore.subscribe('ORDER_SHIPPED', this.handleOrderShipped);
  }
  
  async handleOrderCreated(event) {
    // Update customer order history view
    await db.query(`
      INSERT INTO customer_orders_view (
        customer_id, order_id, total, items_count, status, created_at
      ) VALUES ($1, $2, $3, $4, $5, $6)
    `, [
      event.data.customerId,
      event.data.orderId,
      event.data.total,
      event.data.items.length,
      'PENDING',
      event.created_at
    ]);
    
    // Update product sales view
    for (const item of event.data.items) {
      await db.query(`
        UPDATE product_sales_view 
        SET order_count = order_count + 1,
            units_sold = units_sold + $1,
            revenue = revenue + $2
        WHERE product_id = $3
      `, [item.quantity, item.price * item.quantity, item.productId]);
    }
  }
  
  async handleOrderShipped(event) {
    await db.query(`
      UPDATE customer_orders_view 
      SET status = 'SHIPPED',
          tracking_number = $1,
          carrier = $2,
          shipped_at = $3
      WHERE order_id = $4
    `, [
      event.data.trackingNumber,
      event.data.carrier,
      event.data.shippedAt,
      event.data.orderId
    ]);
  }
  
  async getCustomerOrders(customerId) {
    return await db.query(
      'SELECT * FROM customer_orders_view WHERE customer_id = $1 ORDER BY created_at DESC',
      [customerId]
    );
  }
  
  async getProductSales(productId, days = 30) {
    return await db.query(`
      SELECT 
        DATE(created_at) as date,
        SUM(units_sold) as units,
        SUM(revenue) as revenue
      FROM product_sales_view
      WHERE product_id = $1
        AND created_at >= NOW() - INTERVAL '1 day' * $2
      GROUP BY DATE(created_at)
      ORDER BY date DESC
    `, [productId, days]);
  }
}
```

**Benefits for E-commerce:**
- **Order History:** Complete lifecycle tracking
- **Inventory Projections:** Know when to reorder
- **Returns Analysis:** Track reasons and patterns
- **Fraud Detection:** Spot unusual order patterns
- **Customer 360:** Complete interaction history

---

## Key Patterns in Event Sourcing

### 1. Event Store Implementation

```javascript
class EventStore {
  constructor(database) {
    this.db = database;
    this.subscribers = new Map();
  }
  
  async append(aggregateId, events, expectedVersion) {
    const client = await this.db.beginTransaction();
    
    try {
      // Check current version
      const current = await client.query(
        'SELECT MAX(version) as version FROM events WHERE aggregate_id = $1',
        [aggregateId]
      );
      
      if (current.rows[0].version !== expectedVersion) {
        throw new ConcurrencyError('Version mismatch');
      }
      
      // Save events
      for (let i = 0; i < events.length; i++) {
        const event = events[i];
        const version = expectedVersion + i + 1;
        
        await client.query(
          `INSERT INTO events 
           (id, aggregate_id, event_type, data, metadata, version, created_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7)`,
          [
            uuid(),
            aggregateId,
            event.type,
            event.data,
            event.metadata,
            version,
            new Date()
          ]
        );
        
        // Notify subscribers
        await this.notifySubscribers(event.type, event);
      }
      
      await client.commit();
    } catch (error) {
      await client.rollback();
      throw error;
    }
  }
  
  async readStream(aggregateId, fromVersion = 0) {
    const result = await this.db.query(
      `SELECT * FROM events 
       WHERE aggregate_id = $1 AND version > $2
       ORDER BY version ASC`,
      [aggregateId, fromVersion]
    );
    
    return result.rows;
  }
  
  async readAll(aggregateType, fromDate, toDate) {
    return await this.db.query(
      `SELECT * FROM events 
       WHERE aggregate_type = $1 
         AND created_at BETWEEN $2 AND $3
       ORDER BY created_at ASC`,
      [aggregateType, fromDate, toDate]
    );
  }
  
  subscribe(eventType, handler) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, []);
    }
    this.subscribers.get(eventType).push(handler);
  }
  
  async notifySubscribers(eventType, event) {
    const handlers = this.subscribers.get(eventType) || [];
    for (const handler of handlers) {
      try {
        await handler(event);
      } catch (error) {
        console.error('Subscriber error:', error);
        // Continue processing other subscribers
      }
    }
  }
}
```

### 2. Snapshot Pattern

```javascript
class SnapshotRepository {
  constructor(eventStore, snapshotFrequency = 100) {
    this.eventStore = eventStore;
    this.snapshotFrequency = snapshotFrequency;
  }
  
  async load(aggregateId) {
    // Try to get latest snapshot
    const snapshot = await this.getLatestSnapshot(aggregateId);
    
    let fromVersion = 0;
    let state = null;
    
    if (snapshot) {
      fromVersion = snapshot.version;
      state = snapshot.state;
    }
    
    // Get events since snapshot
    const events = await this.eventStore.readStream(aggregateId, fromVersion);
    
    // Create aggregate and apply events
    const aggregate = this.createAggregate(state);
    aggregate.applyEvents(events);
    
    return aggregate;
  }
  
  async save(aggregate) {
    const events = aggregate.getUncommittedEvents();
    
    if (events.length === 0) return;
    
    // Save events
    await this.eventStore.append(
      aggregate.id,
      events,
      aggregate.version
    );
    
    // Check if snapshot needed
    const newVersion = aggregate.version + events.length;
    if (Math.floor(newVersion / this.snapshotFrequency) > 
        Math.floor(aggregate.version / this.snapshotFrequency)) {
      await this.createSnapshot(aggregate.id, aggregate.getState(), newVersion);
    }
    
    aggregate.markEventsAsCommitted();
  }
  
  async createSnapshot(aggregateId, state, version) {
    await this.db.query(
      `INSERT INTO snapshots (aggregate_id, state, version, created_at)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (aggregate_id) DO UPDATE
       SET state = $2, version = $3, created_at = $4`,
      [aggregateId, state, version, new Date()]
    );
  }
  
  async getLatestSnapshot(aggregateId) {
    const result = await this.db.query(
      'SELECT * FROM snapshots WHERE aggregate_id = $1',
      [aggregateId]
    );
    
    return result.rows[0];
  }
}
```

### 3. Event Versioning

```javascript
// Handle schema evolution
class EventUpcaster {
  constructor(migrations) {
    this.migrations = migrations;
  }
  
  upcast(events) {
    return events.map(event => {
      const version = event.event_version || 1;
      const migrations = this.migrations[event.event_type] || [];
      
      // Apply migrations in order
      for (const migration of migrations.slice(version - 1)) {
        event = migration(event);
      }
      
      return event;
    });
  }
}

// Migration examples
const migrations = {
  'ORDER_CREATED': [
    // Version 1 -> 2: Add currency field
    (event) => ({
      ...event,
      data: {
        ...event.data,
        currency: 'USD' // Default for old events
      }
    }),
    // Version 2 -> 3: Split address into components
    (event) => ({
      ...event,
      data: {
        ...event.data,
        shippingAddress: {
          street: event.data.address,
          city: event.data.city,
          zip: event.data.zip,
          country: 'USA'
        }
      }
    })
  ]
};

// Usage
const upcaster = new EventUpcaster(migrations);
const upcastedEvents = upcaster.upcast(rawEvents);
```

### 4. Projections (Read Models)

```javascript
class Projection {
  constructor(eventStore, viewName) {
    this.eventStore = eventStore;
    this.viewName = viewName;
    this.handlers = {};
  }
  
  when(eventType, handler) {
    this.handlers[eventType] = handler;
  }
  
  async build() {
    // Get all events
    const events = await this.eventStore.readAll();
    
    // Start from scratch
    await this.reset();
    
    // Apply in order
    for (const event of events) {
      await this.handleEvent(event);
    }
  }
  
  async handleEvent(event) {
    const handler = this.handlers[event.event_type];
    if (handler) {
      await handler(event, this.viewName);
    }
  }
  
  async reset() {
    await this.db.query(`TRUNCATE TABLE ${this.viewName}`);
  }
}

// Example projection
const orderSummaryProjection = new Projection(eventStore, 'order_summary');

orderSummaryProjection.when('ORDER_CREATED', async (event, viewName) => {
  await db.query(`
    INSERT INTO ${viewName} 
    (order_id, customer_id, total, status, created_at)
    VALUES ($1, $2, $3, $4, $5)
  `, [
    event.data.orderId,
    event.data.customerId,
    event.data.total,
    'PENDING',
    event.created_at
  ]);
});

orderSummaryProjection.when('ORDER_SHIPPED', async (event, viewName) => {
  await db.query(`
    UPDATE ${viewName}
    SET status = 'SHIPPED', shipped_at = $2
    WHERE order_id = $1
  `, [event.data.orderId, event.created_at]);
});
```

---

## Event Sourcing + CQRS

```
                    ┌─────────────────────────────────────┐
                    │           COMMANDS                  │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────▼───────────────────┐
                    │         EVENT STORE                 │
                    │  (Source of Truth)                  │
                    └───┬──────────────────────┬──────────┘
                        │                      │
            ┌───────────▼───────────┐  ┌───────▼───────────┐
            │     PROJECTIONS       │  │     QUERIES       │
            │  (Build Read Models)  │  │  (Read from Views)│
            └───────────┬───────────┘  └───────┬───────────┘
                        │                      │
            ┌───────────▼───────────┐  ┌───────▼───────────┐
            │     READ DATABASE     │  │   READ DATABASE   │
            │   (Orders View)       │  │  (Products View)  │
            └───────────────────────┘  └───────────────────┘
```

**Integration Example:**
```javascript
class OrderService {
  constructor(eventStore, projections) {
    this.eventStore = eventStore;
    this.projections = projections;
  }
  
  async createOrder(command) {
    // 1. Load aggregate
    const order = await this.loadOrder(command.orderId);
    
    // 2. Execute command to create events
    const events = order.createOrder(command);
    
    // 3. Save events atomically
    await this.eventStore.append(order.id, events, order.version);
    
    // 4. Projections update asynchronously
    // (Event handlers update read models)
    
    return order.id;
  }
  
  async getOrder(orderId) {
    // Read from projection (CQRS query side)
    return await db.query(
      'SELECT * FROM order_details_view WHERE order_id = $1',
      [orderId]
    );
  }
  
  async getOrderHistory(orderId) {
    // Read from event store directly for audit
    const events = await this.eventStore.readStream(orderId);
    
    return events.map(e => ({
      type: e.event_type,
      data: e.data,
      timestamp: e.created_at
    }));
  }
}
```

---

## When to Use Event Sourcing

### ✅ DO Use When:
- **Audit Trail Required:** Financial, healthcare, compliance
- **Complex Business Rules:** Need to understand why state changed
- **Temporal Queries:** "What was the state at any point in time?"
- **Event-Driven Architecture:** Natural fit with pub/sub
- **Debugging/Replay:** Can reproduce issues by replaying events
- **Multiple Read Models:** Different views of same data

### ❌ DON'T Use When:
- **Simple CRUD:** Overkill for basic applications
- **Strong Consistency Required:** Eventual consistency only
- **Small Team:** Adds significant complexity
- **Volatile Schema:** Events are immutable, hard to change
- **Performance Critical:** Replaying events can be slow

---

## Event Sourcing vs Traditional

| Aspect | Traditional | Event Sourcing |
|--------|------------|----------------|
| **Storage** | Current state | Sequence of events |
| **History** | Lost | Complete |
| **Audit** | Separate audit log | Built-in |
| **Temporal Queries** | Difficult | Easy |
| **Performance** | Fast reads | Fast writes, slower reads |
| **Complexity** | Low | High |
| **Schema Changes** | Easy | Hard (event versioning) |
| **Debugging** | Hard | Easy (replay) |

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Event Schema Evolution** | Version events, upcasters |
| **Performance** | Snapshots, projections |
| **Event Store Size** | Archive old events |
| **Consistency** | Handle eventual consistency |
| **Debugging** | Correlation IDs, tracing |
| **Event Design** | Rich events with context |

---

## Summary Checklist

| Component | Implementation |
|-----------|---------------|
| **Event Store** | Append-only, immutable |
| **Aggregates** | Command handlers, produce events |
| **Projections** | Build read models |
| **Snapshots** | Performance optimization |
| **Versioning** | Upcasters for schema changes |
| **Subscribers** | Update projections, trigger side effects |
| **Monitoring** | Track event lag, failures |

**Bottom Line:** Event Sourcing provides complete auditability and temporal query capabilities at the cost of complexity. Best paired with CQRS and used in domains where understanding "why" is as important as "what."