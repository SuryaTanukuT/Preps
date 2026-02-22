# Transactional Outbox Pattern

## Definition
The **Transactional Outbox Pattern** ensures reliable message delivery in distributed systems by storing messages in a database table as part of the same local transaction that modifies application state. A separate process (publisher) reads from this outbox table and publishes messages to the message broker, guaranteeing that messages are eventually sent exactly once.

This pattern solves the dual-write problem: when you need to update a database AND send a message, but can't do both atomically.

---

## Core Concept
```
                    COMMAND SIDE
    ┌─────────────────────────────────────┐
    │  1. Start Transaction               │
    │                                     │
    │  2. Update Database                  │
    │     (Application State)              │
    │                                     │
    │  3. Insert into Outbox Table        │
    │     (Same Transaction)               │
    │                                     │
    │  4. Commit Transaction              │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │      OUTBOX TABLE (Database)        │
    │  ┌─────────────────────────────┐   │
    │  │ id | event | status | tries │   │
    │  │ 1  | OrderCreated | PENDING | 0 │
    │  │ 2  | PaymentDone  | PENDING | 0 │
    │  └─────────────────────────────┘   │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │      RELIABLE PUBLISHER             │
    │  (Polling or CDC)                   │
    │                                      │
    │  while true:                        │
    │    read PENDING events               │
    │    publish to broker                  │
    │    mark as SENT                       │
    └─────────────────┬───────────────────┘
                      │
                      ▼
    ┌─────────────────────────────────────┐
    │         MESSAGE BROKER              │
    │    (Kafka, RabbitMQ, SQS)           │
    └─────────────────────────────────────┘
```

---

## The Problem It Solves

### Dual-Write Problem (Bad Example)
```javascript
// BAD: No transactional outbox
app.post('/api/orders', async (req, res) => {
  // 1. Save to database
  await db.orders.insert(orderData);
  
  // 2. Send message to broker
  // WHAT IF THIS FAILS???
  await broker.publish('order.created', orderData);
  
  res.json({ success: true });
});
```
**Problems:**
- If broker is down, order saved but no event → inconsistency
- If broker succeeds but response fails, client retries → duplicate
- No atomicity between DB and message

### Transactional Outbox Solution (Good Example)
```javascript
// GOOD: Transactional outbox
app.post('/api/orders', async (req, res) => {
  const transaction = await db.beginTransaction();
  
  try {
    // 1. Save to database
    const order = await db.orders.insert(orderData, { transaction });
    
    // 2. Save to outbox (SAME TRANSACTION)
    await db.outbox.insert({
      eventId: uuid(),
      eventType: 'order.created',
      payload: JSON.stringify(order),
      status: 'PENDING',
      createdAt: new Date()
    }, { transaction });
    
    // 3. Commit atomically
    await transaction.commit();
    
    res.json({ success: true });
  } catch (error) {
    await transaction.rollback();
    throw error;
  }
});
```

---

## Industry Use Cases

### 1. BFSI (Banking) - Transaction Events

```javascript
// ============ COMMAND SIDE ============
class TransferService {
  async transferMoney(command) {
    const transaction = await db.beginTransaction();
    
    try {
      // 1. Update accounts (ACID)
      await db.query(
        'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
        [command.amount, command.fromAccount],
        { transaction }
      );
      
      await db.query(
        'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
        [command.amount, command.toAccount],
        { transaction }
      );
      
      // 2. Record transaction
      const transferRecord = await db.query(
        `INSERT INTO transfers 
         (from_account, to_account, amount, status, created_at)
         VALUES ($1, $2, $3, $4, $5) RETURNING id`,
        [command.fromAccount, command.toAccount, command.amount, 'COMPLETED', new Date()],
        { transaction }
      );
      
      // 3. Write to outbox (SAME TRANSACTION)
      await db.query(
        `INSERT INTO outbox 
         (event_id, event_type, aggregate_id, payload, status, created_at)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [
          uuid(),
          'transfer.completed',
          transferRecord.id,
          JSON.stringify({
            transferId: transferRecord.id,
            fromAccount: command.fromAccount,
            toAccount: command.toAccount,
            amount: command.amount,
            timestamp: new Date()
          }),
          'PENDING',
          new Date()
        ],
        { transaction }
      );
      
      // 4. Commit everything atomically
      await transaction.commit();
      
      return { transferId: transferRecord.id };
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}

// ============ RELIABLE PUBLISHER ============
class OutboxPublisher {
  constructor(broker, db, options = {}) {
    this.broker = broker;
    this.db = db;
    this.pollInterval = options.pollInterval || 1000;
    this.batchSize = options.batchSize || 100;
    this.running = false;
  }
  
  start() {
    this.running = true;
    this.poll();
  }
  
  stop() {
    this.running = false;
  }
  
  async poll() {
    while (this.running) {
      try {
        await this.publishPendingEvents();
      } catch (error) {
        console.error('Outbox publisher error:', error);
      }
      
      await this.sleep(this.pollInterval);
    }
  }
  
  async publishPendingEvents() {
    // Use SELECT ... FOR UPDATE SKIP LOCKED to prevent multiple publishers
    const events = await this.db.query(`
      SELECT * FROM outbox 
      WHERE status = 'PENDING' 
      AND attempts < 3
      ORDER BY created_at ASC
      LIMIT $1
      FOR UPDATE SKIP LOCKED
    `, [this.batchSize]);
    
    for (const event of events.rows) {
      await this.publishEvent(event);
    }
  }
  
  async publishEvent(event) {
    const transaction = await this.db.beginTransaction();
    
    try {
      // 1. Update status to PROCESSING
      await this.db.query(
        'UPDATE outbox SET status = $1, updated_at = $2 WHERE id = $3',
        ['PROCESSING', new Date(), event.id],
        { transaction }
      );
      
      // 2. Publish to broker
      await this.broker.publish(
        event.event_type,
        JSON.parse(event.payload),
        { 
          messageId: event.event_id,
          contentType: 'application/json'
        }
      );
      
      // 3. Mark as SENT
      await this.db.query(
        'UPDATE outbox SET status = $1, sent_at = $2, updated_at = $3 WHERE id = $4',
        ['SENT', new Date(), new Date(), event.id],
        { transaction }
      );
      
      await transaction.commit();
      
      console.log(`Published event ${event.event_id} (${event.event_type})`);
    } catch (error) {
      await transaction.rollback();
      
      // Increment attempt count
      await this.db.query(
        `UPDATE outbox 
         SET status = 'PENDING', 
             attempts = attempts + 1,
             last_error = $1,
             updated_at = $2
         WHERE id = $3`,
        [error.message, new Date(), event.id]
      );
      
      console.error(`Failed to publish event ${event.event_id}:`, error);
    }
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// ============ DEAD LETTER QUEUE HANDLER ============
class DeadLetterHandler {
  async processFailedEvents() {
    const failedEvents = await db.query(`
      SELECT * FROM outbox 
      WHERE attempts >= 3 AND status != 'DLQ'
    `);
    
    for (const event of failedEvents.rows) {
      // Move to DLQ table
      await db.query(
        `INSERT INTO dead_letter_queue 
         (event_id, event_type, payload, attempts, last_error, created_at)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [
          event.event_id,
          event.event_type,
          event.payload,
          event.attempts,
          event.last_error,
          new Date()
        ]
      );
      
      // Update original
      await db.query(
        "UPDATE outbox SET status = 'DLQ' WHERE id = $1",
        [event.id]
      );
    }
  }
  
  async retryDeadLetter(dlqId) {
    const event = await db.query(
      'SELECT * FROM dead_letter_queue WHERE id = $1',
      [dlqId]
    );
    
    // Re-insert into outbox
    await db.query(
      `INSERT INTO outbox 
       (event_id, event_type, aggregate_id, payload, status, attempts, created_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7)`,
      [
        event.event_id,
        event.event_type,
        event.aggregate_id,
        event.payload,
        'PENDING',
        0,
        new Date()
      ]
    );
    
    // Delete from DLQ
    await db.query('DELETE FROM dead_letter_queue WHERE id = $1', [dlqId]);
  }
}
```

### 2. E-commerce - Order Processing

```javascript
// ============ ORDER SERVICE WITH OUTBOX ============
class OrderService {
  async createOrder(orderData) {
    const transaction = await db.beginTransaction();
    
    try {
      // 1. Insert order
      const order = await db.query(
        `INSERT INTO orders 
         (user_id, total, status, created_at)
         VALUES ($1, $2, $3, $4) RETURNING id`,
        [orderData.userId, orderData.total, 'PENDING', new Date()],
        { transaction }
      );
      
      // 2. Insert order items
      for (const item of orderData.items) {
        await db.query(
          `INSERT INTO order_items 
           (order_id, product_id, quantity, price, created_at)
           VALUES ($1, $2, $3, $4, $5)`,
          [order.id, item.productId, item.quantity, item.price, new Date()],
          { transaction }
        );
      }
      
      // 3. Write multiple events to outbox
      const events = [
        {
          event_id: uuid(),
          event_type: 'order.created',
          aggregate_id: order.id,
          payload: JSON.stringify({
            orderId: order.id,
            userId: orderData.userId,
            total: orderData.total,
            items: orderData.items,
            timestamp: new Date()
          })
        },
        {
          event_id: uuid(),
          event_type: 'inventory.reserve.requested',
          aggregate_id: order.id,
          payload: JSON.stringify({
            orderId: order.id,
            items: orderData.items.map(i => ({
              productId: i.productId,
              quantity: i.quantity
            }))
          })
        },
        {
          event_id: uuid(),
          event_type: 'analytics.order.started',
          aggregate_id: order.id,
          payload: JSON.stringify({
            orderId: order.id,
            userId: orderData.userId,
            timestamp: new Date()
          })
        }
      ];
      
      for (const event of events) {
        await db.query(
          `INSERT INTO outbox 
           (event_id, event_type, aggregate_id, payload, status, created_at)
           VALUES ($1, $2, $3, $4, $5, $6)`,
          [event.event_id, event.event_type, event.aggregate_id, event.payload, 'PENDING', new Date()],
          { transaction }
        );
      }
      
      await transaction.commit();
      
      return order.id;
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
  
  async updateOrderStatus(orderId, status) {
    const transaction = await db.beginTransaction();
    
    try {
      // 1. Update order
      await db.query(
        'UPDATE orders SET status = $1, updated_at = $2 WHERE id = $3',
        [status, new Date(), orderId],
        { transaction }
      );
      
      // 2. Write status change event
      await db.query(
        `INSERT INTO outbox 
         (event_id, event_type, aggregate_id, payload, status, created_at)
         VALUES ($1, $2, $3, $4, $5, $6)`,
        [
          uuid(),
          'order.status.changed',
          orderId,
          JSON.stringify({
            orderId,
            oldStatus: 'PENDING', // In real app, get current status
            newStatus: status,
            timestamp: new Date()
          }),
          'PENDING',
          new Date()
        ],
        { transaction }
      );
      
      await transaction.commit();
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}

// ============ MULTIPLE CONSUMERS ============
// Each consumer processes different event types
class InventoryConsumer {
  constructor(outboxPublisher) {
    // Subscribe to inventory events
    outboxPublisher.registerHandler('inventory.reserve.requested', this.handleReserveRequest.bind(this));
  }
  
  async handleReserveRequest(event) {
    const { orderId, items } = JSON.parse(event.payload);
    
    for (const item of items) {
      const available = await this.checkStock(item.productId, item.quantity);
      
      if (!available) {
        // Publish failure event (will go through outbox)
        await this.publishInventoryFailed(orderId, item.productId);
        return;
      }
      
      await this.reserveStock(item.productId, item.quantity, orderId);
    }
    
    // Publish success
    await this.publishInventoryReserved(orderId);
  }
}

// ============ EVENT HANDLER WITH IDEMPOTENCY ============
class IdempotentEventHandler {
  constructor(db) {
    this.db = db;
    this.processedEvents = new Set();
  }
  
  async handle(event, handler) {
    // Check if already processed (using database)
    const processed = await this.db.query(
      'SELECT 1 FROM processed_events WHERE event_id = $1',
      [event.event_id]
    );
    
    if (processed.rows.length > 0) {
      console.log(`Event ${event.event_id} already processed, skipping`);
      return;
    }
    
    const transaction = await this.db.beginTransaction();
    
    try {
      // Process the event
      await handler(JSON.parse(event.payload));
      
      // Mark as processed (same transaction)
      await this.db.query(
        'INSERT INTO processed_events (event_id, processed_at) VALUES ($1, $2)',
        [event.event_id, new Date()],
        { transaction }
      );
      
      await transaction.commit();
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}
```

### 3. Real Estate - Property Listing Sync

```javascript
// ============ PROPERTY SERVICE ============
class PropertyService {
  async updatePropertyPrice(propertyId, newPrice, agentId) {
    const transaction = await db.beginTransaction();
    
    try {
      // 1. Get current price
      const property = await db.query(
        'SELECT price FROM properties WHERE id = $1',
        [propertyId],
        { transaction }
      );
      
      // 2. Update price
      await db.query(
        'UPDATE properties SET price = $1, updated_at = $2 WHERE id = $3',
        [newPrice, new Date(), propertyId],
        { transaction }
      );
      
      // 3. Record price history
      await db.query(
        `INSERT INTO price_history 
         (property_id, old_price, new_price, changed_by, created_at)
         VALUES ($1, $2, $3, $4, $5)`,
        [propertyId, property.rows[0].price, newPrice, agentId, new Date()],
        { transaction }
      );
      
      // 4. Write to outbox for multiple systems
      const events = [
        {
          event_id: uuid(),
          event_type: 'property.price.updated',
          aggregate_id: propertyId,
          target: 'search_engine',
          payload: JSON.stringify({
            propertyId,
            newPrice,
            updateType: 'PRICE_CHANGE'
          })
        },
        {
          event_id: uuid(),
          event_type: 'property.price.updated',
          aggregate_id: propertyId,
          target: 'mls_sync',
          payload: JSON.stringify({
            propertyId,
            newPrice,
            effectiveDate: new Date()
          })
        },
        {
          event_id: uuid(),
          event_type: 'property.price.updated',
          aggregate_id: propertyId,
          target: 'buyer_alerts',
          payload: JSON.stringify({
            propertyId,
            newPrice,
            oldPrice: property.rows[0].price,
            changePercentage: ((newPrice - property.rows[0].price) / property.rows[0].price * 100)
          })
        }
      ];
      
      for (const event of events) {
        await db.query(
          `INSERT INTO outbox 
           (event_id, event_type, aggregate_id, target, payload, status, created_at)
           VALUES ($1, $2, $3, $4, $5, $6, $7)`,
          [event.event_id, event.event_type, event.aggregate_id, event.target, event.payload, 'PENDING', new Date()],
          { transaction }
        );
      }
      
      await transaction.commit();
    } catch (error) {
      await transaction.rollback();
      throw error;
    }
  }
}

// ============ TARGET-SPECIFIC PUBLISHERS ============
class TargetedOutboxPublisher {
  constructor(target) {
    this.target = target;
  }
  
  async publishPendingEvents() {
    const events = await db.query(`
      SELECT * FROM outbox 
      WHERE target = $1 
        AND status = 'PENDING'
        AND attempts < 3
      ORDER BY created_at ASC
      LIMIT 100
      FOR UPDATE SKIP LOCKED
    `, [this.target]);
    
    for (const event of events.rows) {
      await this.publishEvent(event);
    }
  }
  
  async publishEvent(event) {
    try {
      // Different publishing logic per target
      switch(this.target) {
        case 'search_engine':
          await this.updateSearchEngine(event);
          break;
        case 'mls_sync':
          await this.syncWithMLS(event);
          break;
        case 'buyer_alerts':
          await this.processBuyerAlerts(event);
          break;
      }
      
      // Mark as sent
      await db.query(
        'UPDATE outbox SET status = $1, sent_at = $2 WHERE id = $3',
        ['SENT', new Date(), event.id]
      );
    } catch (error) {
      // Increment attempts
      await db.query(
        `UPDATE outbox 
         SET attempts = attempts + 1, 
             last_error = $1 
         WHERE id = $2`,
        [error.message, event.id]
      );
    }
  }
  
  async updateSearchEngine(event) {
    const data = JSON.parse(event.payload);
    await axios.post(`${SEARCH_ENGINE_URL}/update`, data);
  }
  
  async syncWithMLS(event) {
    const data = JSON.parse(event.payload);
    // MLS-specific API call
    await mlsClient.updateListing(data);
  }
  
  async processBuyerAlerts(event) {
    const data = JSON.parse(event.payload);
    
    // Find buyers interested in this property
    const interested = await db.query(`
      SELECT user_id, email, phone 
      FROM buyer_preferences 
      WHERE price_range @> $1
        AND property_type = $2
    `, [data.newPrice, data.propertyType]);
    
    // Queue notifications for each buyer
    for (const buyer of interested.rows) {
      await notificationQueue.add({
        userId: buyer.user_id,
        type: 'PRICE_DROP',
        data: {
          propertyId: data.propertyId,
          oldPrice: data.oldPrice,
          newPrice: data.newPrice,
          savings: data.oldPrice - data.newPrice
        }
      });
    }
  }
}
```

---

## Implementation Patterns

### 1. Polling-Based Publisher

```javascript
class PollingOutboxPublisher {
  constructor(db, broker, options = {}) {
    this.db = db;
    this.broker = broker;
    this.batchSize = options.batchSize || 100;
    this.pollInterval = options.pollInterval || 1000;
    this.maxAttempts = options.maxAttempts || 3;
  }
  
  async start() {
    console.log('Outbox publisher started');
    
    while (true) {
      await this.publishBatch();
      await this.sleep(this.pollInterval);
    }
  }
  
  async publishBatch() {
    const connection = await this.db.getConnection();
    
    try {
      await connection.beginTransaction();
      
      // Get pending events with pessimistic lock
      const events = await connection.query(`
        SELECT * FROM outbox
        WHERE status = 'PENDING'
          AND attempts < $1
        ORDER BY created_at ASC
        LIMIT $2
        FOR UPDATE SKIP LOCKED
      `, [this.maxAttempts, this.batchSize]);
      
      for (const event of events.rows) {
        await this.publishEvent(event, connection);
      }
      
      await connection.commit();
    } catch (error) {
      await connection.rollback();
      console.error('Batch publish failed:', error);
    } finally {
      connection.release();
    }
  }
  
  async publishEvent(event, connection) {
    try {
      // Update to PROCESSING
      await connection.query(
        'UPDATE outbox SET status = $1, updated_at = $2 WHERE id = $3',
        ['PROCESSING', new Date(), event.id]
      );
      
      // Publish to broker
      await this.broker.publish(
        event.event_type,
        JSON.parse(event.payload),
        {
          messageId: event.event_id,
          contentType: 'application/json'
        }
      );
      
      // Mark as SENT
      await connection.query(
        'UPDATE outbox SET status = $1, sent_at = $2, updated_at = $3 WHERE id = $4',
        ['SENT', new Date(), new Date(), event.id]
      );
      
    } catch (error) {
      // Increment attempts
      await connection.query(
        `UPDATE outbox 
         SET attempts = attempts + 1,
             last_error = $1,
             status = CASE WHEN attempts + 1 >= $2 THEN 'FAILED' ELSE 'PENDING' END,
             updated_at = $3
         WHERE id = $4`,
        [error.message, this.maxAttempts, new Date(), event.id]
      );
      
      throw error;
    }
  }
  
  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 2. Change Data Capture (CDC) Pattern

```javascript
// Using PostgreSQL logical replication / Debezium
class CDCOutboxPublisher {
  constructor(kafka) {
    this.kafka = kafka;
    this.producer = kafka.producer();
  }
  
  async start() {
    await this.producer.connect();
    
    // Listen to outbox table changes via replication slot
    const client = new pg.Client({
      host: 'localhost',
      port: 5432,
      database: 'myapp',
      user: 'replicator',
      password: 'password',
      replication: 'database'
    });
    
    await client.connect();
    
    const replicationSlot = await client.query(
      "CREATE_REPLICATION_SLOT outbox_slot LOGICAL OUTPUT PLUGIN pgoutput"
    );
    
    // Stream changes
    const stream = await client.query(
      'START_REPLICATION SLOT outbox_slot LOGICAL 0/0'
    );
    
    stream.on('data', async (msg) => {
      const change = this.parseCDC(msg);
      
      if (change.table === 'outbox' && change.operation === 'INSERT') {
        await this.publishToKafka(change.row);
      }
    });
  }
  
  async publishToKafka(row) {
    await this.producer.send({
      topic: row.event_type,
      messages: [{
        key: row.aggregate_id,
        value: row.payload,
        headers: {
          eventId: row.event_id,
          contentType: 'application/json'
        }
      }]
    });
    
    // Mark as processed in outbox (optional - CDC already captured it)
    await db.query(
      'UPDATE outbox SET cdc_processed = true WHERE id = $1',
      [row.id]
    );
  }
  
  parseCDC(msg) {
    // Parse PostgreSQL logical replication message
    // Returns { table, operation, row }
  }
}
```

### 3. Idempotent Consumer Pattern

```javascript
class IdempotentOutboxConsumer {
  constructor(db) {
    this.db = db;
  }
  
  async consume(message, handler) {
    const { messageId, eventType, payload } = message;
    
    // Check if already processed
    const processed = await this.db.query(
      'SELECT 1 FROM processed_messages WHERE message_id = $1',
      [messageId]
    );
    
    if (processed.rows.length > 0) {
      console.log(`Message ${messageId} already processed, skipping`);
      return { status: 'DUPLICATE' };
    }
    
    const transaction = await this.db.beginTransaction();
    
    try {
      // Process message
      await handler(JSON.parse(payload));
      
      // Record as processed
      await this.db.query(
        `INSERT INTO processed_messages 
         (message_id, event_type, processed_at)
         VALUES ($1, $2, $3)`,
        [messageId, eventType, new Date()],
        { transaction }
      );
      
      await transaction.commit();
      
      return { status: 'PROCESSED' };
    } catch (error) {
      await transaction.rollback();
      
      // Record failure for monitoring
      await this.db.query(
        `INSERT INTO failed_messages 
         (message_id, error, failed_at)
         VALUES ($1, $2, $3)`,
        [messageId, error.message, new Date()]
      );
      
      throw error;
    }
  }
}
```

### 4. Partitioned Outbox Pattern

```javascript
// For high-volume scenarios
class PartitionedOutbox {
  constructor(db, partitions = 10) {
    this.db = db;
    this.partitions = partitions;
  }
  
  async insert(event) {
    // Determine partition based on aggregate_id or event_type
    const partition = this.getPartition(event.aggregate_id);
    
    await this.db.query(
      `INSERT INTO outbox_${partition} 
       (event_id, event_type, aggregate_id, payload, status, created_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [event.event_id, event.event_type, event.aggregate_id, event.payload, 'PENDING', new Date()]
    );
  }
  
  getPartition(aggregateId) {
    // Simple hash-based partitioning
    const hash = this.hashString(aggregateId);
    return Math.abs(hash) % this.partitions;
  }
  
  hashString(str) {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0; // Convert to 32-bit integer
    }
    return hash;
  }
  
  // Publisher for specific partition
  async publishFromPartition(partition) {
    const events = await this.db.query(
      `SELECT * FROM outbox_${partition}
       WHERE status = 'PENDING'
       ORDER BY created_at ASC
       LIMIT 100
       FOR UPDATE SKIP LOCKED`
    );
    
    // Process events...
  }
}
```

---

## Database Schema

```sql
-- Main outbox table
CREATE TABLE outbox (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL UNIQUE,
    event_type VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(50) NOT NULL,
    aggregate_type VARCHAR(50),
    payload JSONB NOT NULL,
    headers JSONB DEFAULT '{}',
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    attempts INT NOT NULL DEFAULT 0,
    last_error TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP,
    sent_at TIMESTAMP,
    
    INDEX idx_outbox_status (status, created_at),
    INDEX idx_outbox_aggregate (aggregate_id, event_type)
);

-- Dead letter queue
CREATE TABLE outbox_dlq (
    id BIGSERIAL PRIMARY KEY,
    event_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    attempts INT NOT NULL,
    last_error TEXT,
    failed_at TIMESTAMP NOT NULL DEFAULT NOW(),
    resolved BOOLEAN DEFAULT FALSE,
    resolved_at TIMESTAMP,
    resolution_notes TEXT
);

-- Processed messages (for idempotent consumers)
CREATE TABLE processed_messages (
    message_id UUID PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    processed_at TIMESTAMP NOT NULL,
    consumer_id VARCHAR(100)
);

-- Monitoring
CREATE TABLE outbox_metrics (
    date DATE NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    total_events INT DEFAULT 0,
    failed_events INT DEFAULT 0,
    avg_latency_ms INT,
    PRIMARY KEY (date, event_type)
);
```

---

## Transactional Outbox vs Alternatives

| Pattern | Pros | Cons | Use Case |
|---------|------|------|----------|
| **Transactional Outbox** | Reliable, exactly-once, no 2PC | Added complexity, polling overhead | Most distributed systems |
| **2PC (XA Transactions)** | Strong consistency | Blocking, poor scalability | Legacy systems, small scale |
| **Saga** | No locking, good scalability | Complex compensation | Long-running transactions |
| **Event Sourcing** | Built-in audit, replayable | Storage growth, complex queries | When you need full history |
| **Kafka Connect** | Managed CDC, scalable | Infrastructure complexity | Large scale, Kafka ecosystem |

---

## Best Practices

### 1. Outbox Table Design
```javascript
// Include metadata for debugging
await db.query(`
  INSERT INTO outbox (
    event_id, event_type, aggregate_id, 
    payload, headers, status, created_at
  ) VALUES ($1, $2, $3, $4, $5, $6, $7)
`, [
  uuid(),
  'order.created',
  orderId,
  JSON.stringify(order),
  JSON.stringify({
    userId: req.user.id,
    ip: req.ip,
    userAgent: req.headers['user-agent']
  }),
  'PENDING',
  new Date()
]);
```

### 2. Exponential Backoff for Retries
```javascript
class RetryStrategy {
  calculateDelay(attempt) {
    // Exponential backoff with jitter
    const baseDelay = Math.pow(2, attempt) * 1000;
    const jitter = Math.random() * 1000;
    return baseDelay + jitter;
  }
  
  async retry(event, publishFn) {
    for (let attempt = 1; attempt <= 3; attempt++) {
      try {
        await publishFn(event);
        return;
      } catch (error) {
        if (attempt === 3) throw error;
        
        const delay = this.calculateDelay(attempt);
        await this.sleep(delay);
      }
    }
  }
}
```

### 3. Monitoring and Alerting
```javascript
class OutboxMonitor {
  async checkHealth() {
    // Check for stuck events
    const stuck = await db.query(`
      SELECT COUNT(*) as count 
      FROM outbox 
      WHERE status = 'PENDING' 
        AND created_at < NOW() - INTERVAL '1 hour'
    `);
    
    if (stuck.rows[0].count > 100) {
      await alert.send('Outbox backlog growing', {
        count: stuck.rows[0].count,
        threshold: 100
      });
    }
    
    // Check failed events
    const failed = await db.query(`
      SELECT COUNT(*) as count 
      FROM outbox 
      WHERE attempts >= 3 AND status != 'DLQ'
    `);
    
    if (failed.rows[0].count > 0) {
      await alert.send('Events exceeded retry limit', {
        count: failed.rows[0].count
      });
    }
  }
  
  async collectMetrics() {
    const metrics = await db.query(`
      SELECT 
        event_type,
        COUNT(*) as total,
        SUM(CASE WHEN status = 'SENT' THEN 1 ELSE 0 END) as sent,
        SUM(CASE WHEN attempts >= 3 THEN 1 ELSE 0 END) as failed,
        AVG(EXTRACT(EPOCH FROM (sent_at - created_at))) as avg_latency
      FROM outbox
      WHERE created_at > NOW() - INTERVAL '1 hour'
      GROUP BY event_type
    `);
    
    await this.saveMetrics(metrics.rows);
  }
}
```

### 4. Cleanup Strategy
```javascript
class OutboxCleanup {
  async cleanupOldEvents() {
    // Move to archive after 7 days
    await db.query(`
      INSERT INTO outbox_archive
      SELECT * FROM outbox 
      WHERE created_at < NOW() - INTERVAL '7 days'
        AND status = 'SENT'
    `);
    
    // Delete from main table
    await db.query(`
      DELETE FROM outbox 
      WHERE created_at < NOW() - INTERVAL '7 days'
        AND status = 'SENT'
    `);
    
    // Keep failed events longer
    await db.query(`
      DELETE FROM outbox_dlq
      WHERE failed_at < NOW() - INTERVAL '30 days'
        AND resolved = true
    `);
  }
}
```

---

## When to Use Transactional Outbox

### ✅ DO Use When:
- You need to publish events reliably
- You have dual-write problems (DB + message broker)
- Message loss is unacceptable
- You need exactly-once or at-least-once delivery
- Building event-driven microservices
- Compliance requires audit trails

### ❌ DON'T Use When:
- Messages can be lost (logging, analytics)
- You use Kafka with exactly-once semantics
- Simple monolith with no external systems
- Real-time requirements (< 100ms) for publishing
- Using 2PC is acceptable (small scale)

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Duplicate Messages** | Idempotent consumers, deduplication |
| **Outbox Table Growth** | Regular cleanup, archiving |
| **Publisher Single Point of Failure** | Multiple publisher instances with SKIP LOCKED |
| **Message Ordering** | Partition by aggregate_id, single partition per aggregate |
| **Large Payloads** | Store in S3, reference in outbox |
| **Transaction Timeout** | Keep transactions short, batch appropriately |
| **Monitoring Blindness** | Track metrics, alert on backlog |

---

## Summary Checklist

| Component | Implementation |
|-----------|---------------|
| **Outbox Table** | Part of business transaction |
| **Publisher** | Polling or CDC-based |
| **Idempotency** | Consumer-side deduplication |
| **Retries** | Exponential backoff, max attempts |
| **Dead Letter Queue** | Store permanently failed events |
| **Monitoring** | Track backlog, latency, failures |
| **Cleanup** | Archive old events |
| **Ordering** | Partition by aggregate if needed |

**Bottom Line:** Transactional Outbox is the most reliable way to publish events from a database. It guarantees that messages are eventually sent exactly once, solving the dual-write problem without 2PC. Essential for any event-driven microservice architecture.