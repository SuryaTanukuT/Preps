
## 1) MongoDB vs Amazon DynamoDB vs Apache Cassandra (BFSI-grade comparison)

### Where each fits in BFSI / FinTech

| Need                                                         | Best fit                    | Why                                                   |
| ------------------------------------------------------------ | --------------------------- | ----------------------------------------------------- |
| **Core double-entry ledger**                                 | **PostgreSQL / SQL Server** | strict ACID + constraints + deterministic correctness |
| Customer 360 / profile                                       | MongoDB                     | flexible schema + fast indexed lookups                |
| High-volume **event store / audit logs**                     | MongoDB or Cassandra        | append-heavy + partitionable                          |
| Ultra-high TPS counters (fraud velocity, OTP, risk features) | DynamoDB / Cassandra        | predictable access, massive horizontal scale          |
| Real-time pub/sub + replay                                   | Apache Kafka                | durable log + replay + ordering per key               |
| Search / investigations                                      | Elasticsearch               | full-text + aggregations                              |

### Decision points interviewers care about

**MongoDB**

* ✅ Great for flexible docs, read models, audit/event documents, geo queries, TTL
* ⚠️ For “money correctness across many entities”, you must be **careful**: multi-doc transactions exist, but the modeling + correctness burden is higher than relational ledgers.

**DynamoDB**

* ✅ Serverless scale, predictable access, TTL, global tables, extremely high throughput
* ⚠️ Harder ad-hoc querying; you must design around access patterns + GSIs cost.

**Cassandra**

* ✅ Massive write throughput + multi-region + HA
* ⚠️ Eventual consistency by default; modeling is query-driven; limited flexibility.

**Senior-line you can say**

> “For BFSI, I keep ledger correctness in a relational source-of-truth, and use Mongo/Dynamo/Cassandra as **read models, event stores, and feature stores** where scale + flexible schema matters.”

---

## 2) Node.js / NestJS + MongoDB Production Blueprint (Bank-grade)

### 2.1 Reference architecture (what you draw on whiteboard)

* API (NestJS)
* MongoDB replica set or sharded cluster
* Kafka for event bus
* Redis for idempotency / counters / rate limit
* Observability + audit pipeline

**Core idea:** MongoDB is **not** your ledger truth — it’s your **profile store + read model + event/audit store**.

---

### 2.2 MongoDB topology choices (what to choose and why)

#### A) Replica Set (most common starting point)

Use when:

* You need HA (primary + secondaries)
* Throughput is high but not “planet-scale”
* You can scale vertically + index well

**Bank-grade settings**

* `writeConcern: "majority"` for critical writes
* `readConcern: "majority"` or `snapshot` (txn) where correctness matters
* `retryWrites: true`, plus **app-level idempotency** (still required)

#### B) Sharded Cluster (when volume truly grows)

Use when:

* Collections become huge (audit/events/transactions)
* Write throughput becomes the bottleneck
* You need horizontal scale

**Shard-key rule (interview favorite)**

* Avoid monotonic keys (`createdAt`) → hot shard
* Prefer high-cardinality distribution keys:

  * `hashed(accountId)` for payments/events
  * `hashed(customerId)` for profiles
  * If geo needs residency: zone sharding by region + hashed within region

---

### 2.3 Data modeling patterns (BFSI/FinTech style)

#### Pattern 1: Customer 360 Profile (MongoDB wins)

* Flexible schema for KYC attributes, devices, preferences, risk flags
* Keep a stable core + evolving metadata

Indexes:

* `{ customerId: 1 }` unique
* `{ nationalIdHash: 1 }` (hashed/pseudonymized)
* TTL for “temporary KYC attempts” if needed

#### Pattern 2: Transaction “Read Model” (MongoDB fits)

* Store “queryable transaction history” for customer apps and ops dashboards
* Source-of-truth still posts to ledger elsewhere; Mongo holds the “serving view”

Indexes:

* `{ accountId: 1, createdAt: -1 }`
* `{ txnId: 1 }` unique
* Partial indexes for `status: "PENDING"` etc.

#### Pattern 3: Audit / Compliance events (append-only)

* Immutable-by-policy: no updates, no deletes
* Partitionable by time, archive to cold storage

Indexes:

* `{ entityType: 1, entityId: 1, createdAt: -1 }`
* `{ requestId: 1 }`

**Tamper-evidence option**

* hash chain: `prevHash`, `hash(payload + prevHash)` (store + validate periodically)

#### Pattern 4: Fraud / AML events (high write)

* Store “events” + “features”
* Keep raw events in Mongo/Cassandra, keep counters in Redis/Dynamo (TTL)

---

### 2.4 Idempotency (bank-grade) — correct implementation

**Don’t** only check `payments.findOne({ idempotencyKey })` without unique constraints.
**Do** enforce uniqueness + “return stored result”.

**Collection**
`idempotency_keys`

* `{ key, scope, status, response, createdAt, expiresAt }`
* Unique index: `{ scope: 1, key: 1 }`
* TTL index: `{ expiresAt: 1 }`

**NestJS-ish pattern (pseudo-code)**

```ts
// 1) Try insert idempotency record (wins the race)
// 2) If duplicate key -> return stored response
// 3) Execute business
// 4) Store final response atomically
```

Why this wins:

* Handles concurrent retries safely
* Gives deterministic replay (critical for BFSI APIs)

---

### 2.5 MongoDB transactions (when you use them, and how)

**Use transactions only when you truly need cross-document atomicity**, e.g.:

* wallet debit + credit in Mongo (still not ideal vs relational ledger)
* compliance state transitions across collections

**Transaction hardening**

* Keep txn short
* Use deterministic ordering of updates
* Retry on transient txn errors with jitter
* Use `readConcern: "snapshot"` + `writeConcern: "majority"` for txn

**Senior interview line**

> “I design atomic document boundaries first, and only use multi-document transactions where strict cross-collection correctness is mandatory.”

---

### 2.6 Performance best practices (what architects mention)

**Indexes**

* Every query path must be indexed (avoid COLLSCAN)
* Use compound indexes matching sort/filter
* Minimize index count on hot-write collections (write amplification)

**Document size discipline**

* Avoid unbounded arrays (use separate collection if it grows)
* Use projection to reduce payload size
* Add archiving strategy for audit/events

**Connection pooling (Node)**

* Set `maxPoolSize` based on workload and CPU
* Prefer one shared client per process

**Aggregation performance**

* Push `$match` early
* Use `$project` to shrink docs
* Use `explain("executionStats")`
* `allowDiskUse` for heavy pipelines (but treat it as a smell)

---

### 2.7 Security & compliance knobs (what to explicitly say)

* TLS everywhere
* Least-privilege DB roles
* Field-level encryption / client-side encryption for PII where required
* Audit logging enabled at DB + app level
* Backups encrypted; restore drills
* Write concern majority for critical writes
* Data retention policies (TTL only for ephemeral, not financial records)

---

### 2.8 Common failure scenarios + mitigations (Mongo + Node)

| Failure                | What happens                 | Fix                                           |
| ---------------------- | ---------------------------- | --------------------------------------------- |
| Duplicate transactions | retries create multiple docs | unique idempotency + stored replay            |
| Hot shard              | shard key monotonic          | hashed high-cardinality shard key             |
| Replica lag surprises  | reads see stale              | readPreference primary for correctness paths  |
| Slow queries           | COLLSCAN                     | indexes + correct compound keys + projections |
| Aggregation timeout    | huge pipeline                | match early + pre-aggregates + rollups        |
| Unbounded doc growth   | doc hits limits              | split collections + cap arrays                |
| Change stream gaps     | consumer downtime            | resume tokens + durable event log (Kafka)     |

---

## 3) Ledger architecture deep design (Mongo + Postgres hybrid) — the “real answer”

This is the pattern Middle East BFSI/FinTech teams love because it’s pragmatic and safe:

### 3.1 Responsibilities split

* **Ledger posting & invariants** → PostgreSQL (or SQL Server)
* **Customer profile, read models, audit events, AML/fraud event store** → MongoDB
* **Async event bus** → Apache Kafka
* **Idempotency + counters** → Redis

### 3.2 Data flow (clean and auditable)

1. API receives request (+ idempotency key)
2. Writes **ledger TX + outbox** in Postgres atomically
3. Outbox publisher pushes event to Kafka
4. Consumers update Mongo read models + audit stores
5. Queries (mobile app / ops) hit Mongo for fast reads
6. Balance correctness reads come from ledger (or derived balances maintained from ledger events)

**The pitch**

> “Ledger correctness lives in relational DB; MongoDB serves profiles and read-optimized transaction views. Kafka + outbox keeps everything consistent and replayable.”

---

## 4) Interview-style Q&A (Middle East BFSI/FinTech tone)

### Q1) Where does MongoDB fit in BFSI?

**A:** “Profiles, audit/event stores, AML/fraud events, transaction history read models. Not the core double-entry ledger.”

### Q2) How do you prevent duplicate debits in a Node API?

**A:** “Idempotency key with unique constraint + store the final response and replay it for retries. Also lock/serialize posting at the ledger boundary.”

### Q3) How do you handle high write volume safely in Mongo?

**A:** “Right shard key (hashed high-cardinality), keep documents small, index only what you query, majority write concern for critical collections, and archive cold data.”

### Q4) How do you push Mongo changes to an event bus?

**A:** “Change streams can trigger publish, but for bank-grade reliability I prefer a durable log via outbox/Kafka so replay is guaranteed.”

### Q5) How do you design fraud scoring storage?

**A:** “Counters with TTL in Redis/Dynamo, raw events in Mongo, decisions persisted with audit fields, and stream everything for offline analytics.”

### Q6) What’s the biggest Mongo mistake in BFSI?

**A:** “Using Mongo as the ledger truth and relying on application logic for invariants instead of DB-enforced correctness.”

