https://codefarm0.medium.com/dynamodb-interview-questions-answers-categorized-by-topic-49a0a51eec6a
https://www.hellointerview.com/learn/system-design/deep-dives/dynamodb

---

# 🏦 DynamoDB — Real Production Use Cases (Industry Level)

---

# 🏦 1️⃣ DynamoDB in BFSI / FinTech

In serious banking systems (ENBD, QNB, ADCB style):

> 🔴 Core Ledger → Usually Aurora / PostgreSQL
> 🟢 High-scale state, idempotency, events → DynamoDB

---

## 🔹 BFSI Architecture Positioning

![Image](https://miro.medium.com/1%2A0B9k9qUUL3r5CLEXfu3-bg.jpeg)

![Image](https://ds0xrsm6llh5h.cloudfront.net/blogs/image_3a9e9a93-97f4-4769-b0b3-ac1444b09c6c_20250218094402.png)

![Image](https://substackcdn.com/image/fetch/%24s_%21Pk7N%21%2Cf_auto%2Cq_auto%3Agood%2Cfl_progressive%3Asteep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F3dd04d59-7ea9-487c-911b-3ccc225e7b9a_1600x944.png)

![Image](https://substackcdn.com/image/fetch/%24s_%21n7fG%21%2Cf_auto%2Cq_auto%3Agood%2Cfl_progressive%3Asteep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fb2559302-f9cd-46a4-8681-fb09dfe71cb0_1600x946.png)

DynamoDB typically sits in:

* Idempotency layer
* Session store
* Fraud events
* Audit logs
* Read models
* Token vaults

---

## 🔹 Use Case 1: Payment Idempotency Store

### 🎯 Problem

Retries from:

* Mobile apps
* Network failures
* Gateway timeouts

Risk → Double debit.

### 🏗 Design

```
PK = IDEMPOTENCY#<uuid>
SK = METADATA

status
response
ttl
createdAt
```

Write:

```
ConditionExpression: attribute_not_exists(PK)
```

### Why DynamoDB?

* Atomic conditional writes
* Strong consistency
* 1-digit millisecond latency
* Scales horizontally

---

### 🔥 Interview Q

**Q: How do you prevent double charge?**

Answer:

* Idempotency key
* Conditional write
* Transaction for debit/credit
* TTL for cleanup

---

## 🔹 Use Case 2: Fraud Event Store (High Volume)

Events:

* Card swipe
* Login attempt
* Geo change
* Velocity anomaly

Design:

```
PK = USER#<userId>
SK = FRAUD#<timestamp>
```

Query:

* Last N fraud events
* Events between time window

Best practice:

* Use composite SK
* Write sharding if high traffic user

---

## 🔹 Use Case 3: AML Audit Trail

Pattern:

```
DynamoDB Streams → Lambda → S3
```

Why?

* Immutable storage
* Regulatory compliance
* Cheap long-term storage

---

## 🔹 What NOT to Store in DynamoDB

❌ Complex relational ledger
❌ Heavy aggregation queries
❌ Financial reconciliation engine

That belongs in:

* Aurora
* PostgreSQL

---

# 🏢 2️⃣ DynamoDB in Real Estate

Real estate platforms (Dubai, Doha):

* Property listings
* Lead management
* Agent dashboards
* Slot booking

---

## 🔹 Use Case 1: Property Listing

Design:

```
PK = CITY#Dubai
SK = PROPERTY#<propertyId>
```

GSI:

```
GSI1PK = PROPERTYTYPE#Apartment
GSI1SK = PRICE
```

Why DynamoDB?

* Read-heavy
* Low-latency search
* Millions of listings

---

## 🔹 Use Case 2: Slot Booking (Atomic)

Prevent double booking:

```
ConditionExpression attribute_not_exists(slotId)
```

Atomic write = no race condition.

---

### 🔥 Interview Q

**Q: How would you scale Dubai property search to 10M users?**

Answer:

* DynamoDB for metadata
* OpenSearch for full-text search
* Denormalized item structure
* No Scan
* GSIs for filter patterns

---

# 🛒 3️⃣ DynamoDB in E-Commerce

Used heavily for:

* Shopping carts
* Orders
* Product catalogs
* Flash sales

---

## 🔹 Shopping Cart

```
PK = USER#123
SK = CART
```

TTL:
Auto delete abandoned carts.

Why DynamoDB?

* Per-user partition
* Fast reads/writes
* No joins

---

## 🔹 Flash Sale Design

Problem:
1M users hitting one product.

Solution:

```
PK = PRODUCT#123#01
PK = PRODUCT#123#02
```

Write sharding.

Avoid hot partition.

---

### 🔥 Interview Q

**Q: Why not use DynamoDB for analytics?**

Answer:

* Optimized for key-based lookup
* No heavy aggregation engine
* Use Redshift / Athena for analytics

---

# 🧠 Industry Best Practices

---

## 1️⃣ Access Pattern First

Never design schema first.

Ask:

* What queries?
* What frequency?
* What scale?

---

## 2️⃣ Avoid Hot Partitions

Bad:

```
PK = COUNTRY#UAE
```

Good:

```
PK = COUNTRY#UAE#<propertyId>
```

---

## 3️⃣ Keep Items Small

* 400 KB max
* Larger item = more RCU
* Avoid blobs

---

## 4️⃣ Sparse GSIs

Only index what you query.

---

## 5️⃣ Streams for Event Driven

```
DynamoDB → Lambda → Kafka/SQS
```

---

# 🌍 Multi-Region Middle East Design

If serving:

* UAE
* Qatar
* Saudi

Use:

* Global Tables
* Active-active replication
* Local writes

But:

Ledger → Still relational.

---

# ⚔ DynamoDB vs RDS Summary

| Use Case      | DynamoDB | RDS |
| ------------- | -------- | --- |
| Ledger        | ❌        | ✅   |
| Fraud logs    | ✅        | ❌   |
| Sessions      | ✅        | ❌   |
| Shopping cart | ✅        | ❌   |
| Reporting     | ❌        | ✅   |

---

# 🏦 Now: BFSI Single-Table Design (Architect Level)

This is what makes you sound senior.

---

## 🎯 Required Access Patterns

1. Get account
2. Get transactions (sorted)
3. Get txn by ID
4. Idempotency check
5. Beneficiaries
6. Fraud events
7. Monthly statement
8. Audit history

---

# 🏗 Table: `BankingMainTable`

```
PK
SK
```

---

## 🏦 Account

```
PK = ACCOUNT#<accountId>
SK = METADATA
```

---

## 💳 Transaction

```
PK = ACCOUNT#<accountId>
SK = TXN#<timestamp>#<txnId>
```

Sorted automatically.

---

## 🔁 Idempotency

```
PK = IDEMPOTENCY#<uuid>
SK = METADATA
```

---

## 👤 Beneficiary

```
PK = USER#<userId>
SK = BENEFICIARY#<beneficiaryId>
```

---

## 🚨 Fraud

```
PK = TXN#<txnId>
SK = FRAUD#<timestamp>
```

---

## 📑 Audit

```
PK = ACCOUNT#<accountId>
SK = AUDIT#<timestamp>#<eventId>
```

---

# 🔍 GSIs

### GSI1 – Get Transaction by txnId

```
GSI1PK = TXN#<txnId>
GSI1SK = ACCOUNT#<accountId>
```

---

### GSI2 – Get Accounts by User

```
GSI2PK = USER#<userId>
GSI2SK = ACCOUNT#<accountId>
```

---

### GSI3 – Merchant View

```
GSI3PK = MERCHANT#<merchantId>
GSI3SK = TXN#<timestamp>
```

---

# 🧮 Monthly Statement Query

```
PK = ACCOUNT#123
SK BETWEEN TXN#2026-01-01 AND TXN#2026-01-31
```

No joins.
No scans.

---

# 🔐 Prevent Double Spending

1️⃣ Write idempotency key
2️⃣ TransactWriteItems:

* Update balance
* Insert TXN
* Insert audit

Atomic.

---

# 🧠 Senior One-Liner (Memorize This)

If interviewer asks:

> Explain your DynamoDB banking design.

Say:

> I model entities using prefixed PK/SK patterns. Account acts as the partition root. Transactions and audits are stored as item collections. GSIs handle reverse lookups like txnId and merchant queries. All access is key-based, no scans, optimized for predictable high-scale workloads.

That is architect-level language.

---
