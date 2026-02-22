# Publish-Subscribe (Pub/Sub) Pattern

## Definition
**Publish-Subscribe** is a messaging pattern where senders (publishers) dispatch messages without being explicitly addressed to receivers (subscribers). Instead, messages are categorized into topics/ channels, and subscribers express interest in one or more topics, receiving only relevant messages without knowing about the publishers.

This decouples producers and consumers in time, space, and synchronization.

---

## Core Concept
```
                    ┌─────────────────┐
                    │   PUBLISHERS    │
                    └────────┬────────┘
                             │
                 ┌───────────▼───────────┐
                 │   MESSAGE BROKER      │
                 │  (Topics/Channels)    │
                 └───┬───────┬───────┬───┘
                     │       │       │
            ┌────────▼───┐ ┌─▼───────▼──┐ ┌──▼────────┐
            │ SUBSCRIBER │ │ SUBSCRIBER │ │ SUBSCRIBER│
            │  (Service) │ │  (Service) │ │ (Service) │
            └────────────┘ └────────────┘ └───────────┘
```

---

## Good vs Bad Example

### Good Example
**E-commerce Order Processing:**
```
Order Service (publishes) → "order.created" Topic
                             ├── Inventory Service (subscribes) → Update stock
                             ├── Payment Service (subscribes) → Process payment
                             ├── Email Service (subscribes) → Send confirmation
                             └── Analytics Service (subscribes) → Track event
```
- Order service doesn't know about other services
- New services can subscribe without changing order service
- If email service is down, order processing continues

### Bad Example
**Direct HTTP Calls:**
```
Order Service → POST /inventory/update
              → POST /payment/process  
              → POST /email/send
              → POST /analytics/track
```
- Order service coupled to all downstream services
- If any service fails, order processing fails
- Adding new service requires changing order service code
- Synchronous calls increase response time

---

## Industry Use Cases

### 1. BFSI (Banking)

**Topic Structure:**
```
banking/
├── transaction.created
├── transaction.flagged (fraud)
├── account.updated
├── loan.applied
├── payment.received
└── statement.generated
```

**Real-World Flow:**
```
ATM Network (Publisher) → "transaction.created" Topic
                           ├── Fraud Detection (sub) → Real-time analysis
                           ├── Ledger Service (sub) → Record transaction
                           ├── Notification Service (sub) → SMS alert
                           ├── Budget Service (sub) → Update spending
                           └── Compliance Service (sub) → Audit log
```

**Key Benefits for BFSI:**
- **Audit Trail:** Every event logged immutably
- **Compliance:** Regulators can subscribe to specific topics
- **Reconciliation:** Separate service can verify all transactions
- **Disaster Recovery:** Replay events to rebuild state

**Example Scenario: Suspicious Transaction**
```
1. Card transaction occurs → published to "transaction.created"
2. Fraud service detects pattern → publishes "transaction.flagged"
3. Hold service subscribes to flags → places temporary hold
4. SMS service sends alert → "Suspicious transaction?"
5. Customer responds → publishes "transaction.confirmed/rejected"
6. Hold service releases/cancels based on response
```

### 2. Real Estate

**Topic Structure:**
```
realestate/
├── property.listed
├── property.price.changed
├── property.sold
├── showing.scheduled
├── offer.submitted
├── offer.accepted
└── mls.updated
```

**Real-World Flow:**
```
Agent App (Publisher) → "property.listed" Topic
                         ├── Search Index (sub) → Update Elasticsearch
                         ├── Notification Hub (sub) → Alert interested buyers
                         ├── Image Processor (sub) → Generate thumbnails
                         ├── Virtual Tour (sub) → Create 3D walkthrough
                         └── CRM (sub) → Add to agent portfolio
```

**Key Benefits for Real Estate:**
- **MLS Integration:** Multiple MLS systems publish to central topics
- **Buyer Alerts:** Subscribers get real-time matching properties
- **Price Drop Tracking:** Services can monitor specific price change topics
- **Open House Events:** Schedule services subscribe to coordinate showings

**Example Scenario: Price Drop Alert**
```
1. Agent updates price → publishes "property.price.changed"
2. Price history service records change
3. Matcher service compares against buyer preferences
4. For matched buyers → notification service sends alert
5. Analytics service tracks price trend in neighborhood
```

### 3. E-commerce

**Topic Structure:**
```
ecommerce/
├── order.created
├── order.cancelled
├── order.shipped
├── payment.authorized
├── payment.captured
├── inventory.updated
├── cart.abandoned
├── product.viewed
└── review.submitted
```

**Real-World Flow:**
```
Checkout Service → "order.created" Topic
                    ├── Inventory (sub) → Reserve items
                    ├── Payment (sub) → Authorize payment
                    ├── Email (sub) → Send receipt
                    ├── Recommendations (sub) → Update ML model
                    ├── Shipping (sub) → Create label
                    └── Analytics (sub) → Track conversion
```

**Key Benefits for E-commerce:**
- **Flash Sales:** Handle massive spikes by queueing messages
- **Inventory Sync:** Keep all warehouses in sync across regions
- **Abandoned Cart:** Timer-based topics trigger recovery emails
- **Recommendations:** Real-time updates to personalization engine

**Example Scenario: Flash Sale**
```
1. 10,000 orders hit simultaneously → all published to "order.created"
2. Inventory service processes as fast as it can (doesn't lose messages)
3. Payment service has its own subscriber with rate limiting
4. Email service sends confirmations hours later—customer doesn't wait
5. Analytics captures every event without slowing checkout
```

---

## Popular Pub/Sub Technologies

| Technology | Type | Best For |
|------------|------|----------|
| **Apache Kafka** | Distributed log | High throughput, replay, BFSI |
| **RabbitMQ** | Message broker | Complex routing, AMQP |
| **Redis Pub/Sub** | In-memory | Real-time, low latency |
| **AWS SNS/SQS** | Managed cloud | Serverless, AWS ecosystem |
| **Google Pub/Sub** | Managed cloud | GCP, global scale |
| **NATS** | Lightweight | Cloud native, high performance |

---

## Node.js Implementation Examples

### 1. Basic Redis Pub/Sub
```javascript
// publisher.js - Order Service
const redis = require('redis');
const publisher = redis.createClient();

async function createOrder(orderData) {
  // Save to database
  const order = await db.orders.save(orderData);
  
  // Publish event
  await publisher.publish('order.created', JSON.stringify({
    orderId: order.id,
    userId: order.userId,
    amount: order.total,
    items: order.items.length,
    timestamp: new Date()
  }));
  
  return order;
}

// subscriber.js - Inventory Service
const redis = require('redis');
const subscriber = redis.createClient();

subscriber.subscribe('order.created', (message) => {
  const order = JSON.parse(message);
  
  console.log(`Processing inventory for order: ${order.orderId}`);
  
  // Update inventory asynchronously
  order.items.forEach(item => {
    db.inventory.decrement(item.productId, item.quantity);
  });
});
```

### 2. Kafka with Node.js
```javascript
// producer.js
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka1:9092', 'kafka2:9092']
});

const producer = kafka.producer();

async function emitOrderEvent(order) {
  await producer.connect();
  
  await producer.send({
    topic: 'order.events',
    messages: [
      { 
        key: order.userId, // Ensure order ordering per user
        value: JSON.stringify(order),
        headers: { 
          'event-type': 'ORDER_CREATED',
          'version': '1.0'
        }
      }
    ]
  });
  
  await producer.disconnect();
}

// consumer.js - Multiple instances in a group
const consumer = kafka.consumer({ groupId: 'inventory-group' });

await consumer.connect();
await consumer.subscribe({ topic: 'order.events', fromBeginning: false });

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const order = JSON.parse(message.value.toString());
    const eventType = message.headers['event-type'].toString();
    
    if (eventType === 'ORDER_CREATED') {
      await inventoryService.reserveItems(order);
    }
  }
});
```

### 3. RabbitMQ with Node.js
```javascript
// publisher.js
const amqp = require('amqplib');

async function publishOrderCreated(order) {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  const exchange = 'order.exchange';
  await channel.assertExchange(exchange, 'topic', { durable: true });
  
  channel.publish(
    exchange, 
    'order.created', // routing key
    Buffer.from(JSON.stringify(order)),
    { persistent: true } // messages survive broker restart
  );
  
  setTimeout(() => connection.close(), 500);
}

// subscriber.js - Email Service
async function subscribeEmailService() {
  const connection = await amqp.connect('amqp://localhost');
  const channel = await connection.createChannel();
  
  const exchange = 'order.exchange';
  await channel.assertExchange(exchange, 'topic', { durable: true });
  
  const queue = await channel.assertQueue('email-service-queue', { durable: true });
  
  // Bind to multiple topics
  await channel.bindQueue(queue.queue, exchange, 'order.created');
  await channel.bindQueue(queue.queue, exchange, 'order.shipped');
  await channel.bindQueue(queue.queue, exchange, 'order.cancelled');
  
  channel.consume(queue.queue, (msg) => {
    const order = JSON.parse(msg.content.toString());
    const routingKey = msg.fields.routingKey;
    
    switch(routingKey) {
      case 'order.created':
        emailService.sendOrderConfirmation(order);
        break;
      case 'order.shipped':
        emailService.sendShippingUpdate(order);
        break;
      case 'order.cancelled':
        emailService.sendCancellationNotice(order);
        break;
    }
    
    channel.ack(msg); // acknowledge processing
  });
}
```

### 4. AWS SNS/SQS with Node.js
```javascript
// publisher.js (SNS)
const AWS = require('aws-sdk');
const sns = new AWS.SNS();

async function publishOrderEvent(order) {
  const params = {
    TopicArn: 'arn:aws:sns:us-east-1:123456789012:order-topic',
    Message: JSON.stringify(order),
    MessageAttributes: {
      eventType: {
        DataType: 'String',
        StringValue: 'ORDER_CREATED'
      }
    }
  };
  
  await sns.publish(params).promise();
}

// subscriber.js (SQS)
const sqs = new AWS.SQS();

async function pollQueue() {
  const params = {
    QueueUrl: 'https://sqs.us-east-1.amazonaws.com/123456789012/inventory-queue',
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 20 // long polling
  };
  
  while(true) {
    const result = await sqs.receiveMessage(params).promise();
    
    if (result.Messages) {
      for (const message of result.Messages) {
        const order = JSON.parse(message.Body);
        
        await inventoryService.processOrder(order);
        
        // Delete after processing
        await sqs.deleteMessage({
          QueueUrl: params.QueueUrl,
          ReceiptHandle: message.ReceiptHandle
        }).promise();
      }
    }
  }
}
```

---

## Advanced Patterns

### 1. Dead Letter Queue (DLQ)
```javascript
// Messages that fail processing go to DLQ
channel.consume(queue, async (msg) => {
  try {
    await processMessage(msg);
    channel.ack(msg);
  } catch (error) {
    if (msg.properties.headers['x-retry-count'] >= 3) {
      // Send to DLQ after 3 retries
      channel.sendToQueue('dlq', msg.content, {
        headers: { originalQueue: queue, error: error.message }
      });
      channel.ack(msg);
    } else {
      // Retry with exponential backoff
      const retryCount = (msg.properties.headers['x-retry-count'] || 0) + 1;
      const delay = Math.pow(2, retryCount) * 1000;
      
      setTimeout(() => {
        channel.nack(msg, false, false); // re-queue
      }, delay);
    }
  }
});
```

### 2. Event Sourcing
```javascript
// Store all events as source of truth
const eventStore = [];

async function handleCommand(command) {
  // Validate command
  const events = executeCommand(command, currentState);
  
  // Store events
  for (const event of events) {
    await eventStore.append('order.aggregate', event);
    await publisher.publish(event.type, event);
  }
  
  // Rebuild state by replaying events
  const newState = events.reduce(reducer, currentState);
}
```

### 3. Competing Consumers
```javascript
// Multiple instances of same service process messages
// Kafka consumer group - partitions distributed automatically
const consumer = kafka.consumer({ groupId: 'payment-processors' });

// Each instance gets different partitions
// If one instance dies, partitions rebalanced

// Manual partitioning for SQS
async function processQueue() {
  // SQS handles distribution automatically
  // Multiple instances poll same queue
  // Messages processed once (visibility timeout)
  const messages = await sqs.receiveMessage(params);
  
  // Process in parallel with concurrency control
  await Promise.all(
    messages.map(msg => processWithSemaphore(msg))
  );
}
```

### 4. Message Schema Evolution
```javascript
// Handle different versions of messages
consumer.on('message', (msg) => {
  const version = msg.headers['schema-version'] || '1.0';
  
  switch(version) {
    case '1.0':
      processV1(JSON.parse(msg.value));
      break;
    case '2.0':
      processV2(JSON.parse(msg.value));
      break;
    default:
      // Transform to latest version
      const upgraded = upgradeSchema(msg.value, version);
      processV2(upgraded);
  }
});
```

---

## Best Practices

### 1. Message Design
```javascript
{
  "id": "msg_123456789",        // Unique ID for deduplication
  "type": "order.created",       // Event type
  "version": "1.0",              // Schema version
  "timestamp": "2024-01-15T10:30:00Z", // ISO timestamp
  "source": "order-service-v2",  // Publisher identifier
  "correlationId": "order_789",  // For tracing
  "userId": "user_456",          // Tenant/User context
  "data": {                       // Actual payload
    "orderId": "order_789",
    "amount": 99.99,
    "items": [...]
  }
}
```

### 2. Error Handling
```javascript
// Always handle errors in subscribers
async function safeProcess(message) {
  try {
    await processMessage(message);
    await acknowledge(message);
  } catch (error) {
    // Log with context
    logger.error('Processing failed', { 
      messageId: message.id,
      error: error.message,
      stack: error.stack
    });
    
    // Retry logic
    if (message.retryCount < 3) {
      await requeueWithDelay(message, Math.pow(2, message.retryCount) * 1000);
    } else {
      await sendToDeadLetter(message, error);
    }
  }
}
```

### 3. Monitoring
```javascript
// Track metrics for each topic
const metrics = {
  published: counter('messages.published'),
  consumed: counter('messages.consumed'),
  failed: counter('messages.failed'),
  latency: histogram('message.processing.latency')
};

// Health check for subscribers
app.get('/health/subscribers', (req, res) => {
  const status = {
    kafka: kafkaConsumer.isConnected(),
    rabbit: rabbitChannel.isOpen(),
    backlog: getQueueDepth(),
    lastProcessed: getLastProcessedTimestamp()
  };
  
  res.json(status);
});
```

### 4. Idempotency
```javascript
// Ensure messages processed only once
const processedMessages = new Set();

async function handleMessage(message) {
  const messageId = message.id || message.headers['message-id'];
  
  // Check if already processed (use Redis for distributed)
  if (await redis.sismember('processed', messageId)) {
    logger.info(`Duplicate message ignored: ${messageId}`);
    return ack(message); // Still acknowledge
  }
  
  // Process
  await process(message);
  
  // Mark as processed
  await redis.sadd('processed', messageId);
  await redis.expire('processed', 86400); // 24h TTL
  
  ack(message);
}
```

---

## When to Use Pub/Sub

### ✅ DO Use When:
- Decoupling microservices
- Event-driven architectures
- Broadcasting to multiple consumers
- Asynchronous processing
- Real-time notifications
- Audit logging and compliance
- Building reactive systems

### ❌ DON'T Use When:
- Request-response needed (use HTTP/RPC)
- Transactional consistency required
- Low latency (<10ms) critical path
- Simple point-to-point communication (use queue)
- Small application with 2-3 services

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Message Ordering** | Use partition keys, Kafka for ordered topics |
| **Duplicate Messages** | Idempotent consumers, deduplication IDs |
| **Poison Messages** | DLQ + monitoring, max retry attempts |
| **Backpressure** | Consumer prefetch limits, rate limiting |
| **Message Expiry** | Set TTL, handle late messages gracefully |
| **Schema Changes** | Version fields, schema registry |
| **Debugging** | Correlation IDs, distributed tracing |

---

## Summary Checklist

| Consideration | Implementation |
|--------------|----------------|
| **Decoupling** | Publishers don't know subscribers |
| **Scalability** | Add consumers independently |
| **Reliability** | Persistent messages, acks |
| **Ordering** | Partition keys if needed |
| **Exactly-once** | Idempotent + deduplication |
| **Monitoring** | Queue depth, processing latency |
| **Error Handling** | Retries, DLQ, alerts |
| **Schema** | Versioning, compatibility |

**Bottom Line:** Pub/Sub is the backbone of event-driven architectures, enabling loose coupling and massive scalability. Choose the right broker based on your ordering, persistence, and throughput needs.