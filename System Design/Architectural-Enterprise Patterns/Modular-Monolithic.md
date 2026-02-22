# Modular Monolithic Architecture

## Definition
**Modular Monolithic Architecture** structures an application as a single deployable unit but with **clear, domain-bound modules**. Each module has its own responsibilities, logic, and data, but they all run in the same process and are deployed together.

It combines **monolith simplicity** with **microservices modularity**—clean separation without distributed complexity.

---

## Core Concept
```
┌─────────────────────────────────────────────────┐
│             SINGLE DEPLOYABLE UNIT              │
├─────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Order   │  │ Payment  │  │Inventory │      │
│  │  Module  │◄─┤ Module   │  │ Module   │      │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘      │
│       │             │             │            │
│  ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐      │
│  │Order DB  │  │Payment DB│  │Inventory │      │
│  │(Schema)  │  │(Schema)  │  │DB (Schema)│     │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                  │
│         SHARED DATABASE (Separate Schemas)      │
└─────────────────────────────────────────────────┘
```

---

## Comparison

| Aspect | Traditional Monolith | Modular Monolith | Microservices |
|--------|---------------------|------------------|---------------|
| **Deployment** | Single unit | Single unit | Multiple units |
| **Module Communication** | Direct calls | Interface-based | Network calls |
| **Database** | Single schema | Separate schemas | Separate DBs |
| **Team Autonomy** | Low | Medium | High |
| **Complexity** | Low | Medium | High |

---

## Good Example (Modular)

```javascript
// modules/order/src/services/orderService.js
const paymentInterface = require('../../payment/interfaces/paymentInterface');
const inventoryInterface = require('../../inventory/interfaces/inventoryInterface');

class OrderService {
  async createOrder(data) {
    const order = await orderRepo.create(data);
    
    // Call other modules via interfaces
    await paymentInterface.process(order.id, order.total);
    await inventoryInterface.reserve(order.id, order.items);
    
    return order;
  }
}

// modules/payment/interfaces/paymentInterface.js
module.exports = {
  async process(orderId, amount) {
    // Internal implementation hidden
    return paymentService.process(orderId, amount);
  }
};
```

## Bad Example (Traditional)

```javascript
// Mixed concerns, no boundaries
app.post('/api/orders', async (req, res) => {
  const order = await db.orders.create(req.body);
  await db.payments.create({ orderId: order.id, amount: order.total });
  await db.inventory.updateStock(order.items);
  await email.send(order.userId, 'Confirmation');
  res.json(order);
});
```

---

## Industry Use Cases

### 1. BFSI (Banking)
**Modules:** Account, Customer, Transaction, Loan, Reporting

```javascript
// modules/account/interfaces/accountInterface.js
module.exports = {
  async getBalance(accountId) {
    return accountService.getBalance(accountId);
  },
  async validateAccount(accountId, amount) {
    return accountService.validate(accountId, amount);
  }
};

// modules/transaction/services/transferService.js
const accountInterface = require('../../account/interfaces/accountInterface');

async function transfer(from, to, amount) {
  const valid = await accountInterface.validateAccount(from, amount);
  if (!valid) throw new Error('Insufficient funds');
  
  // Process transfer...
}
```

### 2. E-commerce
**Modules:** Catalog, Cart, Order, Payment, Inventory, Shipping, Customer, Review

```javascript
// modules/catalog/interfaces/catalogInterface.js
module.exports = {
  async getProduct(id) {
    return catalogService.getProduct(id);
  },
  async validateProducts(items) {
    return catalogService.validate(items);
  }
};

// modules/order/services/orderService.js
const catalog = require('../../catalog/interfaces/catalogInterface');
const inventory = require('../../inventory/interfaces/inventoryInterface');

async function createOrder(data) {
  const products = await catalog.validateProducts(data.items);
  await inventory.reserve(data.items);
  // Create order...
}
```

### 3. Real Estate
**Modules:** Property, Search, Showing, Offer, User, Document

```javascript
// modules/property/interfaces/propertyInterface.js
module.exports = {
  async getProperty(id) {
    return propertyService.getProperty(id);
  },
  async updatePrice(id, price) {
    return propertyService.updatePrice(id, price);
  }
};

// modules/offer/services/offerService.js
const property = require('../../property/interfaces/propertyInterface');

async function submitOffer(propertyId, amount) {
  const property = await propertyInterface.getProperty(propertyId);
  if (amount < property.price * 0.8) {
    throw new Error('Offer too low');
  }
  // Create offer...
}
```

---

## Best Practices

### 1. Module Boundaries
```javascript
// modules/order/index.js - Public exports only
module.exports = {
  services: {
    createOrder: require('./services/orderService').createOrder,
    getOrder: require('./services/orderService').getOrder
  },
  interfaces: {
    validateOrder: require('./interfaces/orderInterface').validateOrder
  }
};
```

### 2. Database Separation
```sql
-- Separate schemas per module
CREATE SCHEMA order;
CREATE SCHEMA payment;
CREATE SCHEMA inventory;

-- Tables in respective schemas
CREATE TABLE order.orders (...);
CREATE TABLE payment.payments (...);
CREATE TABLE inventory.stock (...);
```

### 3. Module Communication
```javascript
// ALWAYS use interfaces, NEVER direct repository access
// GOOD
const payment = require('../../payment/interfaces/paymentInterface');
await payment.process(orderId, amount);

// BAD - direct access
const Payment = require('../../payment/models/Payment');
await Payment.create({ orderId, amount });
```

### 4. Module Independence
```javascript
// Each module should be potentially extractable
modules/
  order/
    src/
      controllers/    (API endpoints)
      services/       (business logic)
      repositories/   (data access)
      models/         (domain models)
      interfaces/     (public API for other modules)
    tests/
    package.json      (optional, for future extraction)
```

### 5. Shared Kernel
```javascript
// shared/ - Common utilities (minimal)
shared/
  database/
    connection.js     (DB connection pool)
  logging/
    logger.js         (centralized logging)
  validation/
    schemas.js        (shared validation)
  eventBus/
    index.js          (in-memory event bus)
```

---

## Where to Use Modular Monolith

### ✅ **USE WHEN:**
- **Medium-sized teams** (5-15 developers)
- **Complex domain** with clear boundaries
- **Future microservices** planned
- **Startups/MVPs** that may scale
- **Clear separation** of concerns needed

### ❌ **DON'T USE WHEN:**
- **Tiny team** (1-3 devs) - keep it simple
- **Trivial CRUD** - overengineering
- **Need independent scaling** - use microservices
- **Different tech stacks** needed per module
- **Tight deadline/POC** - traditional monolith

---

## Benefits vs Tradeoffs

| Benefit | Tradeoff |
|---------|----------|
| ✅ Simpler than microservices | ❌ Can't scale modules independently |
| ✅ Clear boundaries | ❌ Still single deployment unit |
| ✅ Easy refactoring | ❌ Can grow into big monolith |
| ✅ Single transaction | ❌ Team coordination needed |
| ✅ Fast inter-module calls | ❌ Tech stack locked |

---

## Summary

**Modular Monolith = Best of Both Worlds:**
- ✓ Single deployable unit (simple ops)
- ✓ Clear module boundaries (maintainable)
- ✓ Interface-based communication (decoupled)
- ✓ Path to microservices (extractable)

**Start here, extract to microservices only when needed.**