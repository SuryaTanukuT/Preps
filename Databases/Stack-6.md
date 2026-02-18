
# 1) One-page “What to say in interviews” cheat sheet (bank-grade)

### The 60-second anchor pitch

> “For BFSI, the **ledger is the source of truth** and must be **ACID + deterministic**. I model money movement as **double-entry**: a journal header + postings lines, **append-only**, with **idempotency** enforced by **unique constraints**. I keep transactions short, lock the right row(s) (`FOR UPDATE`), and validate funds inside the same TX. I avoid distributed transactions with **Outbox + event bus**. I scale reads with **replicas/CQRS/materialized views**, scale history with **time partitioning + archiving**, and ensure compliance via **immutable audit**, encryption, strict RBAC/RLS, and tested DR (RPO/RTO).”

---

## A) Correctness (ledger invariants)

* **Money truth = postings (append-only)**, not a mutable balance column.
* **Invariant:** for each ledger transaction, **SUM(debits) == SUM(credits)**.
* **No updates/deletes** for financial rows: use **reversal postings**.
* **Idempotency:** unique `(tenant_id, idempotency_key)` → retries return same result.

## B) Concurrency & isolation (what you say confidently)

* “For same-account concurrency, I lock the account row (or balance row) and check funds **inside one TX**.”
* “For strict safety, use **SERIALIZABLE** or **REPEATABLE READ + explicit locks** + invariants.”
* “Deadlocks: consistent lock ordering + shorter TX + retry with jitter.”

## C) Scale (how you scale without breaking money)

* Writes: keep TX short, avoid heavy FKs on hottest tables, partition history.
* Reads: replicas + statement read model (CQRS), **as-of timestamp** to handle replica lag.
* Hot accounts: hash-bucket routing / throttles / dedicated shard for whales.

## D) Eventing reliability (bank-grade)

* **Outbox:** write ledger + outbox row in same DB TX → publisher emits to broker.
* Consumers: **idempotent** via unique `(consumer, event_id)`.

## E) Compliance & security bullets

* TLS/mTLS, encryption at rest, **tokenization/field encryption for PII**.
* Least privilege; separation of duties (maker-checker); immutable audit log.
* Retention 7–10 years: partitions + archive (cold storage), preserve retrieval.

---

# 2) 30 “bank-grade” scenarios (question → approach → pitfalls)

### Ledger / posting correctness (1–10)

1. **Two debits hit same account concurrently**
   Approach: lock balance row/account row → validate funds → post entries → commit
   Pitfall: check funds outside TX → race condition / double-spend

2. **Retry storm causes duplicate postings**
   Approach: idempotency unique key + store final outcome + return cached result
   Pitfall: idempotency in cache only (Redis) without DB uniqueness

3. **Fee + FX + principal in one payment**
   Approach: model as multiple postings legs under one txn_id
   Pitfall: stuffing fee into amount and losing auditability

4. **Reversal / chargeback**
   Approach: new ledger_txn reversal_of original + opposite postings
   Pitfall: update/delete original postings

5. **Authorization hold then capture**
   Approach: holds table with TTL + available vs ledger balances
   Pitfall: hold not linked to ledger state → phantom funds

6. **Exactly-once posting with microservices**
   Approach: ledger TX + outbox; consumer dedupe by event_id
   Pitfall: “publish then commit” ordering

7. **Partial failure after debit but before credit**
   Approach: single DB TX (double-entry) prevents partial commit
   Pitfall: multi-step commits without transaction boundary

8. **Backdated posting / late settlement**
   Approach: posted_at separate from created_at; partitions by time; reconcile
   Pitfall: assuming insert time == value time

9. **Statement query too slow**
   Approach: partition pruning + index (account_id, posted_at desc) + monthly MV/read model
   Pitfall: unbounded scans / missing composite index

10. **Maker-checker for high-value payment**
    Approach: payment_request pending → approvals table → on approve create ledger postings
    Pitfall: allowing maker to approve own request; missing audit trail

### Performance & troubleshooting (11–20)

11. **High CPU on DB**
    Approach: top queries (pg_stat_statements) → fix plan/index → reduce JSON ops
    Pitfall: adding indexes blindly (write amplification)

12. **Replication lag**
    Approach: route consistency reads to primary; add “as-of” timestamp; fix long TX
    Pitfall: serving “fresh balance” from replica

13. **Deadlocks in posting flow**
    Approach: consistent lock ordering (accounts sorted) + shorter TX + retries
    Pitfall: locking tables in different order per code path

14. **Hot tenant/account**
    Approach: per-entity throttles + hash bucket + queueing + isolate whales
    Pitfall: global counters / hotspot partition key

15. **Bloat & vacuum pressure**
    Approach: reduce churn tables, partition, tune vacuum, avoid updates on append-only
    Pitfall: frequent UPDATEs on “ledger rows”

16. **Index bloat on write-heavy tables**
    Approach: minimal essential indexes; partial indexes for pending rows; monitor
    Pitfall: too many secondary indexes on postings

17. **Connection exhaustion**
    Approach: pgbouncer/DB proxy + right-size pools + separate read/write pools
    Pitfall: app pool > DB capacity

18. **Slow reconciliation batch**
    Approach: incremental reconciliation; partition; write breaks to cases table
    Pitfall: full-table reconciliation daily

19. **Disk IO saturation**
    Approach: partition hot/cold, reduce working set, tune checkpoints, batch writes
    Pitfall: huge transactions + giant WAL spikes

20. **Plan regression / parameter sniffing issues**
    Approach: stable query shapes; explicit casts; plan hints (engine-specific)
    Pitfall: dynamic SQL with unpredictable predicates

### DR / multi-region / compliance (21–30)

21. **RPO≈0 requirement**
    Approach: synchronous replication (distance limits) + fencing + tested failover
    Pitfall: assuming async replication gives RPO 0

22. **Active-active demanded for ledger**
    Approach: single-writer per account/tenant routing + global txn ids + recon
    Pitfall: conflict resolution for money is extremely risky

23. **GDPR deletion vs financial retention**
    Approach: separate PII store/tokenization; delete token mapping; keep ledger immutable
    Pitfall: deleting ledger rows (regulatory disaster)

24. **PCI scope reduction**
    Approach: tokenize PAN; store tokens only; strict key mgmt; audit access
    Pitfall: storing raw card data in OLTP tables

25. **Audit tamper-evidence**
    Approach: append-only audit + hash chain per partition/day; restrict permissions
    Pitfall: audit log writable by app role

26. **Privileged access review**
    Approach: RBAC + RLS; log privileged queries/actions; break-glass workflow
    Pitfall: shared DBA accounts without audit

27. **PII leaks in logs**
    Approach: structured logging + masking + no raw payload logging
    Pitfall: logging request bodies in production

28. **Settlement files arrive late/out of order**
    Approach: store external refs; idempotent file ingestion; reconcile by date windows
    Pitfall: overwriting “settled” state without history

29. **Search index drift (Elastic) vs DB**
    Approach: CDC/outbox replays + reindex pipeline; DB remains truth
    Pitfall: treating search as source of truth

30. **Property search geo queries at scale**
    Approach: PostGIS for canonical geo + Elastic for faceting/search; cache hot tiles
    Pitfall: trying to do rich geo + faceting only in OLTP

---

# 3) PostgreSQL ledger DDL + indexes + partitioning (copy/paste ready)

This is a **practical BFSI-grade schema**: idempotency, holds, double-entry postings, outbox, audit, maker-checker.
Assumes multi-tenant (`tenant_id`) and money stored in **minor units** (`BIGINT`).

> Notes:
>
> * We enforce “debit = credit” via a **DEFERRABLE constraint trigger** (validated at COMMIT).
> * `ledger_postings` is **partitioned by posted_at** (monthly).
> * App role should not have UPDATE/DELETE on financial tables (grant section included).

```sql
-- Enable UUID generation (PG15+ often has pgcrypto)
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- 0) Core enums
DO $$ BEGIN
  CREATE TYPE entry_type AS ENUM ('DEBIT', 'CREDIT');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE ledger_status AS ENUM ('PENDING', 'POSTED', 'REVERSED', 'FAILED');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  CREATE TYPE payment_status AS ENUM ('RECEIVED', 'PENDING_APPROVAL', 'APPROVED', 'REJECTED', 'POSTED', 'FAILED');
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

-- 1) Customers (PII should be minimized/tokenized in real BFSI)
CREATE TABLE IF NOT EXISTS customers (
  customer_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   uuid NOT NULL,
  external_ref text,
  created_at  timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_customers_tenant ON customers(tenant_id);

-- 2) Accounts
CREATE TABLE IF NOT EXISTS accounts (
  account_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id  uuid NOT NULL,
  customer_id uuid NOT NULL REFERENCES customers(customer_id),
  account_type text NOT NULL,              -- SAVINGS/CURRENT/WALLET/etc
  currency char(3) NOT NULL,
  status text NOT NULL DEFAULT 'ACTIVE',
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_accounts_tenant ON accounts(tenant_id);
CREATE INDEX IF NOT EXISTS idx_accounts_customer ON accounts(customer_id);

-- 3) Payment requests (API intent + idempotency boundary)
CREATE TABLE IF NOT EXISTS payment_requests (
  payment_request_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  idempotency_key text NOT NULL,
  request_id text,                         -- correlation id
  status payment_status NOT NULL DEFAULT 'RECEIVED',
  amount_minor bigint NOT NULL CHECK (amount_minor > 0),
  currency char(3) NOT NULL,
  debtor_account_id uuid NOT NULL REFERENCES accounts(account_id),
  creditor_account_id uuid REFERENCES accounts(account_id), -- optional (merchant/internal)
  external_ref text,                       -- network/bank reference
  result_txn_id uuid,                      -- ledger_transactions.txn_id once posted
  created_by text,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now(),

  UNIQUE (tenant_id, idempotency_key)
);

CREATE INDEX IF NOT EXISTS idx_payment_requests_status
  ON payment_requests(tenant_id, status, created_at DESC);

-- 4) Holds / reservations (auth-hold model)
CREATE TABLE IF NOT EXISTS holds (
  hold_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  account_id uuid NOT NULL REFERENCES accounts(account_id),
  payment_request_id uuid REFERENCES payment_requests(payment_request_id),
  amount_minor bigint NOT NULL CHECK (amount_minor > 0),
  currency char(3) NOT NULL,
  status text NOT NULL DEFAULT 'ACTIVE',   -- ACTIVE/RELEASED/CAPTURED/EXPIRED
  expires_at timestamptz,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_holds_active
  ON holds(tenant_id, account_id, status)
  WHERE status = 'ACTIVE';

-- 5) Ledger transaction header (journal)
CREATE TABLE IF NOT EXISTS ledger_transactions (
  txn_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  txn_type text NOT NULL,                  -- TRANSFER/CARD_AUTH/FEE/FX/REFUND/SETTLEMENT
  status ledger_status NOT NULL DEFAULT 'POSTED',
  currency char(3) NOT NULL,
  payment_request_id uuid REFERENCES payment_requests(payment_request_id),
  request_id text,
  idempotency_key text,
  external_ref text,
  description text,
  reversal_of_txn_id uuid REFERENCES ledger_transactions(txn_id),
  created_by text,
  approved_by text,
  created_at timestamptz NOT NULL DEFAULT now(),
  posted_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_ledger_txn_tenant_posted
  ON ledger_transactions(tenant_id, posted_at DESC);

CREATE INDEX IF NOT EXISTS idx_ledger_txn_payment_request
  ON ledger_transactions(payment_request_id);

-- 6) Ledger postings (journal lines) - partitioned by time
CREATE TABLE IF NOT EXISTS ledger_postings (
  posting_id uuid NOT NULL DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  txn_id uuid NOT NULL REFERENCES ledger_transactions(txn_id),
  account_id uuid NOT NULL REFERENCES accounts(account_id),
  entry entry_type NOT NULL,
  amount_minor bigint NOT NULL CHECK (amount_minor > 0),
  currency char(3) NOT NULL,
  posted_at timestamptz NOT NULL DEFAULT now(),
  narration text,

  PRIMARY KEY (posting_id, posted_at)
) PARTITION BY RANGE (posted_at);

-- Example monthly partitions (create via automation each month)
CREATE TABLE IF NOT EXISTS ledger_postings_2026_02
  PARTITION OF ledger_postings
  FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

CREATE TABLE IF NOT EXISTS ledger_postings_2026_03
  PARTITION OF ledger_postings
  FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');

-- Indexes on parent (propagate to partitions in PG11+ when created on parent)
CREATE INDEX IF NOT EXISTS idx_postings_account_time
  ON ledger_postings(tenant_id, account_id, posted_at DESC);

CREATE INDEX IF NOT EXISTS idx_postings_txn
  ON ledger_postings(tenant_id, txn_id);

-- 7) Derived balances (cacheable/rebuildable)
CREATE TABLE IF NOT EXISTS account_balances (
  tenant_id uuid NOT NULL,
  account_id uuid PRIMARY KEY REFERENCES accounts(account_id),
  ledger_minor bigint NOT NULL DEFAULT 0,      -- total posted
  available_minor bigint NOT NULL DEFAULT 0,   -- ledger - active holds
  updated_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_account_balances_tenant
  ON account_balances(tenant_id);

-- 8) Maker-checker approvals
CREATE TABLE IF NOT EXISTS approvals (
  approval_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  payment_request_id uuid NOT NULL REFERENCES payment_requests(payment_request_id),
  maker text NOT NULL,
  checker text,
  status text NOT NULL DEFAULT 'PENDING',   -- PENDING/APPROVED/REJECTED
  reason text,
  created_at timestamptz NOT NULL DEFAULT now(),
  decided_at timestamptz
);

CREATE INDEX IF NOT EXISTS idx_approvals_pending
  ON approvals(tenant_id, status, created_at DESC)
  WHERE status = 'PENDING';

-- 9) Outbox (reliable event publish)
CREATE TABLE IF NOT EXISTS outbox_events (
  outbox_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  aggregate_type text NOT NULL,     -- PAYMENT/LEDGER/ACCOUNT
  aggregate_id uuid NOT NULL,
  event_type text NOT NULL,         -- payment.posted, ledger.reversed, etc
  event_id uuid NOT NULL DEFAULT gen_random_uuid(),
  payload jsonb NOT NULL,
  occurred_at timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz,
  publish_attempts int NOT NULL DEFAULT 0,

  UNIQUE (tenant_id, event_id)
);

CREATE INDEX IF NOT EXISTS idx_outbox_unpublished
  ON outbox_events(tenant_id, published_at, occurred_at)
  WHERE published_at IS NULL;

-- 10) Audit log (append-only)
CREATE TABLE IF NOT EXISTS audit_log (
  audit_id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL,
  actor text,
  action text NOT NULL,             -- PAYMENT_CREATE / PAYMENT_APPROVE / LEDGER_POST / etc
  entity_type text,
  entity_id uuid,
  request_id text,
  details jsonb,
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_audit_tenant_time
  ON audit_log(tenant_id, created_at DESC);

-- 11) Enforce double-entry invariant per txn via deferred constraint trigger
-- This checks that total debits == total credits for a txn_id at commit time.
CREATE OR REPLACE FUNCTION assert_double_entry() RETURNS trigger AS $$
DECLARE
  d bigint;
  c bigint;
BEGIN
  SELECT COALESCE(SUM(amount_minor),0) INTO d
  FROM ledger_postings
  WHERE tenant_id = NEW.tenant_id AND txn_id = NEW.txn_id AND entry = 'DEBIT';

  SELECT COALESCE(SUM(amount_minor),0) INTO c
  FROM ledger_postings
  WHERE tenant_id = NEW.tenant_id AND txn_id = NEW.txn_id AND entry = 'CREDIT';

  IF d <> c THEN
    RAISE EXCEPTION 'Double-entry violated for txn_id %, debit %, credit %', NEW.txn_id, d, c
      USING ERRCODE = '23514'; -- check_violation
  END IF;

  RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Constraint trigger fires at end of transaction (DEFERRABLE INITIALLY DEFERRED)
DROP TRIGGER IF EXISTS trg_assert_double_entry ON ledger_postings;
CREATE CONSTRAINT TRIGGER trg_assert_double_entry
AFTER INSERT OR UPDATE OR DELETE ON ledger_postings
DEFERRABLE INITIALLY DEFERRED
FOR EACH ROW EXECUTE FUNCTION assert_double_entry();

-- 12) Minimal “append-only” permission posture (illustrative)
-- In real setups: app role only INSERT/SELECT on ledger tables, no UPDATE/DELETE.
-- (Run with a privileged role)
-- REVOKE UPDATE, DELETE ON ledger_transactions, ledger_postings, audit_log FROM app_role;
-- GRANT SELECT, INSERT ON ledger_transactions, ledger_postings, audit_log TO app_role;
```

## Partitioning strategy (what to say)

* Partition `ledger_postings` by **posted_at monthly** (or weekly at very high throughput).
* Keep last N months “hot” on fast storage; detach/archive older partitions.
* Statement queries get partition pruning + `(account_id, posted_at DESC)` index.

## Optional (but interview-strong): RLS pattern (multi-tenant safety)

If you use shared DB + shared schema, add **Row Level Security**:

* Require `tenant_id` in every table
* `SET app.tenant_id = '...'` per request
* RLS policy filters rows by tenant automatically (prevents leakage even if code misses a filter)

---


