https://github.com/Devinterview-io/sql-interview-questions
https://www.innozant.com/microsoft-sql-server-interview-questions/
https://www.geeksforgeeks.org/sql-server/sql-server-interview-questions/

````md
# ✅ SQL Server Senior/Architect Interview Notes (Domain-Driven: BFSI / FinTech / Real Estate / E-commerce / Middle East)

Use these lines to speak like an architect:
> “In a high-volume payment system…”
> “In a banking ledger architecture…”
> “In a GDPR-compliant EU deployment…”

---

# 🏦 1) SQL Server Architecture (How You Speak in Interviews)

## 🧠 Core Engine Components (Mental Model)

- **Buffer Pool (Data Cache)** → caches 8KB pages (data + index)
- **Plan Cache** → stores compiled execution plans
- **Transaction Log (WAL)** → durability + recovery
- **TempDB** → sorting/hashing/version store/temp objects
- **Storage** → pages (8KB) → extents (64KB)
- **Concurrency** → locks (logical) + latches (memory) + row versioning (RCSI/Snapshot)

---

## 🔹 Buffer Pool
- Caches data pages (8KB each)
- Reduces disk I/O
- Critical for OLTP (banking/payments)

**Interview Line (BFSI style):**
> In payment systems, buffer pool hit ratio must be high to reduce `PAGEIOLATCH` waits.

---

## 🔹 Pages & Extents
- Page = **8KB**
- 8 pages = **1 extent (64KB)**
- Clustered index defines physical row order (B-Tree)

**Question:** Why is heap dangerous in OLTP?  
✅ Because heap updates can create **forwarded records** → more random I/O.

---

## 🔹 Transaction Log (Write-Ahead Logging)
- Every change goes **first to log**
- Log writes are **sequential** → fast
- `WRITELOG` waits = log storage bottleneck

**Banking Line:**
> Ledger must guarantee durability before commit; log throughput is a hard constraint.

---

## 🔹 TempDB
Used for:
- Sort
- Hash join
- Temp tables / table variables
- Version store (RCSI/Snapshot)

Why bottleneck:
- Metadata contention
- Allocation contention (PFS/GAM/SGAM)

---

# 🔥 2) Indexing & Query Optimization (High Weight in Interviews)

## ✅ Clustered vs Nonclustered

| Feature | Clustered | Nonclustered |
|---|---|---|
| Stores actual rows? | Yes | No (pointer to clustered key / RID) |
| Count per table | 1 | Many |
| Best for | PK, range scans | filters, lookups |

**Architect line:**
> In high-throughput OLTP, choose a narrow, stable, ever-increasing clustered key (often `BIGINT IDENTITY`) to reduce fragmentation and page splits.

---

## ✅ Covering Index (Avoid Key Lookups)

```sql
CREATE INDEX IX_Orders_Status
ON Orders(Status)
INCLUDE (Amount, CreatedDate);
````

* Avoids key lookup → fewer reads
* Ideal for read-heavy queries on hot tables

---

## ✅ Columnstore (OLAP / Reporting)

* Columnar storage
* Batch mode execution
* Great compression
* 10x analytics performance

**Real Estate example:**

> Monthly property analytics / pricing trends → Columnstore improves scan + aggregate workloads.

---

## ✅ Why SQL Server Doesn’t Use Index (Common Causes)

* Parameter sniffing
* Non-SARGable predicates
* Implicit conversions
* Outdated statistics
* Too many lookups → scan becomes cheaper

---

# 🔐 3) Concurrency (CRITICAL for BFSI)

## ✅ Isolation Levels (Interview Table)

| Level            | Dirty Reads | Non-Repeatable | Phantom |
| ---------------- | ----------- | -------------- | ------- |
| Read Uncommitted | ✅           | ✅              | ✅       |
| Read Committed   | ❌           | ✅              | ✅       |
| Snapshot         | ❌           | ❌              | ❌       |
| Serializable     | ❌           | ❌              | ❌       |

---

## ✅ RCSI vs Snapshot

| RCSI                        | Snapshot                      |
| --------------------------- | ----------------------------- |
| Statement-level consistency | Transaction-level consistency |
| Auto (if enabled)           | Explicit                      |
| Uses version store          | Uses version store            |

**BFSI line:**

> Use Snapshot for ledger reconciliation reads to avoid blocking while guaranteeing consistency.

---

## ✅ Deadlock Handling (Production Steps)

1. Capture deadlock graph (Extended Events)
2. Identify victim & competing statements
3. Reduce lock time (short transactions)
4. Ensure consistent access order
5. Add missing indexes to reduce lock footprint

---

# ⚡ 4) Performance Tuning Strategy (How You Answer)

When system is slow:

1. **Wait stats** → `sys.dm_os_wait_stats`
2. **Top CPU / reads queries** → `sys.dm_exec_query_stats`
3. **Execution plan** (actual vs estimated)
4. **Blocking chains** → `sp_who2` / locks DMVs

---

## ✅ Common Wait Types

| Wait        | Meaning               | Fix                              |
| ----------- | --------------------- | -------------------------------- |
| CXPACKET    | Parallelism imbalance | Tune MAXDOP / cost threshold     |
| PAGEIOLATCH | Slow data file I/O    | Improve storage / memory         |
| WRITELOG    | Log write bottleneck  | Faster log disk / tune commits   |
| LCK_M_X     | Blocking              | Tune indexes + transaction scope |

---

# 🏦 5) HA & DR (Mandatory for UAE / EU)

## ✅ Always On Availability Groups

* Multiple replicas
* Automatic failover (with quorum)
* Readable secondaries (read scale)

---

## ✅ AG vs FCI

| AG                     | FCI                          |
| ---------------------- | ---------------------------- |
| Database-level         | Instance-level               |
| Read replicas possible | No read replica              |
| Requires Enterprise    | Standard supported (limited) |

---

## ✅ RPO / RTO (Interview Gold Line)

> In BFSI, RPO should be near-zero and RTO typically 1–5 minutes depending on SLA.

---

# 🔒 6) Security (GDPR / BFSI)

| Feature          | Protects                               |
| ---------------- | -------------------------------------- |
| TDE              | Data at rest                           |
| Always Encrypted | Column-level (client-side keys)        |
| RLS              | Row filtering (multi-tenant isolation) |
| DDM              | Masking                                |
| Audit            | Access tracking                        |

---

## ✅ TDE vs Always Encrypted

| TDE                   | Always Encrypted  |
| --------------------- | ----------------- |
| Server-side           | Client-side       |
| Protects disk/backups | Protects from DBA |
| Transparent           | Key mgmt required |

**FinTech example:**

> Card/PII columns → Always Encrypted or tokenization.

---

# 📦 7) Partitioning (1B+ Rows)

* Range partition by date (monthly/weekly)
* Partition elimination improves reads
* Switch partition for archive (fast delete)

**Interview line:**

> For 1B-row transaction table, partition monthly and align indexes to keep maintenance predictable.

---

# ☁ 8) Azure SQL DB vs Managed Instance

| Azure SQL DB     | Managed Instance     |
| ---------------- | -------------------- |
| Pure PaaS        | Near full SQL Server |
| Limited cross-db | Supports cross-db    |
| Serverless tier  | No serverless        |

---

# 🔄 9) CDC vs Change Tracking

| CDC               | Change Tracking          |
| ----------------- | ------------------------ |
| Full changed data | Only metadata            |
| Heavier           | Lightweight              |
| ETL pipelines     | Sync / incremental reads |

---

# 🧠 10) Speak Like an Architect (Upgrade Answers)

Instead of:

> “Clustered index stores data.”

Say:

> “In high-throughput payments, I prefer a narrow sequential clustered key to reduce fragmentation and keep inserts efficient.”

---

# 🎯 What You Must Master (Surya – BFSI Profile)

✔ Snapshot isolation + RCSI
✔ Always On AG + DR strategy
✔ Query Store + regression handling
✔ Parameter sniffing fixes
✔ Partitioning + switching
✔ TDE vs Always Encrypted
✔ CDC pipelines
✔ Columnstore analytics

---

# ✅ 50 Real Senior SQL Server Interview Q&A (Architect / BFSI-grade)

## A) Architecture & Storage (1–8)

**1) How does SQL Server store data physically?**
A: 8KB pages → 64KB extents. Clustered index defines physical order (B-Tree). Heaps can create forwarded records.

**2) Latches vs locks?**
A: Locks = logical consistency. Latches = protect in-memory structures (short-lived), e.g., `PAGELATCH_*`.

**3) What happens during checkpoint?**
A: Flush dirty pages → data files; advances recovery point; reduces crash recovery time.

**4) Why transaction log sequential?**
A: Append-only sequential writes → high throughput. `WRITELOG` waits indicate log storage bottleneck.

**5) Buffer pool and memory pressure symptoms?**
A: Cache pages; pressure = low PLE, more physical reads, high `PAGEIOLATCH`.

**6) Plan cache vs data cache?**
A: Plan cache = compiled plans. Data cache = pages. Parameter sniffing = plan cache issue.

**7) TempDB used for?**
A: Sorts, hashes, spools, version store, temp objects.

**8) TempDB bottlenecks and fixes?**
A: Allocation contention (`PAGELATCH_*`) → multiple tempdb files; reduce spills; modern SQL reduces contention.

---

## B) Indexing & Query Plans (9–22)

**9) Clustered vs nonclustered decision?**
A: Clustered key narrow + stable + increasing to reduce fragmentation; nonclustered for access paths.

**10) Key lookup and why bad?**
A: Seek + lookup per row → expensive for many rows. Fix with INCLUDE/covering index.

**11) SARGability?**
A: Index seek-friendly predicates. Avoid functions on columns; avoid implicit conversions.

**12) Filtered indexes use case?**
A: Skewed subsets queried often (queues: `Status='PENDING'`).

**13) Included columns benefit?**
A: Cover query at leaf level without bloating key size.

**14) Join operators?**
A: Nested loops (small outer + indexed inner), hash match (large sets), merge join (sorted inputs).

**15) Parameter sniffing?**
A: Plan compiled for initial parameter values reused for others. Fix: `RECOMPILE`, `OPTIMIZE FOR`, selective techniques.

**16) Statistics role?**
A: Cardinality estimation drives plan choice; stale stats → spills/scans/wrong joins.

**17) Why scan even with index?**
A: Non-SARGable, implicit conversion, low selectivity, stale stats, sniffing, lookup cost.

**18) How read execution plan quickly?**
A: Focus on expensive operators, actual vs estimated rows, spills, scans+lookups, warnings.

**19) Spill meaning and fix?**
A: Sort/hash spills to tempdb; fix with indexes, stats, query rewrite, memory grant improvements.

**20) Columnstore when?**
A: OLAP/reporting. Mixed workload: nonclustered columnstore for analytics.

**21) Fill factor when change?**
A: Random inserts/updates causing splits; lower fill factor leaves space but increases I/O.

**22) Rebuild vs reorganize?**
A: Rebuild = full recreate + stats update (more log). Reorganize = incremental defrag.

---

## C) Transactions, Locking, Isolation (23–34)

**23) ACID in SQL Server?**
A: Transactions/log (atomic+durable), constraints (consistent), locks/versioning (isolation).

**24) Blocking vs deadlock?**
A: Blocking is wait; deadlock is cycle—victim rolled back.

**25) Deadlock troubleshooting?**
A: Capture graph, consistent order, shorten transactions, add indexes, avoid unnecessary serializable.

**26) RCSI vs Snapshot?**
A: RCSI = statement consistent reads; Snapshot = transaction consistent reads; both use version store.

**27) Serializable use and cost?**
A: Prevent phantoms; cost = more locks + blocking.

**28) Optimistic vs pessimistic?**
A: Locks vs versioning/rowversion checks.

**29) NOLOCK?**
A: Avoid in finance—can read dirty/duplicate/missing rows. Prefer RCSI/Snapshot.

**30) Lock escalation?**
A: Many locks become table lock. Fix with better indexes, batching, shorter transactions.

**31) UPDLOCK usefulness?**
A: Prevent conversion deadlocks; useful for “claim row” workflows.

**32) DB-level idempotency?**
A: Unique constraint on idempotency key / business key; insert-first then apply.

**33) Lost update prevention?**
A: Isolation, rowversion checks, conditional updates.

**34) Savepoints?**
A: Partial rollback within a transaction.

---

## D) Procs, Functions, Triggers (35–42)

**35) Why stored procs in BFSI?**
A: Central logic, controlled permissions, fewer roundtrips, auditable deployments.

**36) EXEC vs sp_executesql?**
A: `sp_executesql` supports parameters → better plan reuse + safer.

**37) Scalar UDF vs inline TVF?**
A: Scalar UDF historically slow; SQL 2019+ may inline. Inline TVF optimizes like a view.

**38) THROW vs RAISERROR?**
A: Prefer THROW (modern, better stack). RAISERROR legacy.

**39) Deferred name resolution?**
A: Proc can be created even if referenced table missing—fails at runtime.

**40) Trigger usage?**
A: Use for audit stamping; avoid heavy business logic triggers in hot paths.

**41) inserted/deleted tables?**
A: Trigger pseudo-tables; write set-based (handle multi-row).

**42) DDL/logon triggers risks?**
A: Can block deployments/lockout admins—use only with strict governance.

---

## E) HA/DR, Security, Ops (43–50)

**43) AG vs FCI?**
A: AG supports readable secondaries; FCI is shared storage instance HA.

**44) Backup strategy (Full model)?**
A: Full + diff + frequent log backups; test restores.

**45) Point-in-time recovery?**
A: Restore full NORECOVERY → diff → logs to target time → RECOVERY.

**46) TDE vs Always Encrypted?**
A: TDE protects files; AE protects columns even from DBA (client-side keys).

**47) RLS BFSI use case?**
A: Tenant isolation or role-based record filtering.

**48) Query Store in regressions?**
A: Compare runtime stats, force last known good plan, monitor plan changes.

**49) First DMVs in incident?**
A: `dm_exec_requests`, `dm_exec_query_stats`, `dm_os_wait_stats`, `dm_tran_locks`, `dm_db_index_usage_stats`.

**50) Top production anti-patterns?**
A: NOLOCK everywhere, implicit conversions, long transactions without proper indexes.

---

# 🏦 BFSI Payment Ledger SQL Design Deep Dive (Interview-Ready)

## 1) Non-negotiables (Ledger Invariants)

* **Immutability**: never overwrite financial facts; append reversals/adjustments
* **Double-entry**: Σ(debits) = Σ(credits) per transaction
* **Idempotency**: safe retries, no duplicate postings
* **Auditability**: who/when/source/correlation
* **Reconciliation**: internal vs external settlement
* **Correct under concurrency**: no lost updates, controlled locking

---

## 2) Canonical Double-Entry Model (Core Tables)

* `ledger_transaction` (header): one logical posting event
* `ledger_entry` (lines): debit/credit legs
* `account`: customer + GL accounts (fees/suspense/settlement)
* `idempotency_key`: dedupe requests
* `outbox_event`: publish reliably after commit (Kafka/Rabbit)
* `recon_*`: settlement reconciliation

---

## 3) Recommended Data Types (Payments-safe)

* Internal PK: `BIGINT IDENTITY`
* Public correlation: `UNIQUEIDENTIFIER` (TxnGuid)
* Money: `DECIMAL(19,4)` (or currency-specific scale; never float)
* Time: `DATETIME2(3)` or `DATETIME2(7)` for ordering + audits

---

## 4) Strong Schema Constraints (What makes it “banking-grade”)

### `account`

* `AccountId (PK)`
* `AccountType` (CUSTOMER, GL, SUSPENSE, FEES, SETTLEMENT)
* `CurrencyCode`, `Status`
* Index: `(AccountType, CurrencyCode, Status)`

### `ledger_transaction`

* `TxnId (PK identity)`
* `TxnGuid (unique)`
* `SourceSystem`, `ExternalRef` (unique per source)
* `TxnType` (AUTH/CAPTURE/REFUND/REVERSAL/CHARGEBACK/FEE/ADJUSTMENT)
* `Status` (POSTED/REVERSED)
* `OccurredAt` (business), `CreatedAt` (system)
* **Unique constraint**: `(SourceSystem, ExternalRef)` to enforce idempotency

### `ledger_entry`

* `EntryId (PK identity)`
* `TxnId (FK)`, `AccountId (FK)`
* `Direction` (D/C or +1/-1)
* `Amount DECIMAL(...)` (positive)
* `CurrencyCode`, `EntryType` (PRINCIPAL/FEE/TAX/ROUNDING/SUSPENSE)
* Index:

  * `(AccountId, CreatedAt)` for statements
  * `(TxnId)` for posting integrity
  * INCLUDE: `Amount, Direction, CurrencyCode`

**Hard rule:** no mixed currency per txn unless explicit FX legs.

---

## 5) Enforce “Debits = Credits”

**Option A (preferred): enforce inside posting stored procedure**

* Insert header
* Insert entries
* Validate sum-to-zero
* Commit

Validation:

* `SUM(CASE WHEN Direction='D' THEN Amount ELSE -Amount END) = 0`

**Option B:** maintain `ledger_txn_balance` derived table updated in same proc
**Avoid triggers** for core posting correctness (latency, recursion, complexity).

---

## 6) Idempotency (Mandatory in payments)

**Pattern**

1. Client sends `Idempotency-Key`
2. DB enforces uniqueness
3. If exists → return previous result
4. Else → post

Table:

* `idempotency_key(KeyHash UNIQUE, RequestBodyHash, Status, CreatedAt, ResponseSnapshot)`

Proc:

* Insert idempotency row first
* On duplicate key → fetch snapshot and exit

---

## 7) Balances Strategy

### Option A: compute from entries (pure ledger)

* Always correct
* Expensive for real-time balance

### Option B: derived balance table (recommended for UX + scale)

`account_balance(AccountId, Available, Ledger, UpdatedAt, RowVersion)`

* Updated only by posting proc
* Ledger entries remain source of truth
* Rebuildable via batch reconcile job

**BFSI note:** `Available` vs `Ledger` is crucial due to holds/authorizations.

---

## 8) Holds / Authorizations (Card Model)

Table:

* `auth_hold(HoldId, AccountId, Amount, ExpiresAt, Status, ExternalAuthRef UNIQUE)`

Flow:

* AUTH → create hold
* CAPTURE → convert hold to posted txn (partial allowed)
* REVERSAL/EXPIRY → release hold

Concurrency:

* lock hold row with `UPDLOCK` during capture to prevent double capture.

---

## 9) Concurrency & Isolation Choices (Practical)

Posting procedure:

* Use `READ COMMITTED`
* Strong uniqueness constraints
* Serialize balance updates per account:

  * `SELECT ... WITH (UPDLOCK, ROWLOCK) WHERE AccountId=@id`

Read paths:

* Enable **RCSI** for statements to avoid blocking writes

Avoid `SERIALIZABLE` unless enforcing range invariants.

---

## 10) Partitioning for 10M tx/day

Partition `ledger_entry` by `CreatedAt` (weekly/monthly):

* Partition elimination for statements
* Fast archive with partition switching
* Aligned indexes

Hot partitions (latest) need:

* narrow clustered keys
* tuned fill factor
* minimal nonclustered indexes on write path

---

## 11) Audit + Tamper Evidence

Minimum audit columns:

* `CreatedBy`, `CreatedAt`, `SourceSystem`, `RequestId`, `CorrelationId`

Tamper evidence (hash chain idea):

* `TxnHash = HASH(TxnData + PrevTxnHash)`
* store `PrevTxnHash` to detect tampering

(SQL Server 2022 ledger tables can be mentioned as optional enhancement.)

---

## 12) Outbox Pattern (Reliable Kafka Publish)

`outbox_event(EventId, TxnId, EventType, Payload, CreatedAt, PublishedAt NULL)`

* insert in same DB txn
* background publisher publishes and marks published

Guarantees “effectively once” delivery with idempotent consumers.

---

## 13) Reconciliation (Internal vs External)

Tables:

* `external_settlement(SettlementRef, Amount, Currency, OccurredAt, SourceFileId)`
* `recon_match(InternalTxnId, SettlementRef, MatchStatus, ReasonCode)`

Jobs:

* match by external ref
* handle partials/fees
* flag breaks for ops

**Interview line:**

> Ledger correctness is internal; reconciliation ensures alignment with settlement rails and bank statements.

---

## 14) Posting Stored Procedure (Perfect Interview Outline)

1. Validate idempotency
2. Validate account + currency
3. Insert txn header
4. Insert ledger entries (set-based)
5. Validate sum-to-zero + invariants
6. Update derived balances with row locks
7. Insert outbox event
8. Commit

Emphasize:

* set-based operations
* constraints as safety net
* minimal lock time
* safe retries

---

```
```
