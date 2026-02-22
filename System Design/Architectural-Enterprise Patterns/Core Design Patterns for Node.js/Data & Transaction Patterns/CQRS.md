# CQRS (Command Query Responsibility Segregation)

## Definition
**CQRS** is an architectural pattern that separates read and write operations into different models. **Commands** handle writes (create, update, delete) and **Queries** handle reads. This segregation allows each model to be optimized independently for its specific workload.

The pattern can be applied with or without separate databases and is often used with Event Sourcing.

---

## Core Concept
```
                    ┌─────────────────────────────────────┐
                    │           CLIENT/APPLICATION        │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────┴───────────────────┐
                    │                                    │
            ┌───────▼───────┐                    ┌───────▼───────┐
            │    COMMAND    │                    │     QUERY     │
            │    MODEL      │                    │    MODEL      │
            │  (Write)      │                    │   (Read)      │
            └───────┬───────┘                    └───────┬───────┘
                    │                                    │
            ┌───────▼───────┐                    ┌───────▼───────┐
            │  COMMAND DB   │                    │   QUERY DB    │
            │  (Normalized) │◄────────Sync────────┤ (Denormalized)│
            └───────────────┘      (Eventual)     └───────────────┘
```

---

## Good vs Bad Example

### Good Example
**E-commerce Product Management:**
```javascript
// COMMAND SIDE - Handles writes
app.post('/api/products', async (req, res) => {
  const product = await Product.create(req.body); // Normalized write
  await eventBus.publish('product.created', product);
  res.status(201).json({ id: product.id });
});

app.put('/api/products/:id/inventory', async (req, res) => {
  await Product.updateInventory(req.params.id, req.body.quantity);
  await eventBus.publish('inventory.updated', { 
    productId: req.params.id, 
    quantity: req.body.quantity 
  });
  res.status(200).send();
});

// QUERY SIDE - Handles reads (separate DB)
app.get('/api/products/search', async (req, res) => {
  // Directly from denormalized read DB - optimized for search
  const products = await readDB.query(`
    SELECT p.*, 
           COALESCE(AVG(r.rating), 0) as avgRating,
           COUNT(r.id) as reviewCount,
           i.quantity as stock
    FROM product_view p
    LEFT JOIN review_view r ON p.id = r.productId
    LEFT JOIN inventory_view i ON p.id = i.productId
    WHERE p.category = $1
    GROUP BY p.id, i.quantity
  `, [req.query.category]);
  
  res.json(products);
});
```

### Bad Example
**Monolithic CRUD:**
```javascript
// Single model trying to do everything
app.get('/api/products/:id', async (req, res) => {
  const product = await db.products.findById(req.params.id);
  const reviews = await db.reviews.findByProduct(req.params.id);
  const inventory = await db.inventory.findByProduct(req.params.id);
  
  // Complex join logic in controller
  res.json({
    ...product,
    avgRating: calculateAvg(reviews),
    inStock: inventory.quantity > 0,
    reviews: reviews
  });
});

app.post('/api/products', async (req, res) => {
  // Write and read from same DB
  const product = await db.products.create(req.body);
  await db.inventory.create({ productId: product.id, quantity: 0 });
  await db.searchIndex.update(product); // Blocking search update
  
  res.json(product); // Returns everything
});
```
**Problems:**
- Read queries slow due to joins
- Write operations blocked by read load
- Can't scale reads and writes independently
- One schema tries to serve all needs

---

## Industry Use Cases

### 1. BFSI (Banking) - Transaction System

**Command Side (Write Model):**
```javascript
// Account Command Model
class AccountCommand {
  async openAccount(command) {
    const account = {
      id: uuid(),
      customerId: command.customerId,
      type: command.accountType,
      balance: 0,
      status: 'PENDING_APPROVAL',
      createdAt: new Date(),
      complianceFlags: []
    };
    
    // Validate business rules
    await this.checkDuplicateApplication(command.customerId);
    await this.validateKYC(command.customerId);
    
    // Store in command DB (normalized)
    await commandDB.accounts.insert(account);
    
    // Publish events for query side
    await eventBus.publish('account.opened', account);
    
    return account.id;
  }
  
  async deposit(command) {
    const account = await commandDB.accounts.findById(command.accountId);
    
    // Business rules
    if (account.status !== 'ACTIVE') {
      throw new Error('Account not active');
    }
    
    // Update balance (ACID transaction)
    await commandDB.transaction(async (trx) => {
      await trx.accounts.updateBalance(
        command.accountId, 
        account.balance + command.amount
      );
      
      await trx.transactions.insert({
        accountId: command.accountId,
        type: 'DEPOSIT',
        amount: command.amount,
        timestamp: new Date()
      });
    });
    
    // Publish event
    await eventBus.publish('account.deposited', {
      accountId: command.accountId,
      amount: command.amount,
      newBalance: account.balance + command.amount
    });
  }
}
```

**Query Side (Read Model):**
```javascript
// Account Query Model - Optimized for different views
class AccountQuery {
  async getCustomerDashboard(customerId) {
    // Denormalized view - pre-joined for performance
    return await queryDB.query(`
      SELECT 
        a.id,
        a.type,
        a.balance,
        a.status,
        COALESCE(
          (SELECT json_agg(
            json_build_object(
              'id', t.id,
              'amount', t.amount,
              'date', t.created_at,
              'description', t.description
            )
            ORDER BY t.created_at DESC
            LIMIT 5
          ) FROM transactions t WHERE t.account_id = a.id),
          '[]'::json
        ) as recent_transactions,
        COALESCE(
          (SELECT SUM(t.amount) 
           FROM transactions t 
           WHERE t.account_id = a.id 
           AND t.created_at >= NOW() - INTERVAL '30 days'
           AND t.type = 'WITHDRAWAL'),
          0
        ) as monthly_withdrawals
      FROM accounts a
      WHERE a.customer_id = $1
    `, [customerId]);
  }
  
  async getMonthlyStatement(accountId, month) {
    // Optimized for statement generation
    return await queryDB.query(`
      SELECT 
        date_trunc('day', t.created_at) as day,
        json_agg(
          json_build_object(
            'id', t.id,
            'type', t.type,
            'amount', t.amount,
            'description', t.description,
            'balance', t.running_balance
          ) ORDER BY t.created_at
        ) as transactions,
        SUM(CASE WHEN t.type = 'DEPOSIT' THEN t.amount ELSE 0 END) as total_deposits,
        SUM(CASE WHEN t.type = 'WITHDRAWAL' THEN t.amount ELSE 0 END) as total_withdrawals
      FROM transactions t
      WHERE t.account_id = $1
        AND date_trunc('month', t.created_at) = date_trunc('month', $2::date)
      GROUP BY date_trunc('day', t.created_at)
      ORDER BY day DESC
    `, [accountId, month]);
  }
}
```

**Benefits for BFSI:**
- **Audit Trail:** Every command creates events
- **Compliance Queries:** Read models optimized for reporting
- **Transaction Integrity:** ACID on command side
- **Scalability:** Read replicas handle thousands of balance checks

### 2. Real Estate - Property Management

**Command Side (Write Model):**
```javascript
class PropertyCommand {
  async listProperty(command) {
    const property = {
      id: uuid(),
      address: command.address,
      price: command.price,
      specs: command.specs,
      ownerId: command.ownerId,
      status: 'DRAFT',
      createdAt: new Date()
    };
    
    // Validate business rules
    await this.validateAddress(command.address);
    await this.checkDuplicateListing(command.address);
    
    // Store in command DB (normalized)
    await commandDB.properties.insert(property);
    
    // Publish events
    await eventBus.publish('property.listed', property);
    
    return property.id;
  }
  
  async updatePrice(command) {
    await commandDB.transaction(async (trx) => {
      const property = await trx.properties.findById(command.propertyId);
      
      // Record price history
      await trx.priceHistory.insert({
        propertyId: command.propertyId,
        oldPrice: property.price,
        newPrice: command.newPrice,
        reason: command.reason,
        changedBy: command.agentId,
        timestamp: new Date()
      });
      
      // Update property
      await trx.properties.updatePrice(
        command.propertyId, 
        command.newPrice
      );
    });
    
    await eventBus.publish('property.price.changed', {
      propertyId: command.propertyId,
      newPrice: command.newPrice,
      oldPrice: oldPrice
    });
  }
}
```

**Query Side (Read Model):**
```javascript
class PropertyQuery {
  async searchProperties(criteria) {
    // Denormalized search view - pre-computed for fast filtering
    return await queryDB.query(`
      SELECT 
        p.id,
        p.address,
        p.price,
        p.bedrooms,
        p.bathrooms,
        p.square_feet,
        p.lot_size,
        pi.thumbnail_url,
        ps.avg_rating as school_rating,
        nc.crime_rate,
        nc.walk_score,
        (
          SELECT COUNT(*) 
          FROM saved_searches ss 
          WHERE ss.property_id = p.id 
          AND ss.user_id = $1
        ) as saved_count
      FROM property_search_view p
      LEFT JOIN property_images pi ON p.id = pi.property_id AND pi.is_primary = true
      LEFT JOIN neighborhood_stats nc ON p.zip_code = nc.zip_code
      LEFT JOIN school_ratings ps ON p.school_district_id = ps.district_id
      WHERE p.price BETWEEN $2 AND $3
        AND p.bedrooms >= $4
        AND ST_DWithin(p.location, ST_MakePoint($5, $6), $7)
        AND p.status = 'ACTIVE'
      ORDER BY 
        CASE WHEN $8 = 'price_asc' THEN p.price END ASC,
        CASE WHEN $8 = 'price_desc' THEN p.price END DESC,
        CASE WHEN $8 = 'newest' THEN p.listed_at END DESC
      LIMIT $9 OFFSET $10
    `, [
      criteria.userId,
      criteria.minPrice,
      criteria.maxPrice,
      criteria.minBeds,
      criteria.longitude,
      criteria.latitude,
      criteria.radius,
      criteria.sortBy,
      criteria.limit,
      criteria.offset
    ]);
  }
  
  async getPropertyDetails(propertyId, userId) {
    // Rich, denormalized property view
    return await queryDB.one(`
      SELECT 
        p.*,
        COALESCE(
          (SELECT json_agg(
            json_build_object(
              'url', pi.url,
              'caption', pi.caption,
              'isPrimary', pi.is_primary
            ) ORDER BY pi.sort_order
          ) FROM property_images pi WHERE pi.property_id = p.id),
          '[]'::json
        ) as images,
        COALESCE(
          (SELECT json_agg(
            json_build_object(
              'date', oh.date,
              'startTime', oh.start_time,
              'endTime', oh.end_time,
              'agent', json_build_object(
                'name', a.name,
                'phone', a.phone,
                'photo', a.photo_url
              )
            ) ORDER BY oh.date
          ) FROM open_houses oh
          JOIN agents a ON oh.agent_id = a.id
          WHERE oh.property_id = p.id 
            AND oh.date >= CURRENT_DATE
            LIMIT 3),
          '[]'::json
        ) as upcoming_open_houses,
        (
          SELECT json_agg(
            json_build_object(
              'price', ph.price,
              'date', ph.changed_at,
              'reason', ph.reason
            ) ORDER BY ph.changed_at DESC
          ) FROM price_history ph WHERE ph.property_id = p.id
        ) as price_history,
        EXISTS(
          SELECT 1 FROM user_favorites uf 
          WHERE uf.property_id = p.id AND uf.user_id = $1
        ) as is_favorite
      FROM properties p
      WHERE p.id = $2
    `, [userId, propertyId]);
  }
}
```

**Benefits for Real Estate:**
- **Fast Search:** Pre-joined views with geospatial indexing
- **Rich Details:** Denormalized property pages load instantly
- **MLS Sync:** Multiple sources publish to command side
- **Analytics:** Separate read models for market reports

### 3. E-commerce - Order System

**Command Side (Write Model):**
```javascript
class OrderCommand {
  async createOrder(command) {
    const orderId = uuid();
    
    await commandDB.transaction(async (trx) => {
      // Validate inventory
      for (const item of command.items) {
        const available = await trx.inventory.getAvailable(
          item.productId, 
          item.quantity
        );
        
        if (!available) {
          throw new Error(`Insufficient inventory for ${item.productId}`);
        }
        
        // Reserve inventory
        await trx.inventory.reserve(
          item.productId, 
          item.quantity, 
          orderId
        );
      }
      
      // Create order
      await trx.orders.insert({
        id: orderId,
        userId: command.userId,
        items: command.items,
        shippingAddress: command.shippingAddress,
        paymentMethod: command.paymentMethod,
        status: 'PENDING_PAYMENT',
        createdAt: new Date(),
        total: command.items.reduce(
          (sum, i) => sum + (i.price * i.quantity), 0
        )
      });
      
      // Create payment intent
      await trx.payments.insert({
        orderId,
        amount: command.total,
        method: command.paymentMethod,
        status: 'PENDING',
        idempotencyKey: command.idempotencyKey
      });
    });
    
    // Publish events
    await eventBus.publish('order.created', {
      orderId,
      userId: command.userId,
      amount: command.total,
      items: command.items
    });
    
    return orderId;
  }
  
  async confirmPayment(command) {
    await commandDB.transaction(async (trx) => {
      // Update payment status
      await trx.payments.updateStatus(
        command.orderId, 
        'COMPLETED',
        command.paymentDetails
      );
      
      // Update order status
      await trx.orders.updateStatus(
        command.orderId, 
        'PAYMENT_CONFIRMED'
      );
      
      // Convert reservations to actual inventory deduction
      await trx.inventory.confirmReservations(command.orderId);
    });
    
    // Publish events
    await eventBus.publish('order.payment.confirmed', {
      orderId: command.orderId,
      confirmedAt: new Date()
    });
  }
}
```

**Query Side (Read Model):**
```javascript
class OrderQuery {
  async getOrderDetails(orderId, userId) {
    // Denormalized order view for customers
    return await queryDB.one(`
      SELECT 
        o.id,
        o.status,
        o.created_at,
        o.total,
        (
          SELECT json_agg(
            json_build_object(
              'productId', oi.product_id,
              'productName', p.name,
              'quantity', oi.quantity,
              'price', oi.price,
              'subtotal', oi.price * oi.quantity,
              'image', p.thumbnail_url,
              'deliveryDate', oi.estimated_delivery
            )
          ) FROM order_items oi
          JOIN products p ON oi.product_id = p.id
          WHERE oi.order_id = o.id
        ) as items,
        (
          SELECT row_to_json(sa.*)
          FROM shipping_addresses sa
          WHERE sa.id = o.shipping_address_id
        ) as shipping_address,
        (
          SELECT row_to_json(pm.*)
          FROM payment_methods pm
          WHERE pm.id = o.payment_method_id
        ) as payment_method,
        (
          SELECT json_build_object(
            'number', t.tracking_number,
            'carrier', t.carrier,
            'status', t.status,
            'estimatedDelivery', t.estimated_delivery,
            'events', t.events
          ) FROM shipments t WHERE t.order_id = o.id
        ) as tracking
      FROM orders o
      WHERE o.id = $1 AND o.user_id = $2
    `, [orderId, userId]);
  }
  
  async getUserOrders(userId, status, page, limit) {
    // Optimized order history
    return await queryDB.query(`
      SELECT 
        o.id,
        o.status,
        o.created_at,
        o.total,
        (
          SELECT json_agg(
            json_build_object(
              'productId', oi.product_id,
              'productName', p.name,
              'quantity', oi.quantity,
              'price', oi.price,
              'image', p.thumbnail_url
            )
          ) FROM order_items oi
          JOIN products p ON oi.product_id = p.id
          WHERE oi.order_id = o.id
          LIMIT 3
        ) as items_preview,
        COUNT(*) OVER() as total_count
      FROM orders o
      WHERE o.user_id = $1
        AND ($2::text IS NULL OR o.status = $2)
      ORDER BY o.created_at DESC
      LIMIT $3 OFFSET $4
    `, [userId, status, limit, (page - 1) * limit]);
  }
  
  async getSalesAnalytics(startDate, endDate) {
    // Analytics read model for business intelligence
    return await queryDB.one(`
      SELECT 
        COUNT(DISTINCT o.id) as total_orders,
        SUM(o.total) as total_revenue,
        AVG(o.total) as avg_order_value,
        COUNT(DISTINCT o.user_id) as unique_customers,
        (
          SELECT json_agg(
            json_build_object(
              'date', date_trunc('day', o2.created_at),
              'orders', COUNT(DISTINCT o2.id),
              'revenue', SUM(o2.total)
            )
            FROM orders o2
            WHERE o2.created_at BETWEEN $1 AND $2
            GROUP BY date_trunc('day', o2.created_at)
            ORDER BY date_trunc('day', o2.created_at)
          )
        ) as daily_breakdown,
        (
          SELECT json_agg(
            json_build_object(
              'productId', p.id,
              'productName', p.name,
              'quantitySold', SUM(oi.quantity),
              'revenue', SUM(oi.quantity * oi.price)
            )
            FROM order_items oi
            JOIN products p ON oi.product_id = p.id
            JOIN orders o3 ON oi.order_id = o3.id
            WHERE o3.created_at BETWEEN $1 AND $2
            GROUP BY p.id, p.name
            ORDER BY SUM(oi.quantity) DESC
            LIMIT 10
          )
        ) as top_products
      FROM orders o
      WHERE o.created_at BETWEEN $1 AND $2
        AND o.status != 'CANCELLED'
    `, [startDate, endDate]);
  }
}
```

**Benefits for E-commerce:**
- **Fast Checkout:** Command side focuses on write integrity
- **Product Search:** Read models optimized for filtering
- **Order History:** Denormalized for quick retrieval
- **Real-time Inventory:** Separate read model for stock checks

---

## Data Patterns in CQRS

### 1. Separate Databases Pattern

```javascript
// Configuration
const commandDB = {
  client: 'pg',
  connection: process.env.COMMAND_DB_URL,
  pool: { min: 2, max: 10 } // Optimized for writes
};

const queryDB = {
  client: 'pg',
  connection: process.env.QUERY_DB_URL,
  pool: { min: 10, max: 50 } // Optimized for reads
};

// Schema differences
// Command DB - Normalized
`
CREATE TABLE accounts (
  id UUID PRIMARY KEY,
  customer_id UUID NOT NULL,
  balance DECIMAL NOT NULL,
  version INT NOT NULL,
  created_at TIMESTAMP NOT NULL
);

CREATE TABLE transactions (
  id UUID PRIMARY KEY,
  account_id UUID REFERENCES accounts(id),
  amount DECIMAL NOT NULL,
  type VARCHAR(20) NOT NULL,
  created_at TIMESTAMP NOT NULL
);
`

// Query DB - Denormalized
`
CREATE TABLE customer_portfolio (
  customer_id UUID PRIMARY KEY,
  total_balance DECIMAL NOT NULL,
  account_count INT NOT NULL,
  recent_transactions JSONB,
  monthly_spending JSONB,
  last_updated TIMESTAMP NOT NULL
);

CREATE INDEX idx_customer_portfolio_updated 
ON customer_portfolio(last_updated);
`
```

### 2. Same Database, Different Schemas Pattern

```javascript
// Using PostgreSQL schemas
const commandSchema = 'command';
const querySchema = 'query';

// Command tables in 'command' schema
await commandDB.raw(`
  CREATE TABLE ${commandSchema}.orders (
    id UUID PRIMARY KEY,
    data JSONB NOT NULL,
    version INT NOT NULL
  );
`);

// Read-only views in 'query' schema
await commandDB.raw(`
  CREATE MATERIALIZED VIEW ${querySchema}.order_summary AS
  SELECT 
    o.id,
    o.data->>'userId' as user_id,
    o.data->>'status' as status,
    o.data->>'total' as total,
    o.data->>'createdAt' as created_at
  FROM ${commandSchema}.orders o
  WHERE o.data->>'status' != 'CANCELLED';
  
  CREATE INDEX ON ${querySchema}.order_summary(user_id);
  CREATE INDEX ON ${querySchema}.order_summary(created_at);
`);
```

### 3. Event-Driven Sync Pattern

```javascript
// Event handlers to update read models
class QueryUpdater {
  constructor() {
    // Subscribe to all domain events
    eventBus.subscribe('account.opened', this.handleAccountOpened);
    eventBus.subscribe('account.deposited', this.handleAccountDeposited);
    eventBus.subscribe('account.withdrawn', this.handleAccountWithdrawn);
  }
  
  async handleAccountOpened(event) {
    // Update multiple read models atomically
    await queryDB.transaction(async (trx) => {
      // Update customer portfolio view
      await trx('customer_portfolio')
        .insert({
          customer_id: event.customerId,
          accounts: JSON.stringify([{
            id: event.accountId,
            type: event.accountType,
            balance: 0
          }]),
          total_balance: 0,
          last_updated: new Date()
        })
        .onConflict('customer_id')
        .merge({
          accounts: trx.raw(
            `accounts || ?::jsonb`,
            [JSON.stringify([{ id: event.accountId, type: event.accountType, balance: 0 }])]
          ),
          last_updated: new Date()
        });
      
      // Update account list view
      await trx('account_list').insert({
        account_id: event.accountId,
        customer_id: event.customerId,
        account_type: event.accountType,
        status: 'ACTIVE',
        opened_at: event.timestamp
      });
    });
  }
  
  async handleAccountDeposited(event) {
    await queryDB.transaction(async (trx) => {
      // Update account balance in portfolio
      await trx.raw(`
        UPDATE customer_portfolio 
        SET accounts = jsonb_set(
          accounts,
          $1,
          $2,
          true
        ),
        total_balance = total_balance + $3,
        last_updated = NOW()
        WHERE customer_id = $4
      `, [
        `{${event.accountIndex},balance}`,
        `"${event.newBalance}"`,
        event.amount,
        event.customerId
      ]);
      
      // Insert into transaction feed
      await trx('transaction_feed').insert({
        account_id: event.accountId,
        amount: event.amount,
        type: 'DEPOSIT',
        balance_after: event.newBalance,
        created_at: event.timestamp
      });
    });
  }
}
```

---

## Transaction Patterns in CQRS

### 1. Command-Side Transactions (ACID)

```javascript
class TransferCommand {
  async transferMoney(command) {
    // ACID transaction on command side
    await commandDB.transaction(async (trx) => {
      // Lock accounts for update (pessimistic locking)
      const fromAccount = await trx('accounts')
        .where('id', command.fromAccountId)
        .forUpdate()
        .first();
      
      const toAccount = await trx('accounts')
        .where('id', command.toAccountId)
        .forUpdate()
        .first();
      
      // Business validation
      if (fromAccount.balance < command.amount) {
        throw new InsufficientFundsError();
      }
      
      if (command.amount > 10000) {
        // Flag for compliance review
        await trx('compliance_flags').insert({
          account_id: command.fromAccountId,
          type: 'LARGE_TRANSFER',
          amount: command.amount,
          status: 'PENDING_REVIEW'
        });
        
        // Don't complete transfer yet
        return { status: 'PENDING_COMPLIANCE' };
      }
      
      // Update balances
      await trx('accounts')
        .where('id', command.fromAccountId)
        .update({
          balance: fromAccount.balance - command.amount,
          version: fromAccount.version + 1
        });
      
      await trx('accounts')
        .where('id', command.toAccountId)
        .update({
          balance: toAccount.balance + command.amount,
          version: toAccount.version + 1
        });
      
      // Record transaction
      const transaction = {
        id: uuid(),
        from_account: command.fromAccountId,
        to_account: command.toAccountId,
        amount: command.amount,
        status: 'COMPLETED',
        created_at: new Date()
      };
      
      await trx('transactions').insert(transaction);
      
      return { status: 'COMPLETED', transactionId: transaction.id };
    });
  }
}
```

### 2. Eventual Consistency with Outbox Pattern

```javascript
// Transactional Outbox Pattern
class OrderService {
  async createOrder(command) {
    const orderId = uuid();
    
    // Single transaction that writes to both tables
    await commandDB.transaction(async (trx) => {
      // 1. Insert order
      await trx('orders').insert({
        id: orderId,
        user_id: command.userId,
        data: command,
        status: 'CREATED',
        created_at: new Date()
      });
      
      // 2. Insert into outbox (same transaction!)
      await trx('outbox').insert({
        id: uuid(),
        aggregate_type: 'order',
        aggregate_id: orderId,
        event_type: 'order.created',
        payload: JSON.stringify({
          orderId,
          userId: command.userId,
          amount: command.total,
          items: command.items
        }),
        created_at: new Date()
      });
    });
    
    return orderId;
  }
}

// Outbox publisher (runs separately)
class OutboxPublisher {
  async publishEvents() {
    // Poll outbox table
    const events = await commandDB('outbox')
      .where('published', false)
      .limit(100)
      .forUpdate()
      .skipLocked(); // Skip locked rows for concurrent publishers
    
    for (const event of events) {
      try {
        // Publish to message broker
        await eventBus.publish(event.event_type, JSON.parse(event.payload));
        
        // Mark as published
        await commandDB('outbox')
          .where('id', event.id)
          .update({ published: true, published_at: new Date() });
      } catch (error) {
        // Log error, will be retried
        logger.error('Failed to publish event', { eventId: event.id, error });
      }
    }
  }
}
```

### 3. Saga Pattern for Distributed Transactions

```javascript
// Saga for order fulfillment across services
class OrderFulfillmentSaga {
  async execute(orderId) {
    const sagaId = uuid();
    const context = { orderId };
    
    try {
      // Step 1: Reserve inventory
      context.inventoryResult = await this.reserveInventory(orderId);
      await this.saveSagaState(sagaId, 'INVENTORY_RESERVED', context);
      
      // Step 2: Process payment
      context.paymentResult = await this.processPayment(orderId);
      await this.saveSagaState(sagaId, 'PAYMENT_PROCESSED', context);
      
      // Step 3: Create shipment
      context.shipmentResult = await this.createShipment(orderId);
      await this.saveSagaState(sagaId, 'SHIPMENT_CREATED', context);
      
      // Step 4: Confirm order
      await this.confirmOrder(orderId);
      await this.saveSagaState(sagaId, 'COMPLETED', context);
      
      return { success: true };
    } catch (error) {
      // Compensating transactions
      await this.compensate(sagaId, context);
      throw error;
    }
  }
  
  async compensate(sagaId, context) {
    const state = await this.getSagaState(sagaId);
    
    // Roll back in reverse order
    if (state === 'SHIPMENT_CREATED') {
      await this.cancelShipment(context.shipmentResult.id);
    }
    
    if (state === 'PAYMENT_PROCESSED' || state === 'SHIPMENT_CREATED') {
      await this.refundPayment(context.paymentResult.id);
    }
    
    if (state === 'INVENTORY_RESERVED' || state === 'PAYMENT_PROCESSED') {
      await this.releaseInventory(context.orderId);
    }
    
    await this.saveSagaState(sagaId, 'COMPENSATED', context);
  }
  
  async reserveInventory(orderId) {
    // Call inventory service
    const response = await axios.post(`${INVENTORY_SERVICE}/reserve`, {
      orderId,
      items: await this.getOrderItems(orderId)
    });
    
    return response.data;
  }
  
  // Compensating action
  async releaseInventory(orderId) {
    await axios.post(`${INVENTORY_SERVICE}/release`, { orderId });
  }
}
```

### 4. Optimistic Concurrency Control

```javascript
class AccountCommand {
  async updateBalance(command) {
    const retries = 3;
    
    for (let i = 0; i < retries; i++) {
      try {
        // Read current version
        const account = await commandDB('accounts')
          .where('id', command.accountId)
          .first();
        
        // Update with version check
        const updated = await commandDB('accounts')
          .where({
            id: command.accountId,
            version: account.version // Optimistic lock
          })
          .update({
            balance: account.balance + command.amount,
            version: account.version + 1
          });
        
        if (updated === 0) {
          // Version mismatch, retry
          if (i === retries - 1) {
            throw new ConcurrencyError('Failed to update after retries');
          }
          await this.sleep(Math.pow(2, i) * 100); // Exponential backoff
          continue;
        }
        
        // Success
        return;
      } catch (error) {
        if (i === retries - 1) throw error;
      }
    }
  }
}
```

---

## When to Use CQRS

### ✅ DO Use When:
- **Different read/write loads** (e.g., 90% reads, 10% writes)
- **Complex business logic** on write side
- **Multiple read views** of same data
- **Performance requirements** demand optimization
- **Team structure** allows separate read/write ownership
- **Audit/compliance** needs event sourcing

### ❌ DON'T Use When:
- **Simple CRUD** applications
- **Real-time consistency** required
- **Small team** can't manage complexity
- **Domain not complex** enough to justify
- **Startup/MVP** building initial product

---

## CQRS vs Traditional CRUD

| Aspect | Traditional CRUD | CQRS |
|--------|-----------------|------|
| **Model** | Single model | Separate read/write models |
| **Database** | Single DB | Possibly separate DBs |
| **Consistency** | Strong (ACID) | Eventual |
| **Complexity** | Low | High |
| **Performance** | Balanced | Optimized per operation |
| **Scaling** | Vertical | Horizontal (reads/writes separate) |
| **Use Case** | Simple apps | Complex domains |

---

## Summary Checklist

| Component | Implementation |
|-----------|---------------|
| **Commands** | Validate, apply business rules, emit events |
| **Queries** | Optimized for specific views, denormalized |
| **Sync** | Event-driven, outbox pattern |
| **Transactions** | ACID on command side, eventual consistency |
| **Concurrency** | Optimistic locking, version fields |
| **Monitoring** | Track sync lag, command failures |

**Bottom Line:** CQRS is powerful for complex domains with different read/write patterns. Use it when the benefits of separation outweigh the added complexity, and always pair with proper transaction patterns for data integrity.