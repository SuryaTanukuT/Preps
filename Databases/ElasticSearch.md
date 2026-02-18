
https://developer.okta.com/blog/2022/04/27/ultimate-guide-elasticsearch-nodejs

---

# 🔥 1️⃣ Elasticsearch + Node.js Integration (Production Grade)

For your stack (Node.js, NestJS, Kafka, Redis, AWS), this is how ES fits.

### Official Client

Use:

```
@elastic/elasticsearch
```

### Basic Setup

```js
import { Client } from '@elastic/elasticsearch'

const client = new Client({
  node: 'https://es-cluster:9200',
  auth: {
    username: 'elastic',
    password: 'password'
  },
  tls: {
    rejectUnauthorized: true
  }
})
```

---

## Production Best Practices (Node.js)

✅ Use connection pooling (client handles it)
✅ Use bulk indexing for high throughput
✅ Never index synchronously inside user request flow
✅ Use Kafka → Consumer → Bulk index pattern
✅ Retry with exponential backoff
✅ Use circuit breaker pattern in service layer
✅ Timeouts must be configured
✅ Avoid large response payloads (_source filtering)

---

# 🏦 2️⃣ BFSI Use Cases (Architect Level)

## A) 🔎 Transaction Search (Core Banking)

Requirement:

* Search 50M transactions
* Filter by account, date range, amount
* Must return in < 200ms

Design:

* Index per year: `transactions_2026`
* Shard size: 30GB
* Mapping:

  * account_id → keyword
  * amount → double
  * txn_date → date
  * description → text + keyword
* Use filter context (not scoring)

Query:

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "account_id": "12345" }},
        { "range": { "txn_date": { "gte": "now-30d" }}}
      ]
    }
  }
}
```

Why filter?
→ No scoring → Faster → Uses query cache

Interview Tip:

> In BFSI, most searches are filter-based, not relevance-based.

---

## B) 🚨 Fraud Detection (FinTech)

Pattern:

* High velocity transactions
* Real-time anomaly detection

Architecture:

Kafka → Fraud Service →

1. Store in DB
2. Index in Elasticsearch
3. Run rule-based queries
4. Trigger alert if matched

Example:

* Geo anomaly:

  * Last transaction India
  * Next transaction UAE within 5 minutes

Use:

* geo_point
* scripted queries
* aggregations

---

## C) 📜 Audit Logs (Compliance / AML / GDPR)

Requirement:

* Immutable logs
* 7-year retention
* Search by user/action/date

Best Practice:

* Data streams
* ILM policy

  * Hot: 30 days
  * Warm: 6 months
  * Cold: 1 year
  * Frozen: remaining years

Important:

> Never allow update/delete for audit indices.

---

# 🛒 3️⃣ E-Commerce Design

## Product Search Architecture

![Image](https://i.sstatic.net/5yyY2.png)

![Image](https://miro.medium.com/1%2A3z9uZh68kT2kvTCaDFDB3g.png)

![Image](https://miro.medium.com/0%2AxuHRipbS0io0EYVl.png)

![Image](https://dinarys.com/photos/7/Ecommerce%20Website%20Architecture%20Tutorial%20%201.png)

### Mapping Strategy

| Field      | Type           | Why                  |
| ---------- | -------------- | -------------------- |
| name       | text + keyword | Search + exact match |
| category   | keyword        | Filters              |
| price      | double         | Sorting              |
| rating     | float          | Ranking              |
| created_at | date           | Sorting              |

---

## Autocomplete

Use:

* edge_ngram analyzer
* completion suggester

Example:

```json
"analysis": {
  "analyzer": {
    "autocomplete": {
      "tokenizer": "edge_ngram",
      "filter": ["lowercase"]
    }
  }
}
```

Interview Answer:

> For high-scale autocomplete, I prefer completion suggester because it uses FST in memory and is faster than edge n-gram for large datasets.

---

# 🏢 4️⃣ Real Estate Use Case (Geo Search)

![Image](https://us1.discourse-cdn.com/elastic/original/3X/e/1/e1402be3a50bda96cd66720e27b7488027a4534e.jpeg)

![Image](https://miro.medium.com/1%2AB9hiqE6jpqQihFanUpE-aw.png)

![Image](https://cdn.dribbble.com/userupload/13405866/file/original-b26155826c9af571b556c6c73c2c7b0f.png?format=webp\&resize=400x300\&vertical=center)

![Image](https://cdn.dribbble.com/userupload/23582535/file/original-deff83b901e4f7316a6a6bdcfde3dff1.png?crop=0x0-6402x4801\&format=webp\&resize=400x300\&vertical=center)

### Mapping

```json
"location": {
  "type": "geo_point"
}
```

### Query

```json
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "5km",
          "location": {
            "lat": 17.3850,
            "lon": 78.4867
          }
        }
      }
    }
  }
}
```

Important:

> Geo queries should always be in filter context.

---

# ⚡ 5️⃣ Performance Architecture (Senior Level)

## Ideal Shard Strategy

* 20–50GB per shard
* 1 primary shard per ~30GB data
* Avoid oversharding
* Use index rollover

Bad:

* 5GB shards
* 1000 small shards

Why?
→ Cluster state explosion
→ High heap usage

---

## Bulk Indexing Pattern (Production)

1. Batch size: 5MB–15MB
2. Disable refresh during bulk
3. Increase refresh_interval to 30s
4. Re-enable after bulk
5. Use parallel workers

---

## Heap Rule

* Heap = 50% of RAM
* Max heap ≤ 32GB
* Leave memory for OS cache

---

# 🧠 6️⃣ Senior-Level Interview Q&A

---

### Q1: Why is Elasticsearch near real-time?

Because documents become searchable after a refresh, which happens by default every 1 second.

Refresh:

* Creates new segment
* Makes data visible
* Does NOT fsync to disk

---

### Q2: Difference between refresh, flush, merge?

Refresh → Makes data searchable
Flush → Writes translog to disk
Merge → Combines segments

---

### Q3: How do you design ES for 1 billion documents?

Answer structure:

1. Estimate data size
2. Decide shard count
3. Use rollover
4. Use ILM
5. Use routing if tenant-based
6. Avoid nested unless required
7. Use filter context
8. Benchmark with realistic data

---

### Q4: Why not use Elasticsearch as primary DB?

* No strong transactions
* No joins
* Eventual consistency
* Not ACID
* Risk of data loss if misconfigured

This answer is mandatory in BFSI interviews.

---

# 🌍 7️⃣ Middle East Interview Expectations

In UAE/Qatar, they expect:

✔ Multi-region architecture
✔ Cross-cluster replication
✔ Disaster recovery
✔ Snapshot strategy
✔ Security hardening

Example:

Primary cluster → Dubai
Follower cluster → Qatar
CCR enabled
Daily snapshot to S3

---

# 🔐 8️⃣ BFSI Security Model

* TLS on transport & HTTP
* RBAC roles
* Field-level security (mask PAN numbers)
* API keys for services
* Audit logging enabled
* Encrypted snapshot repository

If you say this confidently → You pass.

---

# 🚀 9️⃣ Advanced Topics (Bonus)

* Vector search (semantic search)
* Hybrid search (BM25 + kNN)
* Runtime fields
* Parent-child vs nested trade-offs
* Transform API for entity-centric indexing

Mentioning these gives you senior edge.

---

# 🏁 Final: How YOU Should Position This in Interviews

Surya, based on your profile (React + Node + Kafka + BFSI experience):

Say this:

> In our architecture, Elasticsearch is used as a secondary search and analytics engine. Core data resides in PostgreSQL, events are published via Kafka, and Elasticsearch is asynchronously indexed for fast search, fraud detection, and analytics. We use ILM, proper shard sizing, RBAC, and snapshot strategies to ensure production reliability.

That answer alone sounds architect-level.

---



---

# 🏦 1️⃣ Complete BFSI Elasticsearch System Design

## 🎯 Use Case

Design a **Banking Transaction Search + Fraud Monitoring System**

Requirements:

* 10M transactions per day
* Search by account/date/amount/status
* Near real-time (< 2 sec indexing)
* 7-year retention (compliance)
* Role-based access (RBAC)
* DR in secondary region
* Must not affect core banking DB

---

# 🏗 2️⃣ High-Level Architecture (HLD)

![Image](https://www.dragon1.com/images/ibm-reference-soa-for-banking.png)

![Image](https://api.contentstack.io/v2/uploads/5988842f4558bdc02f8c3ed6/download?uid=blt4d3f03fcb8aa292d)

![Image](https://www.researchgate.net/publication/324959098/figure/fig2/AS%3A625125602897922%401526052966246/The-new-FX-Core-microservice-architecture.png)

![Image](https://media.licdn.com/dms/image/v2/D4E12AQE4PG3kTiG_kw/article-cover_image-shrink_600_2000/article-cover_image-shrink_600_2000/0/1714343562565?e=2147483647\&t=iZaLVNpBq9rjqEQJbj9E2tY1ODI6MEKgGosjFFcFymU\&v=beta)

### 🧠 Flow

Core Banking DB (Postgres/Oracle)
↓
CDC / Transaction Event
↓
Kafka (event streaming)
↓
Fraud Service + Search Indexer
↓
Elasticsearch Cluster
↓
API Layer (Node/NestJS)
↓
Frontend / Admin Portal

---

## 🔥 Key Principles

✔ Elasticsearch is NOT source of truth
✔ Async indexing via Kafka
✔ Immutable transaction records
✔ Multi-region replication
✔ Strict RBAC

---

# 📐 3️⃣ Low-Level Design (LLD)

## 🔹 Index Strategy

Index pattern:

```
transactions-2026-01
transactions-2026-02
```

Use:

* Data Streams
* Rollover API
* ILM

---

## 🔹 Mapping Design

```json
{
  "mappings": {
    "properties": {
      "txn_id": { "type": "keyword" },
      "account_id": { "type": "keyword" },
      "amount": { "type": "double" },
      "currency": { "type": "keyword" },
      "txn_type": { "type": "keyword" },
      "txn_status": { "type": "keyword" },
      "txn_date": { "type": "date" },
      "geo_location": { "type": "geo_point" },
      "description": { "type": "text" }
    }
  }
}
```

Why keyword?
→ Exact match filtering
→ Aggregations

Why geo_point?
→ Geo fraud detection

---

## 🔹 Fraud Detection Query Example

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "account_id": "A123" }},
        { "range": { "txn_date": { "gte": "now-5m" }}}
      ]
    }
  },
  "aggs": {
    "unique_countries": {
      "cardinality": {
        "field": "geo_location"
      }
    }
  }
}
```

If > 2 countries in 5 mins → Alert

---

# ⚙ 4️⃣ Elasticsearch + Kafka + Redis Architecture

![Image](https://api.contentstack.io/v2/assets/575e4d679e7a83165490e4ff/download?uid=blt6509ea57e625eacc%3Fuid%3Dblt6509ea57e625eacc)

![Image](https://substackcdn.com/image/fetch/%24s_%21lZd6%21%2Cf_auto%2Cq_auto%3Agood%2Cfl_progressive%3Asteep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2F903484b2-8c0c-4ce9-b4ab-e967538aeb78_1972x1197.jpeg)

![Image](https://miro.medium.com/0%2Ay9Ln3IyyJlI-W7QZ.jpeg)

![Image](https://miro.medium.com/1%2AuzCx-ntRno7D__ylFa_0MQ.png)

## 🔄 Event-Driven Architecture

Core DB → CDC → Kafka Topic: `transactions`

Consumers:

1️⃣ Fraud Service
2️⃣ Search Indexer Service

Indexer Service:

* Batch 1000 records
* Bulk API to ES
* Retry on failure
* DLQ on permanent failure

---

## 🔴 Where Redis Fits

Redis is used for:

✔ Rate limiting API
✔ Session caching
✔ Fraud rule counters
✔ Idempotency keys
✔ Recent transaction cache

Example:

* Store last 10 transactions in Redis
* If mismatch with ES → investigate

---

## 🔥 Production Pattern

User → API
↓
Redis check (cache hit?)
↓
If miss → ES query
↓
Return response

Reduces ES load.

---

# 🌍 5️⃣ Multi-Region Design (UAE / Qatar Expectation)

Primary Cluster → Dubai
Follower Cluster → Qatar

Use:

* Cross-Cluster Replication (CCR)
* Daily snapshots to S3
* 15 min RPO

Failover:

* DNS switch
* Promote follower index

---

# 🔐 6️⃣ Security (Mandatory in BFSI Interviews)

* TLS enabled (transport + HTTP)
* Role-based access (search-only users)
* Field-level security (mask PAN)
* Document-level security (branch-level isolation)
* API key auth for services
* Audit logging enabled
* Encrypted snapshot repository

If you mention these → You sound banking-grade.

---

# 🧠 7️⃣ UAE / Qatar Interview-Specific Preparation

They focus on:

✔ Disaster recovery
✔ Multi-region
✔ High availability
✔ Compliance (GDPR, AML)
✔ Scalability
✔ Cost optimization

---

## Likely Questions

---

### Q1: How do you design ES for 10M transactions per day?

Answer structure:

1. Estimate daily size
2. Decide shard count
3. Use rollover
4. Use ILM
5. Use Kafka async indexing
6. Use filter context queries

---

### Q2: How do you secure Elasticsearch?

* Private VPC
* No public access
* TLS
* RBAC
* API keys
* Snapshot encryption

---

### Q3: How do you handle cluster red state?

* Check unassigned shards
* Disk watermark
* Node failure
* Allocation explain API
* Restore from snapshot if needed

---

### Q4: Why not use Elasticsearch as primary DB?

Because:

* No ACID guarantees
* Eventual consistency
* No complex joins
* Not designed for transactional workloads

In BFSI interviews, this answer is critical.

---

# 🏁 8️⃣ Final Interview Pitch (Use This)

Surya, say this:

> In our BFSI architecture, Elasticsearch acts as a distributed search and analytics layer. Core banking data is stored in Postgres, events are streamed via Kafka, indexed asynchronously into Elasticsearch using bulk operations, and cached with Redis for hot reads. We use ILM for retention, CCR for disaster recovery, RBAC for security, and snapshot strategies for compliance. This ensures low-latency search without impacting transactional systems.

That’s a senior architect answer.

---
