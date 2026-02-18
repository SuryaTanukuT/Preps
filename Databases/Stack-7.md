
---

# 🏦 1) BFSI (Banking / Core Finance / Ledger)

## ✅ Real use cases

* Core ledger (GL + sub-ledger), balances, account lifecycle
* Internal transfers, external rails posting (NEFT/RTGS/SWIFT equivalents)
* Card auth/capture/refund/reversal
* Loans: disbursal, repayment schedules, interest accrual, delinquency
* Reconciliation (nostro, settlement files), regulatory reporting
* Maker-checker approvals, audit & investigations

## ✅ Database choices (canonical)

| Component                          | DB choice                                      | Why                                   |
| ---------------------------------- | ---------------------------------------------- | ------------------------------------- |
| **Ledger source-of-truth**         | **PostgreSQL / SQL Server**                    | ACID + constraints + deterministic TX |
| Statements / reporting reads       | Read replicas + materialized views / OLAP sink | isolate read load                     |
| Idempotency / rate limit / session | Redis                                          | low latency + TTL                     |
| Audit/event stream                 | Kafka + immutable store (DB/object storage)    | replay + long retention               |
| Search over cases/logs             | Elasticsearch                                  | investigator queries                  |

## ✅ Best practices (bank-grade)

### 1) Double-entry ledger (non-negotiable)

* Journal header (`ledger_transactions`)
* Journal lines (`ledger_postings`) — append-only
* **Invariant:** `SUM(debits) == SUM(credits)` per txn

### 2) Concurrency correctness

* Balance check **inside same DB transaction**
* Lock critical rows (`SELECT … FOR UPDATE`)
* Prefer **SERIALIZABLE** (or strict locks + invariants)

### 3) Idempotency (must)

* Unique `(tenant_id, idempotency_key)`
* Store final result (txn_id/status) for replay-safe retries

### 4) Immutability

* Never update/delete money postings
* Use reversal entries linked to original txn

### 5) Partitioning + retention

* Partition `ledger_postings` by `posted_at` (monthly/weekly)
* Archive old partitions (7–10 years retention is common)
* Keep “hot” partitions small + indexed

## ✅ Architecture patterns

### Primary pattern: “Ledger-first + Outbox”

* **Ledger TX commits → outbox row → async publish**
* Avoid distributed 2PC; use **Outbox + Saga** for cross-service workflows

**Reference flow**
`API → Auth → (Redis idempotency) → Postgres TX (lock + post) → Outbox → Kafka → downstream`

## ✅ Common failure scenarios (and what you answer)

| Failure                                    | Root cause                       | Fix                                           |
| ------------------------------------------ | -------------------------------- | --------------------------------------------- |
| Double debit                               | no idempotency / non-atomic flow | DB unique idempotency + single TX             |
| Wrong balance under concurrency            | check-then-write outside TX      | lock + validate + post inside TX              |
| Deadlocks                                  | inconsistent lock ordering       | canonical lock order + short TX + retry       |
| Replica shows stale balance                | replication lag                  | balance reads from primary (or “as-of”)       |
| “Ledger posted but downstream not updated” | publish failed after commit      | outbox publisher + replay                     |
| Performance collapse on statements         | unbounded scans                  | partition pruning + indexes + CQRS read model |

## 🎤 BFSI Interview Q&A (short, senior-ready)

1. **How do you prevent double spending?**
   **A:** “Idempotency key + unique constraint, lock the account row, validate funds and write postings in one TX. Optionally SERIALIZABLE to prevent write skew.”

2. **Why is balance not the source of truth?**
   **A:** “Balance is derived. Source of truth is append-only postings; balance table is cacheable/rebuildable.”

3. **How do you handle reversals/chargebacks?**
   **A:** “Create a reversal txn with opposite postings linked to original; never mutate past rows.”

4. **How do you guarantee ‘debit=credit’?**
   **A:** “Enforce via transaction-level invariant—constraint trigger or stored procedure validating sums before commit.”

5. **How do you design for auditability?**
   **A:** “Immutable audit log + correlation IDs across request→ledger→outbox→events; strict privileges (no update/delete).”

6. **Replica lag—what do users see?**
   **A:** “Consistency-critical reads (balances) from primary; statements can tolerate replica lag with as-of timestamp.”

---

# 💳 2) FinTech (Payments / Wallets / NeoBanks)

## ✅ Real use cases

* Wallet top-up, P2P transfer, merchant pay
* Card issuing lifecycle, auth/capture/refund
* BNPL schedules, fees, penalties
* FX conversion, multi-currency wallets
* Fraud scoring, velocity rules, device risk
* Settlement + reconciliation with partners

## ✅ Database choices

| Component                 | DB                           | Why                            |
| ------------------------- | ---------------------------- | ------------------------------ |
| Wallet ledger             | Postgres/SQL Server          | correctness like BFSI          |
| Fraud counters / velocity | Redis / DynamoDB / Cassandra | high write + TTL + low latency |
| Event stream              | Kafka                        | durable streaming + replay     |
| Customer/profile          | Postgres or MongoDB          | depends on schema volatility   |
| Search                    | Elasticsearch                | merchant/txn/case search       |

## ✅ Best practices

### 1) Wallet correctness model

* Ledger postings = truth
* Balance table = derived (fast reads)
* Holds/reservations for auth/capture

### 2) Event-driven boundaries (strongly recommended)

* Payment commit emits event via Outbox
* Downstream: notifications, risk, analytics consume events idempotently

### 3) Consumer idempotency

* Unique `(consumer_name, event_id)` in consumer DB

### 4) Multi-currency + FX

* Store amounts in minor units + currency code
* FX legs are separate postings with rate snapshot

## ✅ Architecture patterns

* **CQRS** for read-heavy dashboards
* **Saga orchestration** for multi-step flows (payment → reserve → confirm)
* **Feature store** pattern for fraud: fast TTL counters + event stream sink

## ✅ Common failure scenarios

| Failure                 | Cause                      | Fix                                           |
| ----------------------- | -------------------------- | --------------------------------------------- |
| Duplicate wallet credit | retry without idempotency  | idempotency in DB + result replay             |
| Fraud engine overload   | too many synchronous calls | async scoring + cached signals + backpressure |
| Lost events             | publish outside TX         | Outbox pattern                                |
| Counter drift           | non-atomic increments      | atomic ops + TTL + periodic reconciliation    |

## 🎤 FinTech Interview Q&A

1. **How do you design real-time fraud scoring?**
   **A:** “Velocity counters in Redis/DynamoDB with TTL, consume payment events from Kafka, store decision + features for audit.”

2. **How do you avoid calling downstream services in checkout?**
   **A:** “Commit ledger first, publish event, downstream runs async. Only strict dependencies remain synchronous.”

3. **How do you handle exactly-once semantics?**
   **A:** “Effectively-once via idempotency keys + consumer dedupe + outbox replay.”

---

# 🛒 3) E-Commerce (High traffic + some eventual consistency)

## ✅ Real use cases

* Catalog, pricing, inventory, cart, checkout
* Orders, shipments, returns, refunds
* Recommendations, personalization
* Search, filters, facets, ranking

## ✅ Database choices

| Component         | DB                         | Why                              |
| ----------------- | -------------------------- | -------------------------------- |
| Orders + payments | Postgres/MySQL             | transactional                    |
| Catalog core      | Postgres or MongoDB        | depends on attribute flexibility |
| Cart              | Redis (+ DB fallback)      | ultra-low latency                |
| Inventory         | Strong DB (Postgres/MySQL) | oversell prevention              |
| Search            | Elasticsearch              | inverted index + faceting        |
| Events            | Kafka/RabbitMQ             | async processing + retries       |

## ✅ Best practices

### 1) Inventory correctness (the “money” of e-comm)

* Single source of truth for stock
* Atomic decrement in strong store
* Reservation/hold with TTL for checkout

### 2) Flash sale resilience

* Per-SKU throttling + queueing
* Single-writer inventory service (or partitioned ownership)
* Pre-allocated buckets (optional) for extreme spikes

### 3) Search = eventually consistent

* Outbox/CDC to sync catalog → search index
* Don’t use search as source of truth

### 4) Cart model

* Redis for active carts, TTL for abandonment
* Persist important carts in DB periodically (or on checkout)

## ✅ Architecture patterns

* **CQRS + denormalized read models** (browse vs checkout)
* **Outbox/CDC** from OLTP to search + analytics
* **Async pipeline** for emails, invoices, recommendations

## ✅ Common failure scenarios

| Failure                  | Cause                              | Fix                                                |
| ------------------------ | ---------------------------------- | -------------------------------------------------- |
| Overselling              | cache-trust / non-atomic decrement | strong store + reservation                         |
| Inventory stuck          | TTL/hold not released              | TTL release job + idempotent cleanup               |
| Search shows wrong price | lag in indexing                    | versioned docs + reindex + “source-of-truth fetch” |
| Cart lost                | Redis eviction                     | DB fallback + persistence                          |

## 🎤 E-Commerce Interview Q&A

1. **How do you prevent overselling in flash sales?**
   **A:** “Inventory in strong DB, atomic decrement, reservation with TTL; optionally queue requests per SKU.”

2. **Why separate Elasticsearch from DB?**
   **A:** “Search requires inverted indexes, scoring and faceting; OLTP isn’t built for that.”

3. **Where is eventual consistency acceptable?**
   **A:** “Search, recommendations, emails—never for order payment capture or inventory decrement.”

---

# 🏠 4) Real Estate / Property Platforms (Search-heavy + Geo-heavy + Doc-heavy)

## ✅ Real use cases

* Listings CRUD, broker/agent CRM, leads
* Geo search (radius/bbox), filters (price, beds, area)
* Similar properties, ranking, personalization
* Media/documents (images, floorplans, legal docs)
* Moderation, fraud/spam listing detection

## ✅ Database choices

| Component          | DB                 | Why                              |
| ------------------ | ------------------ | -------------------------------- |
| Canonical listings | Postgres           | relational integrity + reporting |
| Geo queries        | Postgres + PostGIS | spatial indexes + joins          |
| Full-text + facets | Elasticsearch      | filter + ranking + aggregation   |
| Media/documents    | Object store (S3)  | durable, cheap, scalable         |
| Activity/events    | Kafka              | tracking + analytics pipeline    |

## ✅ Best practices

### 1) Geo modeling

* Store lat/lng + geometry point
* Spatial index (GiST) for fast geo queries
* Combine with business filters efficiently

### 2) Search design (two-step is common)

* Query Elastic for matching IDs + ranking
* Fetch canonical details from Postgres (or store enough denormalized fields)

### 3) Document-heavy workflows

* Files in object store
* DB stores metadata + permissions + audit events
* Index only searchable metadata in Elastic

### 4) Multi-tenant broker platforms

* Shared DB + `tenant_id` + (optional) Postgres RLS
* Or schema-per-tenant for strict isolation (ops tradeoff)

## ✅ Architecture patterns

* **Elastic as read model**, Postgres as source-of-truth
* **Caching** popular geo tiles / popular filters
* **Event pipeline** for analytics & ranking signals

## ✅ Common failure scenarios

| Failure               | Cause                       | Fix                                                    |
| --------------------- | --------------------------- | ------------------------------------------------------ |
| Slow geo + filters    | wrong indexes / heavy joins | PostGIS index + precomputed facets + denorm read model |
| Elastic drift from DB | missed updates              | outbox/CDC + replay + periodic reindex                 |
| Duplicate listings    | weak dedupe                 | canonical keys + similarity hashing + moderation queue |
| Hot query spikes      | popular locations           | cache hot tiles + rate limit                           |

## 🎤 Real Estate Interview Q&A

1. **How do you implement geo-radius search with filters?**
   **A:** “Use Elastic for faceting/filtering/ranking; PostGIS for canonical geo queries; cache hot tiles.”

2. **How do you handle document storage securely?**
   **A:** “Object store for files, DB metadata + signed URLs + audit trails; never store blobs in OLTP.”

---

# 🔥 Cross-domain “what to choose + why” summary

## DB strategy by domain

| Domain      | Source of truth                  | Read optimization     | Cache/kv             | Search               |
| ----------- | -------------------------------- | --------------------- | -------------------- | -------------------- |
| BFSI        | Postgres/SQL Server ledger       | replicas + CQRS + MVs | Redis (idempotency)  | Elastic (cases/logs) |
| FinTech     | Postgres ledger + events         | CQRS + stream sinks   | Redis/Dynamo (fraud) | Elastic              |
| E-Comm      | Orders in SQL; catalog SQL/Mongo | denorm read models    | Redis carts          | Elastic              |
| Real Estate | Postgres + PostGIS               | Elastic read model    | cache tiles          | Elastic              |

## Failure-thinking that wins interviews

* Correctness boundaries: **ledger/inventory are strict**, search/reco are eventual
* Reliability: **outbox + replay** beats “best-effort publish”
* Scale: partition hot history, isolate reads, mitigate hotspots
* Security: least privilege + auditability + encryption + retention discipline

---
