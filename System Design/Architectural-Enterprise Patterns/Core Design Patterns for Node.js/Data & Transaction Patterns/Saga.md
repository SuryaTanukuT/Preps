# Saga Pattern

## Definition
**Saga** is a design pattern for managing distributed transactions across multiple microservices. Instead of using a single ACID transaction (which doesn't scale across services), a saga breaks the transaction into a series of local transactions, each with a compensating transaction that can undo its effects if something fails.

A saga ensures **data consistency** in distributed systems through **eventual consistency** and **compensating actions**.

---

## Core Concept
```
                    SUCCESS FLOW
    ┌─────────────────────────────────────────────┐
    │  Service A    Service B    Service C        │
    │  ┌──────┐     ┌──────┐     ┌──────┐        │
    │  │  T1  │────▶│  T2  │────▶│  T3  │────┐   │
    │  └──────┘     └──────┘     └──────┘    │   │
    │                                         ▼   │
    │                                      SUCCESS│
    └─────────────────────────────────────────────┘

                    FAILURE FLOW (Compensation)
    ┌─────────────────────────────────────────────┐
    │  Service A    Service B    Service C        │
    │  ┌──────┐     ┌──────┐     ┌──────┐        │
    │  │  T1  │────▶│  T2  │────▶│  T3  │───X    │
    │  └──┬───┘     └──┬───┘     └──────┘        │
    │     │            │                          │
    │     │◀───────────│                          │
    │  ┌──▼───┐     ┌──▼───┐                      │
    │  │  C1  │◀────│  C2  │                      │
    │  └──────┘     └──────┘                      │
    │                                             │
    │                                    FAILED   │
    └─────────────────────────────────────────────┘
    
    T1, T2, T3 = Transactions
    C1, C2 = Compensating Transactions
```

---

## Types of Sagas

### 1. Choreography-based Saga
Services communicate via events. Each service performs its transaction and publishes events that trigger other services.

### 2. Orchestration-based Saga
A central coordinator (orchestrator) tells each service what to do and manages compensation on failure.

---

## Comparison: Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
|--------|--------------|---------------|
| **Coordination** | Decentralized (events) | Centralized (orchestrator) |
| **Coupling** | Loose (via events) | Tighter (via orchestrator) |
| **Complexity** | Harder to trace | Easier to manage |
| **Single Point of Failure** | No | Yes (orchestrator) |
| **Best For** | Simple workflows, fewer services | Complex workflows, many services |
| **Monitoring** | Difficult | Easy (orchestrator logs) |
| **Changes** | Affect multiple services | Isolated to orchestrator |

---

## 1. Choreography-based Saga

### Definition
Services communicate through events. Each service listens for events, performs its local transaction, and emits events for the next step. There's no central controller.

### Core Concept
```
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │  Order   │      │ Payment  │      │Inventory │
    │ Service  │      │ Service  │      │ Service  │
    └────┬─────┘      └────┬─────┘      └────┬─────┘
         │                 │                 │
         │ OrderCreated    │                 │
         │────────────────>│                 │
         │                 │ PaymentProcessed│
         │                 │────────────────>│
         │                 │                 │ InventoryReserved
         │                 │                 │─────────┐
         │                 │                 │         │
         │         OrderConfirmed             │         │
         │<───────────────────────────────────│         │
         │                                   ▼         ▼
         │                                SUCCESS
```

### Example: E-commerce Order Processing

```javascript
// Event Bus (RabbitMQ/Kafka)
const eventBus = {
  publish(event) {
    console.log(`Event published: ${event.type}`);
    // In real implementation, publish to message broker
  },
  
  subscribe(eventType, handler) {
    console.log(`Subscribed to: ${eventType}`);
    // In real implementation, subscribe to queue
  }
};

// ============ ORDER SERVICE ============
class OrderService {
  constructor() {
    // Listen for events from other services
    eventBus.subscribe('payment.processed', this.handlePaymentProcessed.bind(this));
    eventBus.subscribe('inventory.reserved', this.handleInventoryReserved.bind(this));
    eventBus.subscribe('payment.failed', this.handlePaymentFailed.bind(this));
    eventBus.subscribe('inventory.failed', this.handleInventoryFailed.bind(this));
  }
  
  async createOrder(orderData) {
    console.log(`[Order] Creating order: ${orderData.orderId}`);
    
    // 1. Create order in PENDING state
    const order = {
      id: orderData.orderId,
      userId: orderData.userId,
      items: orderData.items,
      total: orderData.total,
      status: 'PENDING'
    };
    
    await this.saveOrder(order);
    
    // 2. Publish event to start saga
    eventBus.publish({
      type: 'order.created',
      data: {
        orderId: order.id,
        userId: order.userId,
        amount: order.total,
        items: order.items
      },
      timestamp: new Date()
    });
    
    return order;
  }
  
  async handlePaymentProcessed(event) {
    console.log(`[Order] Payment processed for order: ${event.data.orderId}`);
    
    // Update order status
    await this.updateOrderStatus(event.data.orderId, 'PAYMENT_COMPLETED');
    
    // Check if inventory already reserved
    const order = await this.getOrder(event.data.orderId);
    if (order.inventoryReserved) {
      await this.completeOrder(event.data.orderId);
    }
  }
  
  async handleInventoryReserved(event) {
    console.log(`[Order] Inventory reserved for order: ${event.data.orderId}`);
    
    // Update order status
    await this.updateOrderInventoryFlag(event.data.orderId, true);
    
    // Check if payment already processed
    const order = await this.getOrder(event.data.orderId);
    if (order.status === 'PAYMENT_COMPLETED') {
      await this.completeOrder(event.data.orderId);
    }
  }
  
  async handlePaymentFailed(event) {
    console.log(`[Order] Payment failed for order: ${event.data.orderId}`);
    
    // Cancel order (no compensation needed - nothing happened yet)
    await this.cancelOrder(event.data.orderId, 'PAYMENT_FAILED');
  }
  
  async handleInventoryFailed(event) {
    console.log(`[Order] Inventory reservation failed for order: ${event.data.orderId}`);
    
    // Payment already succeeded? Need to refund!
    const order = await this.getOrder(event.data.orderId);
    
    if (order.status === 'PAYMENT_COMPLETED') {
      // Trigger payment refund
      eventBus.publish({
        type: 'order.refund.requested',
        data: {
          orderId: event.data.orderId,
          amount: order.total,
          reason: 'INVENTORY_UNAVAILABLE'
        },
        timestamp: new Date()
      });
    } else {
      // Just cancel
      await this.cancelOrder(event.data.orderId, 'INVENTORY_FAILED');
    }
  }
  
  async completeOrder(orderId) {
    console.log(`[Order] Order completed: ${orderId}`);
    
    await this.updateOrderStatus(orderId, 'COMPLETED');
    
    eventBus.publish({
      type: 'order.completed',
      data: { orderId },
      timestamp: new Date()
    });
  }
}

// ============ PAYMENT SERVICE ============
class PaymentService {
  constructor() {
    eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
    eventBus.subscribe('order.refund.requested', this.handleRefundRequested.bind(this));
  }
  
  async handleOrderCreated(event) {
    console.log(`[Payment] Processing payment for order: ${event.data.orderId}`);
    
    try {
      // Process payment (call payment gateway)
      const paymentResult = await this.processPayment({
        userId: event.data.userId,
        amount: event.data.amount,
        orderId: event.data.orderId
      });
      
      // Success event
      eventBus.publish({
        type: 'payment.processed',
        data: {
          orderId: event.data.orderId,
          paymentId: paymentResult.id,
          amount: event.data.amount
        },
        timestamp: new Date()
      });
    } catch (error) {
      // Failure event
      eventBus.publish({
        type: 'payment.failed',
        data: {
          orderId: event.data.orderId,
          reason: error.message
        },
        timestamp: new Date()
      });
    }
  }
  
  async handleRefundRequested(event) {
    console.log(`[Payment] Processing refund for order: ${event.data.orderId}`);
    
    // Process refund
    await this.processRefund(event.data.orderId, event.data.amount);
    
    eventBus.publish({
      type: 'payment.refunded',
      data: {
        orderId: event.data.orderId,
        amount: event.data.amount
      },
      timestamp: new Date()
    });
  }
}

// ============ INVENTORY SERVICE ============
class InventoryService {
  constructor() {
    eventBus.subscribe('order.created', this.handleOrderCreated.bind(this));
    eventBus.subscribe('payment.refunded', this.handlePaymentRefunded.bind(this));
  }
  
  async handleOrderCreated(event) {
    console.log(`[Inventory] Reserving inventory for order: ${event.data.orderId}`);
    
    try {
      // Check and reserve inventory
      for (const item of event.data.items) {
        await this.reserveItem(item.productId, item.quantity, event.data.orderId);
      }
      
      // Success event
      eventBus.publish({
        type: 'inventory.reserved',
        data: {
          orderId: event.data.orderId,
          items: event.data.items
        },
        timestamp: new Date()
      });
    } catch (error) {
      // Failure event
      eventBus.publish({
        type: 'inventory.failed',
        data: {
          orderId: event.data.orderId,
          reason: error.message
        },
        timestamp: new Date()
      });
    }
  }
  
  async handlePaymentRefunded(event) {
    console.log(`[Inventory] Releasing inventory for cancelled order: ${event.data.orderId}`);
    
    // Release reserved inventory
    await this.releaseInventory(event.data.orderId);
  }
}

// Usage
const orderService = new OrderService();
const paymentService = new PaymentService();
const inventoryService = new InventoryService();

// Start a saga
orderService.createOrder({
  orderId: 'order-123',
  userId: 'user-456',
  items: [
    { productId: 'p1', quantity: 2, price: 50 }
  ],
  total: 100
});
```

**Choreography Flow - Success:**
1. Order Service: `order.created` event
2. Payment Service: Processes → `payment.processed` event
3. Inventory Service: Reserves → `inventory.reserved` event
4. Order Service: Completes → `order.completed` event

**Choreography Flow - Failure (Inventory fails after payment):**
1. Order Service: `order.created` event
2. Payment Service: Success → `payment.processed` event
3. Inventory Service: Fails → `inventory.failed` event
4. Order Service: Requests refund → `order.refund.requested` event
5. Payment Service: Refunds → `payment.refunded` event
6. Inventory Service: (Already failed, no action needed)

---

## 2. Orchestration-based Saga

### Definition
A central orchestrator tells each service what to do. The orchestrator maintains the saga state, executes steps in order, and triggers compensating transactions if any step fails.

### Core Concept
```
                    ┌─────────────────┐
                    │   ORCHESTRATOR  │
                    │   (Saga Manager)│
                    └─────┬─────┬─────┘
                          │     │
              ┌───────────┘     └───────────┐
              │                             │
    ┌─────────▼─────────┐         ┌─────────▼─────────┐
    │   Order Service   │         │  Payment Service  │
    │  "Create Order"   │         │ "Process Payment" │
    └─────────┬─────────┘         └─────────┬─────────┘
              │                             │
              └───────────┐     ┌───────────┘
                          │     │
                    ┌─────▼─────▼─────┐
                    │ Inventory Service│
                    │ "Reserve Items"  │
                    └──────────────────┘
```

### Example: Travel Booking Saga

```javascript
// ============ SAGA ORCHESTRATOR ============
class TravelBookingOrchestrator {
  constructor() {
    this.sagas = new Map(); // Store saga states
  }
  
  async bookTrip(bookingData) {
    const sagaId = this.generateSagaId();
    const saga = {
      id: sagaId,
      status: 'STARTED',
      data: bookingData,
      steps: [],
      compensationStack: []
    };
    
    this.sagas.set(sagaId, saga);
    
    // Start saga asynchronously
    this.executeSaga(saga).catch(error => {
      this.compensateSaga(saga, error);
    });
    
    return { sagaId, status: 'PROCESSING' };
  }
  
  async executeSaga(saga) {
    console.log(`[Saga ${saga.id}] Starting travel booking`);
    
    // Step 1: Book Flight
    try {
      const flightResult = await this.bookFlight(saga.data.flight);
      saga.steps.push({ name: 'FLIGHT_BOOKED', result: flightResult });
      saga.compensationStack.unshift('CANCEL_FLIGHT');
      console.log(`[Saga ${saga.id}] Flight booked: ${flightResult.confirmation}`);
    } catch (error) {
      throw new Error(`Flight booking failed: ${error.message}`);
    }
    
    // Step 2: Book Hotel
    try {
      const hotelResult = await this.bookHotel(saga.data.hotel);
      saga.steps.push({ name: 'HOTEL_BOOKED', result: hotelResult });
      saga.compensationStack.unshift('CANCEL_HOTEL');
      console.log(`[Saga ${saga.id}] Hotel booked: ${hotelResult.confirmation}`);
    } catch (error) {
      throw new Error(`Hotel booking failed: ${error.message}`);
    }
    
    // Step 3: Book Car Rental
    try {
      const carResult = await this.bookCar(saga.data.car);
      saga.steps.push({ name: 'CAR_BOOKED', result: carResult });
      saga.compensationStack.unshift('CANCEL_CAR');
      console.log(`[Saga ${saga.id}] Car booked: ${carResult.confirmation}`);
    } catch (error) {
      throw new Error(`Car booking failed: ${error.message}`);
    }
    
    // Step 4: Process Payment
    try {
      const paymentResult = await this.processPayment({
        amount: saga.data.totalPrice,
        bookingRef: saga.id
      });
      saga.steps.push({ name: 'PAYMENT_PROCESSED', result: paymentResult });
      saga.compensationStack.unshift('REFUND_PAYMENT');
      console.log(`[Saga ${saga.id}] Payment processed: ${paymentResult.transactionId}`);
    } catch (error) {
      throw new Error(`Payment failed: ${error.message}`);
    }
    
    // All steps succeeded
    saga.status = 'COMPLETED';
    console.log(`[Saga ${saga.id}] Travel booking completed successfully!`);
    
    // Send confirmation
    await this.sendConfirmation(saga);
    
    return saga;
  }
  
  async compensateSaga(saga, error) {
    console.log(`[Saga ${saga.id}] Compensating due to: ${error.message}`);
    
    saga.status = 'COMPENSATING';
    
    // Execute compensations in reverse order
    for (const step of saga.compensationStack) {
      try {
        switch(step) {
          case 'CANCEL_FLIGHT':
            await this.cancelFlight(saga.steps.find(s => s.name === 'FLIGHT_BOOKED').result);
            console.log(`[Saga ${saga.id}] Flight cancelled`);
            break;
            
          case 'CANCEL_HOTEL':
            await this.cancelHotel(saga.steps.find(s => s.name === 'HOTEL_BOOKED').result);
            console.log(`[Saga ${saga.id}] Hotel cancelled`);
            break;
            
          case 'CANCEL_CAR':
            await this.cancelCar(saga.steps.find(s => s.name === 'CAR_BOOKED').result);
            console.log(`[Saga ${saga.id}] Car cancelled`);
            break;
            
          case 'REFUND_PAYMENT':
            await this.refundPayment(saga.steps.find(s => s.name === 'PAYMENT_PROCESSED').result);
            console.log(`[Saga ${saga.id}] Payment refunded`);
            break;
        }
      } catch (compError) {
        console.error(`[Saga ${saga.id}] Compensation failed for ${step}:`, compError);
        // Log for manual intervention
        await this.logManualInterventionRequired(saga.id, step, compError);
      }
    }
    
    saga.status = 'COMPENSATED';
    console.log(`[Saga ${saga.id}] Saga compensated`);
  }
  
  // External service calls
  async bookFlight(flightData) {
    // Call flight booking API
    return { confirmation: `FL-${Date.now()}` };
  }
  
  async bookHotel(hotelData) {
    // Call hotel booking API
    return { confirmation: `HT-${Date.now()}` };
  }
  
  async bookCar(carData) {
    // Call car rental API
    return { confirmation: `CR-${Date.now()}` };
  }
  
  async processPayment(paymentData) {
    // Call payment API
    return { transactionId: `PAY-${Date.now()}` };
  }
  
  // Compensation methods
  async cancelFlight(flightResult) {
    // Call cancellation API
  }
  
  async cancelHotel(hotelResult) {
    // Call cancellation API
  }
  
  async cancelCar(carResult) {
    // Call cancellation API
  }
  
  async refundPayment(paymentResult) {
    // Call refund API
  }
  
  generateSagaId() {
    return `saga-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }
}

// ============ STATE MANAGEMENT ============
class SagaStateStore {
  constructor() {
    // In production, use database
    this.store = new Map();
  }
  
  async save(sagaId, state) {
    this.store.set(sagaId, {
      ...state,
      updatedAt: new Date()
    });
  }
  
  async get(sagaId) {
    return this.store.get(sagaId);
  }
  
  async updateStatus(sagaId, status) {
    const saga = await this.get(sagaId);
    if (saga) {
      saga.status = status;
      saga.updatedAt = new Date();
      await this.save(sagaId, saga);
    }
  }
}

// ============ API ENDPOINT ============
app.post('/api/travel/book', async (req, res) => {
  const orchestrator = new TravelBookingOrchestrator();
  
  const { sagaId, status } = await orchestrator.bookTrip(req.body);
  
  res.json({
    sagaId,
    status,
    statusUrl: `/api/travel/status/${sagaId}`
  });
});

app.get('/api/travel/status/:sagaId', async (req, res) => {
  const saga = orchestrator.sagas.get(req.params.sagaId);
  
  if (!saga) {
    return res.status(404).json({ error: 'Saga not found' });
  }
  
  res.json({
    sagaId: saga.id,
    status: saga.status,
    steps: saga.steps,
    data: saga.data
  });
});
```

**Orchestration Flow - Success:**
```
Orchestrator → Flight Service: Book flight
Orchestrator ← Flight Service: Flight booked
Orchestrator → Hotel Service: Book hotel
Orchestrator ← Hotel Service: Hotel booked
Orchestrator → Car Service: Book car
Orchestrator ← Car Service: Car booked
Orchestrator → Payment Service: Process payment
Orchestrator ← Payment Service: Payment processed
Orchestrator: Saga COMPLETED
```

**Orchestration Flow - Failure (Hotel fails):**
```
Orchestrator → Flight Service: Book flight (SUCCESS)
Orchestrator → Hotel Service: Book hotel (FAILS)
Orchestrator → Flight Service: Cancel flight (Compensation)
Orchestrator: Saga COMPENSATED
```

---

## Industry Use Cases

### 1. BFSI - Money Transfer Between Banks

```javascript
// CHOREOGRAPHY-BASED SAGA for inter-bank transfer
class TransferSaga {
  async transferMoney(command) {
    const { fromAccount, toAccount, amount, transferId } = command;
    
    // Step 1: Debit from source bank
    eventBus.publish({
      type: 'debit.requested',
      data: { accountId: fromAccount, amount, transferId }
    });
  }
}

// Source Bank Service
class SourceBankService {
  constructor() {
    eventBus.subscribe('debit.requested', this.handleDebit.bind(this));
    eventBus.subscribe('credit.failed', this.handleCreditFailed.bind(this));
  }
  
  async handleDebit(event) {
    try {
      // Debit account
      await this.debitAccount(event.data.accountId, event.data.amount);
      
      eventBus.publish({
        type: 'debited',
        data: { 
          transferId: event.data.transferId,
          fromAccount: event.data.accountId,
          amount: event.data.amount
        }
      });
    } catch (error) {
      eventBus.publish({
        type: 'debit.failed',
        data: { 
          transferId: event.data.transferId,
          reason: error.message 
        }
      });
    }
  }
  
  async handleCreditFailed(event) {
    // Compensation: Reverse the debit
    await this.reverseDebit(
      event.data.fromAccount, 
      event.data.amount,
      event.data.transferId
    );
    
    eventBus.publish({
      type: 'debit.reversed',
      data: { transferId: event.data.transferId }
    });
  }
}

// Destination Bank Service
class DestinationBankService {
  constructor() {
    eventBus.subscribe('debited', this.handleDebited.bind(this));
    eventBus.subscribe('debit.reversed', this.handleDebitReversed.bind(this));
  }
  
  async handleDebited(event) {
    try {
      // Credit account
      await this.creditAccount(
        event.data.toAccount, 
        event.data.amount,
        event.data.transferId
      );
      
      eventBus.publish({
        type: 'transfer.completed',
        data: { transferId: event.data.transferId }
      });
    } catch (error) {
      // Can't credit - request compensation
      eventBus.publish({
        type: 'credit.failed',
        data: {
          transferId: event.data.transferId,
          fromAccount: event.data.fromAccount,
          amount: event.data.amount,
          reason: error.message
        }
      });
    }
  }
  
  async handleDebitReversed(event) {
    // Transfer failed completely
    eventBus.publish({
      type: 'transfer.failed',
      data: { transferId: event.data.transferId }
    });
  }
}
```

### 2. Real Estate - Property Purchase

```javascript
// ORCHESTRATION-BASED SAGA for property purchase
class PropertyPurchaseOrchestrator {
  async purchaseProperty(command) {
    const sagaId = uuid();
    
    try {
      // Step 1: Verify funds
      const fundsVerified = await this.verifyBuyerFunds(
        command.buyerId, 
        command.offerAmount
      );
      
      // Step 2: Place deposit in escrow
      const escrowId = await this.createEscrow(
        command.buyerId,
        command.sellerId,
        command.depositAmount
      );
      
      // Step 3: Schedule inspection
      const inspectionId = await this.scheduleInspection(
        command.propertyId,
        command.inspectionDate
      );
      
      // Step 4: Update property status to PENDING
      await this.updatePropertyStatus(command.propertyId, 'PENDING');
      
      // Step 5: Notify all parties
      await this.sendNotifications({
        buyer: command.buyerId,
        seller: command.sellerId,
        agent: command.agentId,
        message: 'Offer accepted, inspection scheduled'
      });
      
      return { success: true, sagaId };
      
    } catch (error) {
      // Compensate in reverse order
      if (inspectionId) await this.cancelInspection(inspectionId);
      if (escrowId) await this.releaseEscrow(escrowId);
      if (fundsVerified) await this.releaseFundsHold(command.buyerId, command.offerAmount);
      
      throw error;
    }
  }
}
```

### 3. E-commerce - Order Fulfillment

```javascript
// HYBRID SAGA (Orchestration + Choreography)
class OrderFulfillmentSaga {
  constructor() {
    this.orchestrator = new SagaOrchestrator();
    this.setupSteps();
  }
  
  setupSteps() {
    // Define saga steps
    this.orchestrator
      .step('RESERVE_INVENTORY')
        .invoke(this.reserveInventory)
        .withCompensation(this.releaseInventory)
      .step('PROCESS_PAYMENT')
        .invoke(this.processPayment)
        .withCompensation(this.refundPayment)
      .step('CREATE_SHIPMENT')
        .invoke(this.createShipment)
        .withCompensation(this.cancelShipment)
      .step('UPDATE_LOYALTY_POINTS')
        .invoke(this.updateLoyaltyPoints)
        // No compensation needed for loyalty points
      .step('SEND_CONFIRMATION')
        .invoke(this.sendEmails);
        // No compensation needed
  }
  
  async execute(order) {
    const sagaId = await this.orchestrator.start(order);
    
    // Return immediately, process async
    return { 
      orderId: order.id, 
      status: 'PROCESSING',
      trackingUrl: `/api/orders/${order.id}/status`
    };
  }
  
  // Step implementations
  async reserveInventory(order) {
    const inventoryService = new InventoryService();
    
    for (const item of order.items) {
      const available = await inventoryService.checkStock(
        item.productId, 
        item.quantity
      );
      
      if (!available) {
        throw new Error(`Insufficient stock for ${item.productId}`);
      }
      
      await inventoryService.reserve(
        item.productId, 
        item.quantity, 
        order.id
      );
    }
    
    return { reserved: true };
  }
  
  async releaseInventory(order, result) {
    const inventoryService = new InventoryService();
    
    for (const item of order.items) {
      await inventoryService.releaseReservation(
        item.productId, 
        item.quantity, 
        order.id
      );
    }
  }
  
  async processPayment(order) {
    const paymentService = new PaymentService();
    
    const payment = await paymentService.charge({
      amount: order.total,
      paymentMethod: order.paymentMethod,
      orderId: order.id
    });
    
    return { paymentId: payment.id };
  }
  
  async refundPayment(order, result) {
    if (result?.paymentId) {
      const paymentService = new PaymentService();
      await paymentService.refund(result.paymentId);
    }
  }
  
  async createShipment(order) {
    const shippingService = new ShippingService();
    
    const shipment = await shippingService.createLabel({
      address: order.shippingAddress,
      items: order.items,
      orderId: order.id
    });
    
    return { 
      trackingNumber: shipment.trackingNumber,
      carrier: shipment.carrier
    };
  }
  
  async cancelShipment(order, result) {
    if (result?.trackingNumber) {
      const shippingService = new ShippingService();
      await shippingService.cancelLabel(result.trackingNumber);
    }
  }
  
  async updateLoyaltyPoints(order) {
    const loyaltyService = new LoyaltyService();
    await loyaltyService.addPoints(
      order.userId, 
      Math.floor(order.total / 10)
    );
  }
  
  async sendEmails(order) {
    const emailService = new EmailService();
    
    await emailService.sendOrderConfirmation(order.userId, order.id);
    await emailService.sendReceipt(order.userId, order.id, order.total);
  }
}

// State machine for saga orchestration
class SagaOrchestrator {
  constructor() {
    this.steps = [];
    this.currentStep = 0;
    this.sagaStore = new SagaStore();
  }
  
  step(name) {
    const step = new SagaStep(name);
    this.steps.push(step);
    return step;
  }
  
  async start(context) {
    const sagaId = uuid();
    const saga = {
      id: sagaId,
      status: 'STARTED',
      context,
      steps: [],
      currentStep: 0
    };
    
    await this.sagaStore.save(saga);
    
    // Execute asynchronously
    setImmediate(() => this.execute(saga));
    
    return sagaId;
  }
  
  async execute(saga) {
    try {
      for (let i = saga.currentStep; i < this.steps.length; i++) {
        const step = this.steps[i];
        const result = await step.invoke(saga.context);
        
        saga.steps.push({
          name: step.name,
          result,
          completedAt: new Date()
        });
        saga.currentStep = i + 1;
        
        await this.sagaStore.save(saga);
      }
      
      saga.status = 'COMPLETED';
      await this.sagaStore.save(saga);
      
    } catch (error) {
      await this.compensate(saga, error);
    }
  }
  
  async compensate(saga, error) {
    saga.status = 'COMPENSATING';
    saga.error = error.message;
    await this.sagaStore.save(saga);
    
    // Compensate in reverse order
    for (let i = saga.steps.length - 1; i >= 0; i--) {
      const step = saga.steps[i];
      const stepDef = this.steps.find(s => s.name === step.name);
      
      if (stepDef.compensation) {
        try {
          await stepDef.compensation(saga.context, step.result);
        } catch (compError) {
          // Log compensation failure for manual handling
          console.error(`Compensation failed for step ${step.name}:`, compError);
          await this.sagaStore.recordCompensationFailure(saga.id, step.name, compError);
        }
      }
    }
    
    saga.status = 'COMPENSATED';
    await this.sagaStore.save(saga);
  }
}

class SagaStep {
  constructor(name) {
    this.name = name;
    this.invokeFn = null;
    this.compensationFn = null;
  }
  
  invoke(fn) {
    this.invokeFn = fn;
    return this;
  }
  
  withCompensation(fn) {
    this.compensationFn = fn;
    return this;
  }
}
```

---

## Saga Implementation Patterns

### 1. State Machine Pattern

```javascript
class SagaStateMachine {
  constructor() {
    this.states = {
      PENDING: 'PENDING',
      RESERVING_INVENTORY: 'RESERVING_INVENTORY',
      PROCESSING_PAYMENT: 'PROCESSING_PAYMENT',
      CREATING_SHIPMENT: 'CREATING_SHIPMENT',
      COMPLETED: 'COMPLETED',
      COMPENSATING: 'COMPENSATING',
      COMPENSATED: 'COMPENSATED',
      FAILED: 'FAILED'
    };
    
    this.transitions = {
      [this.states.PENDING]: [this.states.RESERVING_INVENTORY],
      [this.states.RESERVING_INVENTORY]: [this.states.PROCESSING_PAYMENT, this.states.COMPENSATING],
      [this.states.PROCESSING_PAYMENT]: [this.states.CREATING_SHIPMENT, this.states.COMPENSATING],
      [this.states.CREATING_SHIPMENT]: [this.states.COMPLETED, this.states.COMPENSATING],
      [this.states.COMPENSATING]: [this.states.COMPENSATED, this.states.FAILED]
    };
  }
  
  async transition(saga, toState) {
    if (!this.transitions[saga.state]?.includes(toState)) {
      throw new Error(`Invalid transition from ${saga.state} to ${toState}`);
    }
    
    saga.state = toState;
    saga.updatedAt = new Date();
    
    await this.save(saga);
    
    return saga;
  }
}
```

### 2. Compensation Log Pattern

```javascript
class CompensationLogger {
  async logCompensation(sagaId, stepName, error) {
    await db.query(`
      INSERT INTO compensation_log 
      (saga_id, step_name, error, created_at)
      VALUES ($1, $2, $3, $4)
    `, [sagaId, stepName, error.message, new Date()]);
  }
  
  async getFailedCompensations() {
    return await db.query(`
      SELECT * FROM compensation_log 
      WHERE resolved = false
      ORDER BY created_at
    `);
  }
  
  async manualResolve(sagaId, stepName) {
    await db.query(`
      UPDATE compensation_log 
      SET resolved = true, resolved_at = NOW()
      WHERE saga_id = $1 AND step_name = $2
    `, [sagaId, stepName]);
  }
}
```

### 3. Idempotency Pattern

```javascript
class IdempotentSagaStep {
  async execute(stepId, operation) {
    // Check if already executed
    const existing = await redis.get(`step:${stepId}:result`);
    
    if (existing) {
      return JSON.parse(existing);
    }
    
    // Execute with lock to prevent duplicate
    const lock = await redis.setnx(`step:${stepId}:lock`, '1', 'EX', 30);
    
    if (!lock) {
      throw new Error('Step already in progress');
    }
    
    try {
      const result = await operation();
      
      // Store result with TTL
      await redis.setex(`step:${stepId}:result`, 86400, JSON.stringify(result));
      
      return result;
    } finally {
      await redis.del(`step:${stepId}:lock`);
    }
  }
}
```

---

## When to Use Each Saga Type

### Choreography ✅ Use When:
- Simple workflows with 2-3 services
- Services are loosely coupled
- You want decentralized control
- Event-driven architecture already in place
- Teams own their services independently

### Orchestration ✅ Use When:
- Complex workflows with many steps
- Business logic is centralized
- Need clear visibility and monitoring
- Error handling is complex
- Different teams own different services

---

## Saga vs Traditional Transaction

| Aspect | Traditional (2PC) | Saga |
|--------|------------------|------|
| **Consistency** | Strong (ACID) | Eventual |
| **Isolation** | Serializable | None (need semantic locks) |
| **Scope** | Single database | Multiple services |
| **Locking** | Locks resources | No long locks |
| **Performance** | Slower, blocking | Faster, non-blocking |
| **Failure Handling** | Rollback | Compensation |
| **Complexity** | Lower | Higher |
| **Scalability** | Limited | High |

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Missing Compensation** | Every step must have compensation |
| **Non-Idempotent Operations** | Make all operations idempotent |
| **No Isolation** | Use semantic locks (e.g., "reserved" status) |
| **Compensation Failure** | Log for manual intervention |
| **Orchestrator SPOF** | Cluster orchestrator, persistent state |
| **Cyclic Dependencies** | Design linear workflows |
| **Timeout Handling** | Set timeouts for all steps |

---

## Summary Checklist

| Consideration | Implementation |
|--------------|----------------|
| **Saga Type** | Choreography (simple) / Orchestration (complex) |
| **Step Definition** | Each step has forward action + compensation |
| **State Persistence** | Store saga state in database |
| **Idempotency** | Prevent duplicate execution |
| **Timeout** | Handle stuck sagas |
| **Monitoring** | Track saga status, failures |
| **Manual Recovery** | UI for failed compensations |
| **Isolation** | Semantic locks to prevent anomalies |

**Bottom Line:** Sagas are essential for distributed transactions. Choose choreography for simpler event-driven flows and orchestration for complex business processes. Always design compensating transactions and handle idempotency.