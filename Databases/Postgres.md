https://www.linkedin.com/pulse/postgresql-interview-questions-answers-qaisar-abbas-4hcrf/
---

# ✅ PostgreSQL Master Framework (Interview Edition)

Structured from **engine internals → query planning → scaling → cloud → compliance**.
Use this as your **one-stop interview notes** for BFSI / FinTech / Middle East.

---

## 🧠 0) Quick Mental Model (30-sec answer)

**PostgreSQL** = **ACID OLTP database** with:

* **MVCC** for concurrency (readers don’t block writers)
* **WAL** for durability + crash recovery + replication
* **Cost-based planner** for query optimization
* Strong features: **indexes, partitions, JSONB, FTS, extensions (PostGIS), HA/DR**

> In BFSI: Think of Postgres as a **durable ledger engine** + **audit-ready storage**.

---

# 1️⃣ Architecture & Internals (How Postgres Works)

## 1.1 Process Model (Process-per-connection)

**Definition:** Postgres uses a **postmaster** that spawns **one backend process per client connection**.

### Key Processes

| Process                         | Definition                          | Why it exists                             | Typical symptom if overloaded         |
| ------------------------------- | ----------------------------------- | ----------------------------------------- | ------------------------------------- |
| **Postmaster**                  | Parent process that manages cluster | Controls startup/shutdown, forks backends | Cannot accept new connections         |
| **Backend**                     | One process per connection          | Executes SQL, manages transactions        | Too many connections → memory/CPU hit |
| **WAL writer**                  | Flushes WAL buffers                 | Durability throughput                     | WAL bottleneck, lag                   |
| **Checkpointer**                | Performs checkpoints                | Limits crash recovery time                | IO spikes if mis-tuned                |
| **Background writer**           | Writes dirty buffers gradually      | Smooth IO vs burst                        | sudden write storms                   |
| **Autovacuum launcher/workers** | Vacuum/analyze automatically        | Prevent bloat + xid wraparound            | bloat + slowdown + wraparound risk    |

✅ **Use cases:** Banking APIs, high concurrency OLTP
✅ **Best practices:**

* Use **PgBouncer** (pooling) to avoid thousands of backend processes
* Keep `max_connections` sane; prefer pooling
* Monitor `pg_stat_activity`, connection spikes

---

## 1.2 Memory Architecture (Must-know knobs)

| Component                | Definition                     | Why it matters                                          |
| ------------------------ | ------------------------------ | ------------------------------------------------------- |
| **shared_buffers**       | In-memory cache for data pages | Too low → disk reads; too high can reduce OS cache      |
| **work_mem**             | Memory per sort/hash node      | Too low → disk spills; too high → OOM under concurrency |
| **maintenance_work_mem** | Memory for VACUUM/CREATE INDEX | Index builds & vacuum speed                             |
| **wal_buffers**          | Buffers for WAL records        | Helps high write throughput                             |

✅ **Best practice:** Work_mem is **per operation**, not per query. Be careful at high concurrency.

---

## 1.3 MVCC (Multi-Version Concurrency Control)

**Definition:** Postgres maintains **multiple row versions** so readers and writers don’t block each other.

* UPDATE/DELETE create new versions and mark old versions dead
* Row visibility is decided by **transaction snapshots** (xmin/xmax)

✅ **Use case:** Payments ledger with high parallel reads/writes
✅ **Best practice:**

* Keep transactions **short**
* Avoid long-running transactions (they prevent vacuum cleanup)
* Track bloat and autovacuum effectiveness

---

## 1.4 VACUUM vs AUTOVACUUM vs ANALYZE

* **VACUUM**: removes dead tuples (makes space reusable), prevents wraparound via freezing
* **AUTOVACUUM**: background automation of vacuum + analyze
* **ANALYZE**: updates planner statistics (row counts, distributions)

✅ **If autovacuum doesn’t run:**

* table/index bloat
* query plans degrade
* transaction ID wraparound risk (serious)

---

## 1.5 WAL (Write-Ahead Logging)

**Definition:** Changes are written to **WAL first**, then data pages later. Ensures **durability**.

WAL enables:

* crash recovery
* streaming replication
* PITR (point-in-time recovery)

✅ **BFSI angle:** WAL is core to **durable money movement**
✅ **Best practice:** Monitor WAL volume, checkpoint frequency, replication lag.

---

## 1.6 Checkpoints

**Definition:** A checkpoint forces dirty pages to disk and records a consistent recovery point.

* Too frequent → IO spikes, lower throughput
* Too infrequent → long crash recovery, huge WAL

✅ **Best practice knobs:**

* `checkpoint_timeout`
* `max_wal_size`
* `checkpoint_completion_target`

---

## 1.7 Transaction IDs & Wraparound (Senior-level)

**Definition:** Postgres uses 32-bit transaction IDs. If old tuples aren’t vacuumed/frozen, it risks wraparound.

✅ **Best practice:** Never disable autovacuum; monitor freezing warnings.

---

# 2️⃣ SQL Core (Fundamentals Interviewers Expect)

## 2.1 Data Types (BFSI Tip)

| Type            | Definition     | Use case                                  |
| --------------- | -------------- | ----------------------------------------- |
| `numeric(p,s)`  | exact decimal  | money, interest                           |
| `bigint`        | 64-bit integer | identifiers, counters                     |
| `uuid`          | distributed ID | multi-service uniqueness                  |
| `timestamp(tz)` | time           | tx timestamps                             |
| `jsonb`         | binary JSON    | metadata, events (not core money columns) |
| arrays          | list type      | tags, small sets                          |

✅ **BFSI rule:** Never store money in float/double.

---

## 2.2 Schemas

**Definition:** Namespace inside a database.

✅ Use case: multi-tenant isolation (`tenant_a.*`, `tenant_b.*`)
✅ Best practice: RLS is usually safer than schema-per-tenant at huge scale (depends on org).

---

## 2.3 Sequences / Identity

✅ Prefer:

* `GENERATED ALWAYS AS IDENTITY` (modern)
* Use UUID for distributed shards

---

## 2.4 NULL semantics

* NULL means “unknown”, not empty/0
* Use `IS NULL`, `COALESCE`

---

# 3️⃣ Querying & Joins (Execution Reality)

## 3.1 EXISTS vs IN

* `EXISTS` stops early (semi-join style) → often better for correlated checks
* `IN (subquery)` can be fine too, but big sets may cost more

---

## 3.2 LATERAL JOIN (Interview-winning)

**Definition:** Lets a subquery reference columns from the left table.

✅ Use case: “top N items per group”, nearest locations per order, dynamic subquery per row.

---

# 4️⃣ Aggregation & Window Functions

## 4.1 GROUP BY + HAVING

**Use case:** reporting, reconciliation, merchant settlement summaries.

---

## 4.2 FILTER Clause

Example:

```sql
COUNT(*) FILTER (WHERE status='SUCCESS')
```

✅ Cleaner than multiple CASE expressions.

---

## 4.3 Window Functions

**Definition:** Compute analytics over a “window” without collapsing rows.

Common:

* `ROW_NUMBER`, `RANK`, `DENSE_RANK`
* `LAG`, `LEAD`
* Running totals with `SUM() OVER(...)`

✅ BFSI: anomaly detection, balance evolution, velocity checks
✅ Best practice: index partition keys when large.

---

# 5️⃣ CTEs & Subqueries (Important nuance)

## 5.1 CTE (WITH)

**Definition:** Named subquery for readability.

⚠ **Important nuance:** In modern Postgres, CTEs are **not always materialized**; planner may inline them unless forced.

* `MATERIALIZED` forces materialization
* `NOT MATERIALIZED` encourages inlining

✅ Best practice: Use CTE for clarity; verify plan with EXPLAIN.

---

## 5.2 Recursive CTE

✅ Use cases: org charts, category trees, graph-like traversals.

---

# 6️⃣ Indexing (Choosing the right index)

## 6.1 Index Types

| Index       | Definition              | Best for                      | Not great for                            |
| ----------- | ----------------------- | ----------------------------- | ---------------------------------------- |
| **B-Tree**  | ordered index           | equality/range                | deep JSON contains                       |
| **Hash**    | hash lookup             | equality only                 | ranges, multi-purpose                    |
| **GIN**     | inverted index          | JSONB @>, array contains, FTS | high update rate can be heavy            |
| **GiST**    | generalized search tree | PostGIS, ranges, similarity   | pure containment at extreme scale vs GIN |
| **SP-GiST** | space-partitioned       | specific shapes / tries       | not default choice                       |
| **BRIN**    | block-range             | huge append-only time tables  | random data                              |

---

## 6.2 Partial Index

**Definition:** Index only subset of rows.

✅ Use case: only index “active=true” or “status='PENDING'”.

---

## 6.3 Expression Index

✅ Use case: `lower(email)` for case-insensitive search.

---

## 6.4 Covering Index (INCLUDE)

✅ Helps index-only scans by storing extra columns.

---

## 6.5 Index-only Scan (Senior topic)

**Definition:** Query can be answered from index without table fetch **if visibility map allows**.

✅ Best practice: vacuum helps visibility map → more index-only scans.

---

## 6.6 Why Index Not Used?

* low selectivity
* outdated statistics
* function/operator mismatch
* small table (seq scan cheaper)
* parameter sniffing / generic plan effects

---

# 7️⃣ Transactions, Concurrency & Locking

## 7.1 ACID in Postgres

* Atomicity: txn boundaries
* Consistency: constraints
* Isolation: MVCC + locks
* Durability: WAL + fsync

---

## 7.2 Isolation Levels

| Level           | What you get                          | BFSI fit              |
| --------------- | ------------------------------------- | --------------------- |
| Read Committed  | default, fresh snapshot per statement | typical APIs          |
| Repeatable Read | consistent snapshot per transaction   | long reads/reporting  |
| Serializable    | strongest (prevents anomalies)        | critical ledger paths |

**Serializable in Postgres:** implemented via **SSI (Serializable Snapshot Isolation)** which detects dangerous structures and may abort transactions.

✅ Interview line: “Serializable may cause retries; app must handle retry logic.”

---

## 7.3 Locks

* Row locks: `SELECT ... FOR UPDATE`
* Table locks: DDL, heavy operations
* Advisory locks: app-controlled mutex

✅ BFSI use cases: settlement job single runner, idempotency coordination.

---

## 7.4 Deadlocks

**Definition:** Two txns hold locks each other needs.

✅ Fix:

* lock resources in consistent order
* keep transactions short
* index FK columns (reduces lock scans)

---

## 7.5 SAVEPOINT

**Definition:** nested rollback point inside a transaction.

✅ Use case: multi-step business flow where partial rollback needed.

---

# 8️⃣ Query Optimization & Performance Debugging

## 8.1 EXPLAIN vs EXPLAIN ANALYZE

* EXPLAIN: estimated plan
* EXPLAIN ANALYZE: executes and shows actual timing/rows

✅ Interview flow:

1. Identify expensive node
2. Compare estimated vs actual rows
3. Fix stats/index/query shape
4. Re-check plan

---

## 8.2 Scans & Joins (Know these terms)

* Seq scan
* Index scan
* Bitmap index + bitmap heap scan
* Nested loop join
* Hash join
* Merge join

✅ Rule of thumb:

* Nested loop: great for small outer + indexed inner
* Hash join: good for large equality joins
* Merge join: good when both sides sorted/indexed

---

## 8.3 Statistics

Planner uses stats from `ANALYZE` (`pg_statistic` internally).

✅ Best practice: analyze after bulk loads; watch skewed distributions.

---

## 8.4 Bloat Management

Detect via `pg_stat_user_tables` + bloat tools; symptoms:

* increasing table size
* slower reads
* more IO

Fix options:

* VACUUM (reclaim internal space)
* VACUUM FULL (rewrites table; blocks)
* REINDEX (fix index bloat)
* Partitioning (prevention)

---

# 9️⃣ Partitioning & Sharding

## 9.1 Partitioning

**Definition:** Split table into child tables by key (range/list/hash).

✅ Benefits:

* pruning
* smaller indexes
* easier retention (detach old partitions)

✅ Best practice:

* choose partition key based on query patterns (time, tenant, region)
* keep partitions manageable (not too many)

---

## 9.2 Partition Pruning

Planner eliminates partitions not needed for query.

✅ Best practice: ensure queries include partition key predicates.

---

## 9.3 Sharding (Beyond one node)

**Definition:** split data across multiple Postgres clusters/nodes.

Options:

* Citus
* app-level sharding

✅ BFSI sharding keys:

* tenant
* region
* account range/hash

---

# 🔟 Replication, HA & DR

## 10.1 Physical vs Logical Replication

| Physical          | Logical                    |
| ----------------- | -------------------------- |
| byte-level        | row/table changes          |
| HA, read replicas | migrations, selective sync |
| whole cluster     | chosen tables              |

---

## 10.2 Synchronous vs Asynchronous

* Sync: strong durability, higher latency
* Async: lower latency, risk of lag

✅ BFSI typical: synchronous within region + async cross-region.

---

## 10.3 Replication Slots

**Definition:** Prevent WAL cleanup until replica/consumer catches up.

✅ Best practice: monitor slots; stuck slots → WAL disk explosion.

---

## 10.4 PITR (Point-in-Time Recovery)

Needs:

* base backup
* WAL archiving
* restore to timestamp/LSN

✅ BFSI: define RPO/RTO and test DR drills.

---

# 1️⃣1️⃣ Security & Compliance (BFSI Grade)

## 11.1 Roles/Privileges

* GRANT/REVOKE
* least privilege
* separation of duties

---

## 11.2 pg_hba.conf

**Definition:** Host-based auth rules: who can connect from where, using what auth.

✅ Best practice:

* restrict CIDRs
* enforce TLS
* use strong auth (SCRAM)

---

## 11.3 Row-Level Security (RLS)

**Definition:** DB-enforced per-row access policies.

✅ Use case: multi-tenant banking: user sees only own accounts.

---

## 11.4 Encryption

* In transit: TLS
* At rest: disk encryption / cloud KMS
* Column-level: `pgcrypto` (careful with indexing/search)

---

## 11.5 Auditing

Options:

* `pgaudit`
* trigger-based audit logs
* WAL archiving retention

✅ BFSI best practice: immutable audit logs + secure retention + monitoring.

---

# 1️⃣2️⃣ Extensions (Know the “why”)

| Extension          | Definition                  | Use case                   |
| ------------------ | --------------------------- | -------------------------- |
| pg_stat_statements | query performance history   | top slow queries           |
| PostGIS            | geo types + spatial indexes | ride matching, real estate |
| pg_trgm            | trigram similarity          | fuzzy search               |
| pgcrypto           | crypto functions            | column encryption/hashing  |
| TimescaleDB        | time-series layer           | metrics, events            |
| Citus              | distributed Postgres        | sharding scale             |

---

# 1️⃣3️⃣ Administration & Monitoring

## 13.1 Tools

* `pg_dump/pg_restore` (logical backups)
* `pg_basebackup` (physical)
* `pg_upgrade` (major upgrades)
* PgBouncer (pooling)

---

## 13.2 Monitoring Views

| View                   | Why it matters             |
| ---------------------- | -------------------------- |
| pg_stat_activity       | current queries/sessions   |
| pg_locks               | lock contention            |
| pg_stat_user_tables    | vacuum/analyze/bloat hints |
| pg_stat_statements     | long-term query profiling  |
| pg_indexes / pg_tables | metadata                   |

---

# 1️⃣4️⃣ Advanced Topics (Often asked in senior rounds)

## 14.1 FDW (Foreign Data Wrapper)

**Definition:** Query external databases as tables.

✅ Use case: reporting across systems without ETL (use cautiously).

---

## 14.2 LISTEN/NOTIFY

**Definition:** lightweight pub-sub notifications inside Postgres.

✅ Use case: internal event triggers, cache invalidation (not heavy streaming).

---

## 14.3 Advisory Locks

App-defined locks: `pg_advisory_lock(key)`.

✅ Use case: singleton jobs (settlement, reconciliation).

---

## 14.4 Generated Columns

**Definition:** computed column stored or virtual.

✅ Use case: computed fields for indexing/search.

---

# 🏦 BFSI Ledger Blueprint (Pure PostgreSQL)

## Tables (minimal)

```text
accounts
transactions
ledger_entries
idempotency_keys
audit_log
```

## Rules (must say in interview)

* Ledger is **append-only** (no update/delete)
* Balance = **sum(ledger_entries)** or cached snapshot
* Each transaction is **atomic** and **idempotent**
* Enforce **double-entry**: debit + credit = 0

✅ Enforcements:

* constraints
* unique keys
* serializable / retries
* `SELECT FOR UPDATE` where needed
* audit log triggers

---

# 🚨 Real-time Fraud Detection Pipeline with Postgres (Interview Ready)

## Architecture

1. Write transaction to Postgres ledger
2. Capture changes via **logical decoding / Debezium**
3. Publish to Kafka/Kinesis
4. Fraud engine consumes stream:

   * velocity checks
   * geo anomalies
   * device fingerprint scoring
   * rules + ML model
5. Write decision back to Postgres (flag/hold/review)

✅ Why not do fraud fully in Postgres?

* streaming + ML needs dedicated compute
* Postgres stays authoritative OLTP store

---

# 🗺 PostGIS at Scale (Millions of GIS rows)

✅ Best practices:

* Use `GiST` index on geometry/geography
* Use **partitioning by region + time**
* Keep “live driver location” in Redis, persist snapshots to Postgres
* Use Postgres for **durable state**, Redis for **high-frequency updates**

---

# ✅ Missing Items I Added (from your draft)

* Memory internals (shared_buffers/work_mem/etc.)
* CTE materialization nuance (MATERIALIZED/NOT MATERIALIZED)
* Serializable isolation implemented via SSI + retry requirement
* Replication slots risk (WAL growth)
* Visibility map role in index-only scans
* FDW/LISTEN-NOTIFY/advisory locks definitions + safe usage

---
```markdown
# ✅ PostgreSQL Interview Master Question Bank — 100 Q&A (Senior / Architect / BFSI)


---

## 🟢 A) Core SQL & Fundamentals (1–15)

### 1) What is MVCC and how does PostgreSQL implement it?
**MVCC (Multi-Version Concurrency Control)** keeps multiple versions of a row so **readers don’t block writers** and **writers don’t block readers**.  
Postgres stores transaction visibility using row metadata (`xmin/xmax`) and each statement/transaction reads from a consistent **snapshot**. Old versions become **dead tuples** and are cleaned by VACUUM.

### 2) Difference between JSON and JSONB?
`JSON` stores text as-is (preserves formatting) and is slower to query.  
`JSONB` stores a binary representation, supports indexing well (GIN), and is generally preferred for querying/filtering.  
Use `JSONB` for app metadata/events; avoid for core transactional columns when relational columns fit.

### 3) When do you use numeric vs float?
Use `numeric(p,s)` for **money** (exact precision).  
`float/double` are approximate → rounding issues.  
BFSI rule: never store currency amounts in float.

### 4) What is a sequence?
A **sequence** generates unique numeric values (often for IDs).  
Modern preferred approach: `GENERATED ALWAYS AS IDENTITY`.  
In distributed/sharded systems, UUIDs or shard-local sequences reduce contention.

### 5) What happens if you don’t vacuum?
Dead tuples accumulate → **table bloat + index bloat**, worse plans, more IO.  
Also risk **transaction ID wraparound**, which can lead to forced shutdown to protect data integrity.  
Vacuum is not optional at scale.

### 6) Explain isolation levels in PostgreSQL.
- **Read Committed (default):** snapshot per statement  
- **Repeatable Read:** snapshot per transaction  
- **Serializable:** strongest; prevents anomalies via **SSI** and can abort transactions requiring retries  
Higher isolation can reduce anomalies but increases retries/latency under contention.

### 7) What is a correlated subquery?
A subquery that references columns from the outer query.  
Example: “exists a payment for this customer”.  
Can be efficient with `EXISTS` + proper indexes.

### 8) Difference between EXISTS and IN?
`EXISTS` stops after first match and is often better for correlated checks.  
`IN (subquery)` may materialize/scan a larger set depending on plan.  
Always validate with `EXPLAIN (ANALYZE)`.

### 9) What is a LATERAL join?
`LATERAL` allows a subquery in FROM to reference columns from the left side.  
Useful for “top N per group”, nearest locations, per-row computed sets.  
Often cleaner than complex correlated subqueries.

### 10) What is an index-only scan?
Planner can answer query from index **without hitting heap** if visibility map indicates tuples are visible.  
Vacuum improves visibility map → enables more index-only scans.  
Good for read-heavy workloads.

### 11) How does GROUP BY work internally?
Planner groups rows using either **HashAggregate** (build hash table) or **GroupAggregate** (sorted input).  
Choice depends on data size, memory, and sort availability.  
Tune `work_mem` to avoid disk spills for hash/sorts.

### 12) What are window functions?
They compute values over a “window” of rows without collapsing the result set.  
Examples: ranking, running totals, lag/lead.  
Great for analytics, fraud patterns, ledger running balance checks.

### 13) What is a CTE and when is it materialized?
A **CTE (`WITH`)** is a named subquery for readability.  
In modern Postgres, CTEs may be **inlined** unless marked `MATERIALIZED`.  
Use `MATERIALIZED` when reuse is beneficial; otherwise allow optimizer freedom.

### 14) Difference between INNER and LEFT join?
**INNER:** returns only matching rows.  
**LEFT:** keeps all left rows and fills missing right side with NULLs.  
LEFT join is common in reporting; INNER in strict relationships.

### 15) What is a CHECK constraint?
Validates a condition on rows (e.g., amount > 0).  
Enforces consistency at DB layer; critical for BFSI invariants (no negative balances if rule requires).

---

## 🟡 B) Indexing & Query Optimization (16–35)

### 16) When do you use GIN vs GiST?
- **GIN:** containment/search on `JSONB @>`, arrays, full-text (`tsvector`)  
- **GiST:** geometric/range/spatial (PostGIS), similarity, R-tree like behavior  
GIN is usually faster for containment but heavier on writes.

### 17) What is a BRIN index and when is it useful?
BRIN stores min/max summaries per block range.  
Great for **huge append-only** tables where data is naturally ordered (time-series).  
Tiny index size; not ideal for random distribution.

### 18) Why might PostgreSQL ignore an index?
- Low selectivity (too many rows matched)  
- Outdated stats (needs ANALYZE)  
- Function/operator mismatch (no usable index operator class)  
- Small table (seq scan cheaper)  
- Parameterized generic plan effects

### 19) How does the cost-based optimizer work?
Postgres estimates costs using statistics, IO/CPU assumptions, and chooses plan with lowest estimated cost.  
Wrong estimates → wrong plan → fix via ANALYZE, better indexing, or query rewrite.

### 20) What is selectivity?
Fraction of rows matched by a predicate.  
High selectivity (few rows) → index scan likely.  
Low selectivity (many rows) → seq scan/bitmap likely.

### 21) What is a bitmap heap scan?
Planner uses an index to build a bitmap of matching heap pages, then fetches heap pages efficiently.  
Used when many rows match but index scan would be too random-IO heavy.

### 22) How do you debug a slow query using EXPLAIN ANALYZE?
1) Run `EXPLAIN (ANALYZE, BUFFERS)`  
2) Find highest cost/time nodes  
3) Compare estimated vs actual rows  
4) Fix stats/indexes/query shape  
5) Re-check plan; validate using production-like data

### 23) What does EXPLAIN ANALYZE show?
Actual execution time, actual row counts, and real plan path.  
It proves whether the plan is good and where time is spent.  
Add `BUFFERS` to see IO vs CPU behavior.

### 24) What is effective_cache_size?
Planner hint estimating how much OS+DB cache is available for data.  
It doesn’t allocate memory; affects plan choice (index vs seq scan likelihood).

### 25) What is work_mem used for?
Memory per sort or hash operation.  
Too low → spills to disk (slow).  
Too high at high concurrency → memory blowups. Tune carefully.

### 26) How to detect index bloat?
Symptoms: index size grows faster than table, slower reads.  
Use `pg_stat_user_indexes`, `pg_relation_size`, and bloat estimation queries/tools.  
Fix: `REINDEX` or recreate index; prevent via autovacuum tuning + fillfactor.

### 27) What is a covering index?
An index that includes all columns needed by a query, enabling index-only scans.  
In Postgres: `CREATE INDEX ... INCLUDE (col3, col4)`.

### 28) What are expression indexes?
Indexes on computed expressions, e.g., `lower(email)`.  
Useful for case-insensitive lookup and derived predicates.

### 29) How does partition pruning work?
Planner removes irrelevant partitions using partition key predicates.  
Best when query includes constraints on partition key; otherwise may scan many partitions.

### 30) What is query parallelism?
Postgres can parallelize scans/aggregations/joins under conditions.  
Controlled by `max_parallel_workers_per_gather`, table size thresholds, and plan suitability.  
OLTP often benefits less than analytics.

### 31) When does PostgreSQL use hash join?
Typically for large equality joins when building a hash table is cheaper than nested loops.  
Needs enough memory; otherwise may batch/spill.

### 32) Nested loop vs merge join difference?
- **Nested loop:** best when outer is small and inner has index  
- **Merge join:** best when both sides sorted (or can be sorted cheaply) and join is equality/range-friendly

### 33) How do statistics affect performance?
Planner estimates rows and costs from stats.  
Bad stats → wrong plan choices (e.g., seq scan instead of index).  
Run `ANALYZE` after bulk loads; consider extended statistics for correlated columns.

### 34) What is ANALYZE?
Collects statistics about table data distribution and row counts.  
Improves query plans; critical after large data changes.

### 35) How do you tune autovacuum?
Adjust based on table size and churn:
- lower `autovacuum_vacuum_scale_factor` for hot tables  
- increase `autovacuum_max_workers`  
- raise `autovacuum_vacuum_cost_limit` if IO allows  
- ensure `autovacuum_naptime` reasonable  
Monitor: vacuum lag, dead tuples, bloat.

---

## 🔴 C) Concurrency, WAL, Recovery (36–55)

### 36) What causes deadlocks?
Two transactions lock resources in opposite order and wait on each other.  
Postgres detects and aborts one transaction.

### 37) How does PostgreSQL detect deadlocks?
A deadlock detector periodically checks lock waits and builds wait-for graph; if cycle found, aborts a victim transaction.

### 38) Row-level lock vs advisory lock?
- **Row lock:** enforced by DB on rows/tables; automatic semantics  
- **Advisory lock:** app-controlled mutex using keys; not tied to rows  
Use advisory locks for singleton jobs (settlement) or external coordination.

### 39) What is SELECT FOR UPDATE?
Acquires row lock so others can’t modify (and sometimes can’t lock) those rows until commit.  
Used in balance updates, maker-checker state transitions, inventory/payment reservation.

### 40) How does Serializable isolation work in Postgres?
Uses **SSI (Serializable Snapshot Isolation)** to detect dangerous conflict patterns.  
It may abort transactions with serialization failures → app must retry.  
Great for correctness in ledger flows.

### 41) What are phantom reads?
A transaction re-runs a query and sees new rows inserted by another transaction.  
Serializable isolation prevents this anomaly.

### 42) How does MVCC avoid read locks?
Reads use snapshots and check row visibility without locking rows.  
Writers create new versions instead of overwriting, so readers remain non-blocking.

### 43) Risks of long-running transactions?
- prevents vacuum from cleaning dead tuples  
- causes bloat  
- increases replication lag (logical decoding may stall)  
- holds locks longer  
Keep transactions short.

### 44) What is vacuum freeze?
Freezing marks old tuples as permanently visible to avoid xid wraparound.  
Critical for long-lived clusters.

### 45) What is xid wraparound?
Transaction IDs are finite (32-bit). If tuples aren’t vacuumed/frozen, Postgres risks interpreting very old xids as “in the future”.  
Postgres forces aggressive vacuum or shutdown to protect correctness.

### 46) What is WAL?
Write-Ahead Log: records changes before data pages.  
Enables durability, crash recovery, replication, PITR.

### 47) How does crash recovery work?
On restart, Postgres replays WAL from last checkpoint to restore consistent state.  
WAL ensures committed transactions persist; uncommitted are rolled back via MVCC.

### 48) What is synchronous_commit?
Controls whether transaction waits for WAL flush on commit.  
- `on`: strongest durability  
- `off`: faster but risk losing last few transactions on crash  
BFSI typically prefers durability; optimize elsewhere first.

### 49) What is fsync?
Forces OS to flush writes to durable storage.  
Critical for real durability; disabling it is unsafe except in dev/testing.

### 50) What happens during a checkpoint?
Dirty buffers are flushed, and a checkpoint record is written.  
Reduces recovery time but causes IO. Tune to avoid spikes.

### 51) How do replication slots work?
Slots prevent WAL removal until replica/consumer confirms receipt.  
Great for logical decoding/replication but dangerous if consumer stalls → WAL disk fills.

### 52) What is replication lag?
Delay between primary WAL generation and replica replay.  
Caused by network, IO limits, heavy writes, slow replicas, long transactions.

### 53) Physical vs logical replication?
- **Physical:** byte-level, whole cluster, HA/read replicas  
- **Logical:** table-level changes, migrations/selective replication, CDC streams

### 54) What is hot standby?
A replica that can accept read queries while replaying WAL.  
Used for read scaling and failover readiness.

### 55) How do you prevent double payments?
Combine:
- idempotency key table + unique constraint  
- unique transaction reference  
- serializable / row locks for critical state  
- outbox pattern for exactly-once event publishing

---

## 🟣 D) Scaling, HA, DR, Operations (56–75)

### 56) How do you design HA in Postgres?
Primary + synchronous replica (same region/AZ) + async cross-region replica.  
Use automated failover (Patroni/repmgr/cloud).  
Include PgBouncer, backups, monitoring, and failover drills.

### 57) How to set up streaming replication?
Take base backup (`pg_basebackup`), configure `primary_conninfo`, enable WAL settings, start replica.  
In managed cloud, use built-in replication and read replicas.

### 58) How does PITR work?
Restore base backup, then replay archived WAL up to a timestamp/LSN.  
Requires WAL archiving configured continuously. Used for “undo to point before incident”.

### 59) What is RPO vs RTO?
- **RPO:** acceptable data loss window  
- **RTO:** acceptable downtime window  
Banking often targets very low RPO and tight RTO.

### 60) How do you shard Postgres?
- Citus (distributed Postgres)  
- App-level sharding by tenant/account hash  
Keep shards independent; avoid cross-shard transactions if possible.

### 61) What is Citus?
Extension that distributes tables across worker nodes using a distribution key.  
Coordinator plans queries; workers store shards.  
Good for scale-out OLTP and analytics, with tradeoffs on cross-shard joins.

### 62) How do you scale read traffic in Postgres?
Read replicas + load balancing, query routing, caching (Redis), and separating analytics workloads.  
Use logical replication to a reporting/warehouse system for heavy analytics.

### 63) When to use partitioning?
When tables are very large and queries filter on a partition key (time, tenant, region).  
Also for retention (detach old partitions) and vacuum/index efficiency.

### 64) Global vs local indexes in partitioning?
Postgres primarily uses **local indexes per partition**.  
Global indexes are not a native general feature; design queries/keys accordingly.

### 65) How to design for 10M tx/day?
Append-only ledger, partition by time, careful indexing, batching, WAL tuning, PgBouncer, replicas, and separate analytics.

### 66) How to avoid write bottlenecks?
Reduce indexes, batch inserts, avoid hotspots (don’t update same row), partition, tune WAL/checkpoints, use fast storage, consider sharding.

### 67) How to reduce WAL overhead?
Minimize index updates, use `wal_compression`, tune checkpoints, avoid frequent updates, consider unlogged tables only for non-critical ephemeral data.

### 68) How do you monitor Postgres?
- `pg_stat_activity`, `pg_locks`  
- `pg_stat_statements` (top queries)  
- replication lag, WAL volume, autovacuum progress  
- CPU/IO, buffer cache hit ratio, slow query logs

### 69) What is pg_stat_statements?
Extension that tracks query fingerprints, timings, calls, rows, and IO stats.  
Best tool for “top N worst queries”.

### 70) What is bloat?
Wasted space due to MVCC dead tuples and index fragmentation.  
Leads to more IO and slower queries. Managed via vacuum, reindex, partitioning.

### 71) When to use materialized views?
For expensive aggregations used repeatedly where freshness can be delayed.  
Refresh periodically; avoid in high-write OLTP paths.

### 72) How to design multi-tenant DB?
Options:
- single schema + tenant_id + RLS  
- schema-per-tenant  
- database-per-tenant  
At large scale, tenant_id + RLS is common; shard by tenant for big customers.

### 73) What is connection pooling and why needed?
Postgres has process-per-connection; too many connections hurts memory/CPU.  
Poolers (PgBouncer) reuse connections and reduce overhead.

### 74) PgBouncer vs Pgpool?
- **PgBouncer:** lightweight connection pooling (most common)  
- **Pgpool:** pooling + load balancing + additional features (more complex)  
Most teams use PgBouncer + app-aware routing.

### 75) What is logical decoding?
Reading WAL changes as logical events (insert/update/delete).  
Used for CDC, Debezium, Kafka streaming, auditing pipelines.

---

## 🏦 E) BFSI, Ledger, Compliance, Security (76–100)

### 76) How do you design a tamper-proof ledger?
Append-only ledger entries + immutable constraints, no updates/deletes, use reversal entries.  
Add strong auditing, WAL retention, and access controls. Consider hashing chain per batch for extra tamper evidence.

### 77) How to enforce ACID strictly?
Use transactions, constraints, proper isolation (serializable for critical paths), WAL durability, and strict error handling with retries when needed.

### 78) How to implement maker-checker?
Use a workflow table: `PENDING -> APPROVED/REJECTED`.  
Checker action requires role-based authorization.  
Use row locks (`FOR UPDATE`) to prevent double approvals; audit every transition.

### 79) How to implement audit logging?
Options:
- trigger-based audit tables (row-level old/new)  
- `pgaudit` for statement auditing  
- CDC via logical decoding for immutable audit stream  
Secure retention + restricted access.

### 80) How to encrypt sensitive columns?
Use TLS in transit, disk/KMS at rest.  
For column-level: `pgcrypto` (but can hurt indexing/search).  
For production: tokenize + store mapping securely; encrypt only what you must.

### 81) How to implement RLS for multi-tenant banking?
Enable RLS and create policies based on `tenant_id` and session variables (`current_setting`).  
Ensures DB-level enforcement even if app bugs occur.

### 82) How to detect fraud patterns in SQL?
Use window functions for velocity/risk patterns:
- many tx in short window  
- geo-distance anomalies  
- amount deviations vs baseline  
But at scale, do real-time detection via streaming engine + Postgres as system of record.

### 83) How to design immutable tables?
Avoid UPDATE/DELETE permissions; enforce via triggers or security definer functions.  
Use reversal rows for corrections.  
Maintain strict audit trail.

### 84) How to design DR across regions?
Primary + sync replica (local) + async replica (remote).  
WAL archiving + PITR.  
Regular DR drills; defined RPO/RTO.

### 85) How to guarantee no data loss?
Use synchronous replication within region + durable commits (`synchronous_commit=on`).  
Also ensure WAL is on reliable storage; verify failover semantics.

### 86) How to detect replication lag?
Monitor:
- `pg_stat_replication` (write/flush/replay LSN difference)  
- replica replay timestamps  
Alert on lag thresholds and slot growth.

### 87) How to prevent race conditions?
Use transactions + correct isolation, row locks, unique constraints, idempotency keys, and deterministic lock ordering.

### 88) How to handle high concurrency payments?
Append-only ledger model, avoid hotspot rows, partition, and use serializable + retries only where required.  
Use idempotency and outbox for reliable events.

### 89) How to avoid hotspot rows?
Don’t update a single balance row per tx.  
Use ledger entries + async aggregates, partitioning, and shard by account/tenant for very large systems.

### 90) How to ensure GDPR compliance?
Data minimization, encryption, access logging, retention policies, and deletion/anonymization processes where legally required.  
Separate PII storage and apply strict RBAC/RLS.

### 91) How to archive old transactions?
Partition by time; detach old partitions to cheaper storage; keep read-only access in archive DB/warehouse.

### 92) How to monitor WAL growth?
Track `pg_wal` size, checkpoint frequency, archive success, and replication slot backlog.  
Stuck slots or archiver failures cause WAL explosion.

### 93) How to prevent table bloat in ledger?
Partitioning + aggressive autovacuum tuning per hot partitions.  
Keep transactions short; avoid long-running snapshots.  
Reindex or rebuild partitions periodically if needed.

### 94) How to design idempotent API storage?
Create table keyed by idempotency key with unique constraint.  
On request: insert key first; if conflict, return stored response/tx reference.  
Works for payment retries safely.

### 95) How to design balance consistency?
Balance is derived from ledger entries; optionally maintain a cached summary table updated transactionally (or async with reconciliation).  
Always keep ledger authoritative.

### 96) How to scale analytics separately?
Use logical replication/CDC to stream data into analytics store (warehouse).  
Keep OLTP cluster focused on writes + point reads.

### 97) How to isolate reporting workload?
Use read replicas or a dedicated reporting cluster.  
Consider materialized views there; avoid heavy queries on primary.

### 98) How to protect from insider threats?
Least privilege, strong auditing, separation of duties, RLS, encrypted backups, monitoring, and restricted access to prod.  
Audit all privileged actions.

### 99) How to implement disaster recovery drill?
Regularly restore from backups + WAL to a point in time.  
Validate application integrity, run reconciliation checks, measure RPO/RTO, and document gaps.

### 100) What would you improve in PostgreSQL for banking?
Better built-in global sharding primitives, richer auditing/immutability tooling, stronger native encryption/search tradeoffs, and more automated bloat/maintenance insights at very large scale.

---
```

