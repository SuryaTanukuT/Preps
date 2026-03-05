
---

# 🔥 1️⃣ Allied World Insurance (AWAC – Singapore)

### Domain: Insurance (General + Commercial)

Type: Enterprise B2B insurance platform

---

## 🎯 What You Actually Built

* Customer portal (policy management)
* Admin portal (underwriting, approvals)
* Policy document workflows
* Claims handling APIs
* Rule-based automation
* Payment integrations
* DynamoDB-based storage
* Lambda-based backend
* React + Redux frontend

---

## 🧠 Architecture (Explain Like This)

Presentation Layer
→ React SPA (Customer & Admin dashboards)

API Layer
→ API Gateway
→ AWS Lambda (Node.js)

Business Logic Layer
→ Policy rules engine
→ Claims validation
→ Payment integration logic

Data Layer
→ DynamoDB (6 core tables)
→ S3 (documents)

Workflow Layer
→ AWS Step Functions (policy lifecycle automation)

Security
→ JWT authentication
→ Role-based access (Customer / Underwriter / Admin)
→ Encryption at rest
→ PCI-DSS aligned payment flow

---

## 🔥 If They Ask: Why Only 6 Tables?

Say:

> Insurance domain was designed around denormalized NoSQL model in DynamoDB. We optimized for access patterns rather than relational normalization.

That’s correct.

Explain:

* Policy table
* Customer table
* Claims table
* Payment table
* User roles table
* Audit log table

DynamoDB works on access patterns, not joins.

---

## 🔥 Complexities in Insurance

### 1️⃣ Policy Lifecycle Complexity

Policy states:

Draft → Under Review → Approved → Active → Expired → Claimed

Solution:
Used AWS Step Functions to model state transitions.

If they ask how:

Step Function orchestrated:

* Validate request
* Underwriter approval
* Payment confirmation
* Policy activation

This ensures deterministic workflow.

---

### 2️⃣ Rule Engine Complexity

Insurance has:

* Eligibility rules
* Premium calculation
* Risk validation

Solution:
Modular rule validation services inside Lambda.

---

### 3️⃣ Data Validation

You mentioned scanning & validating existing data.

Say:

> We implemented validation middleware to ensure legacy data consistency before policy migration.

---

### 4️⃣ Payments

If asked about Razorpay/Stripe:

Correct explanation:

Frontend → gets session token
Backend → creates payment intent
Webhook → confirms transaction
Policy activated only after verification

---

## 🔥 Tradeoffs

Lambda pros:

* Cost-effective
* Auto-scaling

Cons:

* Cold start latency
* Harder debugging

DynamoDB pros:

* High scalability
* Flexible schema

Cons:

* No complex joins
* Access pattern design critical

Mention tradeoffs → senior-level answer.

---

# 🔥 2️⃣ Lloyds Banking Group (Retail + Commercial Banking)

This is high-value answer.

---

## 🎯 What You Built

Greenfield banking system:

* Notifications engine
* Customer dashboards
* Payment alerts
* Rule-based event triggers
* Account monitoring

---

## 🧠 Architecture

Frontend
→ React / Next.js

Backend
→ NestJS microservices

Event Layer
→ Kafka or EventBridge

Database
→ PostgreSQL

Cache
→ Redis

Notifications
→ Email/SMS service

---

## 🔥 Challenging Part

Notification engine at scale.

Problem:
Millions of events (transactions, payments, balance changes).

Solution:

* Event-driven architecture
* Publish event on transaction
* Notification service consumes
* Rule evaluation
* Send notification async

---

## 🔥 Tradeoffs

Event-driven:

* Scalable
* Decoupled

- Eventual consistency

Monolith:

* Simpler

- Hard to scale

You chose event-driven.

---

# 🔥 3️⃣ ENBD (Emirates NBD – Digital Banking)

Core banking stack.

Explain:

* Microservices
* Secure authentication
* RBAC
* Account management
* API rate limiting
* PCI compliance

Focus on:

Security & regulatory compliance.

---

# 🔥 4️⃣ British Telecom (BT)

Domain: Telecom / Network Analytics

Architecture:

* High-frequency data
* Aggregation dashboards
* Client-based segmentation
* SOA architecture

Challenge:

Handling large volume time-series data.

Solution:

* Aggregation services
* Caching
* Partitioned tables

---

# 🔥 5️⃣ Eli Lilly (Healthcare)

Domain: Healthcare workflows

You built:

* Reusable component libraries
* Micro-frontend architecture
* Shared UI packages
* Backend reusable microservices
* AI integrations (Claude, RAG)

Architecture:

Frontend packages → NPM library
Microservices → Node/Nest
Database → PostgreSQL
Vector DB → Qdrant
AI → LangChain/LangGraph

---

# 🔥 If They Ask: Most Challenging Architecture

Choose Lloyds notification engine.

Explain:

1. Event-driven design
2. Idempotency handling
3. Rate limiting
4. Failure retry strategy
5. Monitoring

---

# 🔥 If They Ask: Compare Insurance vs Banking Architecture

Insurance:

* Workflow heavy
* Rule-based
* Step Functions useful

Banking:

* Transaction heavy
* Event-driven
* Strong consistency required

---

# 🔥 If They Ask: How Would You Redesign Today?

Say:

* Use Kubernetes instead of only Lambda
* Observability with tracing
* Better CQRS separation
* Saga pattern for distributed transactions

---

# 🔥 Backend Responsibilities (Spicy but Real)

What backend did:

* Authentication
* Authorization
* Business rules
* Data validation
* Payment verification
* Event publishing
* Retry mechanisms
* Rate limiting
* Audit logging
* Error handling

Never say “just API writing”.

---

# 🔥 If They Ask: Testing

Unit:
Jest

E2E:
Playwright

Coverage:
85%+

Explain:

* Mocked DynamoDB
* Tested Lambda handlers
* API contract testing

---

# 🔥 Final Advice

When explaining architecture:

Always follow:

Client → API → Service → DB → Event → Cache → Security → Monitoring

Never jump randomly.

---
