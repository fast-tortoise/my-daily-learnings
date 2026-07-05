# PostgreSQL + System Design Q/A Continuation Notes

## How to Continue This Chat

We are doing **situation-based PostgreSQL questions**.

Flow:

1. Assistant gives one practical scenario.
2. I answer with my design/query/thought process.
3. Assistant reviews my answer.
4. Assistant corrects gaps and explains production-level improvements.
5. Assistant gives the next question.

Focus areas:

* PostgreSQL
* System design
* Database design
* Backend engineering
* Production-scale tradeoffs

Preferred examples:

* Payments
* Inventory
* Kafka consumers
* Job queues
* Bulk ingestion
* Retries
* Idempotency
* Large tables

Start next with:

> Transaction isolation problems

Unless I ask for another topic.

---

# Topics Covered So Far

## 1. PostgreSQL Internals Senior Developers Should Know

Important internals:

* MVCC + VACUUM
* Query planner + indexes
* Transactions + locks
* Unique constraints / `ON CONFLICT`
* WAL + replication
* Connection pooling

---

## 2. Unique Constraints and `ON CONFLICT`

Key learnings:

* `PRIMARY KEY` automatically creates a unique index.
* `UNIQUE` constraint also automatically creates a unique index.
* `ON CONFLICT(column)` requires one of:

    * Primary key
    * Unique constraint
    * Unique index

Example:

```sql
CREATE UNIQUE INDEX uq_payment_order
ON payments(order_id);

INSERT INTO payments(order_id, amount)
VALUES (100, 500)
ON CONFLICT (order_id) DO NOTHING;
```

Important point:

* Unique indexes are especially useful for:

    * Partial uniqueness
    * Expression-based uniqueness

Example partial unique index:

```sql
CREATE UNIQUE INDEX uq_active_user_email
ON users(email)
WHERE deleted_at IS NULL;
```

---

## 3. `SKIP LOCKED`

### Purpose

`SKIP LOCKED` allows multiple workers to safely process different DB jobs without waiting on locked rows.

### Basic Pattern

```text
READY
  -> SELECT ... FOR UPDATE SKIP LOCKED
  -> mark PROCESSING
  -> COMMIT
  -> process outside transaction
  -> mark DONE / FAILED
```

### Important Production Rule

Do **not** process the actual job inside the DB transaction.

Bad:

```text
BEGIN
pick job
call external API / process job
mark DONE
COMMIT
```

Better:

```text
BEGIN
pick job
mark PROCESSING
COMMIT

process outside transaction

BEGIN
mark DONE / FAILED
COMMIT
```

Reason:

* Keeps transactions short
* Avoids long-held locks
* Prevents DB connection exhaustion
* Helps VACUUM work properly
* Reduces lock contention

### Crash Recovery

Need fields like:

```text
picked_at
worker_id
retry_count
max_retries
next_retry_at
```

A background recovery job should move stuck jobs back:

```text
PROCESSING -> READY / RETRYABLE
```

when:

```text
picked_at < now() - timeout
```

---

# 4. Per-Entity Ordering Problem

## Scenario

```text
job1(entity1)
job2(entity2)
job3(entity1)
```

Requirement:

```text
job1 must finish before job3
job2 can run in parallel
```

---

## Solution A: Dependency Check

Pick a job only if there is no older unfinished job for the same entity.

Conceptual query:

```sql
SELECT j.*
FROM jobs j
WHERE j.status = 'READY'
  AND NOT EXISTS (
      SELECT 1
      FROM jobs older
      WHERE older.entity_id = j.entity_id
        AND older.created_at < j.created_at
        AND older.status IN ('READY', 'PROCESSING', 'RETRYABLE')
  )
ORDER BY j.created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

Good:

* Simple logical model
* No separate lock table required

Concern:

* Can become expensive on a large jobs table
* Needs very good indexes

---

## Solution B: Entity Lock Table

Use a separate table:

```sql
CREATE TABLE entity_locks (
    entity_id BIGINT PRIMARY KEY,
    locked_at TIMESTAMP,
    worker_id TEXT
);
```

Flow:

```text
1. Worker locks one entity row using FOR UPDATE SKIP LOCKED.
2. Worker picks oldest READY job for that entity.
3. Worker marks job PROCESSING and commits.
4. Worker processes outside transaction.
5. Worker marks job DONE/FAILED.
6. Worker releases entity lock.
```

Benefit:

```text
One worker owns one entity at a time.
Different entities process in parallel.
Same entity remains sequential.
```

---

## Solution C: Kafka Alternative

Partition Kafka messages by `entity_id`.

```text
key = entity_id
```

Benefit:

* Ordering is naturally guaranteed within an entity.
* Different entities can process in parallel across partitions.

Limitation:

* One very hot entity can still bottleneck on one partition.

---

# 5. Payment Reconciliation Job System

## Scenario

External payment providers send events:

```text
payment_id
merchant_id
event_type
event_time
payload
```

Requirement:

```text
Same merchant jobs must process in event_time order.
Different merchants can process in parallel.
```

Scale:

```text
10 million jobs/day
1000 worker pods
PostgreSQL as main DB
```

Correct scale calculation:

```text
10,000,000 / 24 = ~416,666 jobs/hour
~6,944 jobs/minute
~116 jobs/second
```

---

## Suggested Table

```sql
CREATE TABLE payment_jobs (
    job_id BIGSERIAL PRIMARY KEY,

    merchant_id BIGINT NOT NULL,
    payment_id TEXT NOT NULL,
    event_type TEXT NOT NULL,
    event_time TIMESTAMP NOT NULL,
    payload JSONB NOT NULL,

    status TEXT NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    max_retries INT NOT NULL DEFAULT 5,
    next_retry_at TIMESTAMP,

    picked_at TIMESTAMP,
    worker_id TEXT,

    created_at TIMESTAMP NOT NULL DEFAULT now(),
    updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

Possible statuses:

```text
READY
PROCESSING
DONE
RETRYABLE
DEAD
```

---

## Important Indexes

For picking ready jobs:

```sql
CREATE INDEX idx_payment_jobs_ready_order
ON payment_jobs(merchant_id, event_time, job_id)
WHERE status = 'READY';
```

For stuck processing jobs:

```sql
CREATE INDEX idx_payment_jobs_processing_timeout
ON payment_jobs(picked_at)
WHERE status = 'PROCESSING';
```

For retryable jobs:

```sql
CREATE INDEX idx_payment_jobs_retryable
ON payment_jobs(next_retry_at, job_id)
WHERE status = 'RETRYABLE';
```

---

# 6. Merchant-Level Locking

## Problem

Same merchant jobs must be sequential.

Different merchants should run in parallel.

## Flow

### Step 1: Lock Merchant

```sql
BEGIN;

SELECT merchant_id
FROM merchant_locks
WHERE locked_at IS NULL
   OR locked_at < now() - interval '10 minutes'
FOR UPDATE SKIP LOCKED
LIMIT 1;

UPDATE merchant_locks
SET locked_at = now(),
    worker_id = :worker_id
WHERE merchant_id = :merchant_id;

COMMIT;
```

---

### Step 2: Pick Oldest Job for Merchant

```sql
BEGIN;

SELECT job_id
FROM payment_jobs
WHERE merchant_id = :merchant_id
  AND status = 'READY'
ORDER BY event_time, job_id
LIMIT 1
FOR UPDATE SKIP LOCKED;

UPDATE payment_jobs
SET status = 'PROCESSING',
    picked_at = now(),
    worker_id = :worker_id
WHERE job_id = :job_id;

COMMIT;
```

---

### Step 3: Process Outside Transaction

```text
Process job outside DB transaction.
```

---

### Step 4: Mark Result

Success:

```sql
UPDATE payment_jobs
SET status = 'DONE',
    updated_at = now()
WHERE job_id = :job_id;
```

Failure:

```sql
UPDATE payment_jobs
SET status = 'RETRYABLE',
    retry_count = retry_count + 1,
    next_retry_at = now() + interval '5 minutes',
    updated_at = now()
WHERE job_id = :job_id;
```

Release merchant lock:

```sql
UPDATE merchant_locks
SET locked_at = NULL,
    worker_id = NULL
WHERE merchant_id = :merchant_id;
```

---

# 7. Hot Merchant Bottleneck

## Problem

One merchant sends:

```text
5 million jobs/day
```

Other merchants send only thousands.

With merchant-level locking:

```text
One merchant = one worker lane
```

So the huge merchant becomes a bottleneck.

---

## First Improvement: Batch Processing

Instead of:

```text
lock merchant
pick 1 job
process 1 job
release merchant
```

Do:

```text
lock merchant
pick next 100 / 500 / 1000 jobs
process sequentially
release merchant
```

Example:

```sql
BEGIN;

SELECT job_id
FROM payment_jobs
WHERE merchant_id = :merchant_id
  AND status = 'READY'
ORDER BY event_time, job_id
LIMIT 500
FOR UPDATE SKIP LOCKED;

UPDATE payment_jobs
SET status = 'PROCESSING',
    picked_at = now(),
    worker_id = :worker_id
WHERE job_id IN (:job_ids);

COMMIT;
```

Then process batch sequentially outside transaction.

---

## Limitation of Batching

Batching improves DB efficiency but does not create true parallelism inside the same merchant.

If merchant-wide ordering is strict:

```text
job999 must finish before job1000
```

then one merchant remains a single sequential stream.

---

## Senior-Level Redesign

Ask if ordering can be reduced from:

```text
merchant_id
```

to a smaller business-safe key:

```text
merchant_id + payment_id
merchant_id + account_id
merchant_id + terminal_id
merchant_id + settlement_date
```

Example:

```text
merchant=10, payment=A -> sequential
merchant=10, payment=B -> sequential
```

This allows parallelism within the same merchant.

---

## Kafka Parallelism Note

If Kafka key is:

```text
merchant_id
```

then one huge merchant still maps to one partition lane.

To parallelize, use a smaller key if business rules allow:

```text
merchant_id + payment_id
```

---

# 8. Slow Worker Query and Index Design

## Given Index

```sql
CREATE INDEX idx_ready_jobs
ON payment_jobs(status, merchant_id, event_time);
```

## Given Query

```sql
SELECT job_id
FROM payment_jobs
WHERE status = 'READY'
ORDER BY event_time
LIMIT 100
FOR UPDATE SKIP LOCKED;
```

## Table Size

```text
500 million old DONE jobs
2 million READY jobs
1 million PROCESSING jobs
```

---

## Problem

The index order is:

```text
status -> merchant_id -> event_time
```

But the query needs:

```text
status -> global event_time order
```

Because `merchant_id` is between `status` and `event_time`, PostgreSQL cannot use the index efficiently for global `ORDER BY event_time`.

---

## Better Index

```sql
CREATE INDEX idx_ready_jobs_event_time
ON payment_jobs(event_time, job_id)
WHERE status = 'READY';
```

Why this is better:

* Only indexes READY rows
* Supports `ORDER BY event_time`
* Keeps index much smaller
* Helps worker query directly

---

## More Improvements

### 1. Use Partial Indexes

```sql
CREATE INDEX idx_ready_jobs_event_time
ON payment_jobs(event_time, job_id)
WHERE status = 'READY';
```

```sql
CREATE INDEX idx_processing_jobs_timeout
ON payment_jobs(picked_at)
WHERE status = 'PROCESSING';
```

---

### 2. Move DONE Jobs Out

Keep hot table small:

```text
payment_jobs_hot
payment_jobs_history
```

Move completed jobs:

```text
DONE / DEAD -> history table
```

---

### 3. Partition by Date

Use range partitioning:

```text
payment_jobs_2026_07_05
payment_jobs_2026_07_06
payment_jobs_2026_07_07
```

Benefits:

* Smaller indexes
* Faster cleanup
* Better partition pruning
* Easier archival
* Drop old partitions instead of deleting millions of rows

---

### 4. Use `UPDATE ... RETURNING`

Instead of separate select and update, claim jobs atomically:

```sql
WITH picked AS (
    SELECT job_id
    FROM payment_jobs
    WHERE status = 'READY'
    ORDER BY event_time, job_id
    LIMIT 100
    FOR UPDATE SKIP LOCKED
)
UPDATE payment_jobs j
SET status = 'PROCESSING',
    picked_at = now(),
    worker_id = :worker_id
FROM picked
WHERE j.job_id = picked.job_id
RETURNING j.*;
```

Benefits:

* Atomic job claiming
* Cleaner application logic
* Fewer round trips
* Less race-prone

---

### 5. Tune Autovacuum

Queue tables get many updates:

```text
READY -> PROCESSING -> DONE
```

This creates dead tuples.

Possible table-specific tuning:

```sql
ALTER TABLE payment_jobs SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.01
);
```

---

### 6. Avoid Massive Updates/Deletes

Bad:

```sql
DELETE FROM payment_jobs
WHERE status = 'DONE';
```

Better:

```text
Delete in chunks
Move to history in chunks
Or drop old partitions
```

Example chunk cleanup:

```sql
DELETE FROM payment_jobs
WHERE job_id IN (
    SELECT job_id
    FROM payment_jobs
    WHERE status = 'DONE'
    LIMIT 10000
);
```

---

# 9. Important PostgreSQL + DB Design Topics to Continue

## A. Indexing and Query Planner

Useful question:

> Why is my query slow even though I have an index?

Covers:

* B-tree index order
* Composite indexes
* Partial indexes
* Covering indexes
* `EXPLAIN ANALYZE`
* Index scan vs sequential scan
* Selectivity
* Cardinality
* Statistics

---

## B. Transactions and Isolation Levels

Useful for:

* Payments
* Inventory
* Wallets
* Booking systems
* Idempotency

Covers:

* `READ COMMITTED`
* `REPEATABLE READ`
* `SERIALIZABLE`
* Dirty read
* Non-repeatable read
* Phantom read
* Lost update
* Write skew

Recommended next topic.

---

## C. Locking Patterns

Useful for distributed systems using DB as a coordinator.

Covers:

* Row locks
* `FOR UPDATE`
* `SKIP LOCKED`
* `NOWAIT`
* Advisory locks
* Deadlocks
* Lock timeout
* Optimistic locking
* Pessimistic locking

---

## D. Idempotency and Uniqueness

Useful for:

* Payments
* Retries
* Kafka consumers
* Duplicate API requests

Covers:

* Unique constraints
* `ON CONFLICT`
* Idempotency keys
* Dedup tables
* Inbox pattern
* Exactly-once illusion

---

## E. PostgreSQL Job Queue Production Design

Covers:

* Status-based queues
* Retry design
* Dead-letter jobs
* Worker crashes
* Stuck `PROCESSING` rows
* Batch picking
* Hot table cleanup

---

## F. Partitioning and Archival

Useful for large tables.

Covers:

* Range partitioning
* Hash partitioning
* Partition pruning
* Dropping old partitions
* Hot vs cold data
* History tables

---

## G. Connection Pooling and DB Overload

Useful for Spring Boot production systems.

Covers:

* HikariCP
* PostgreSQL `max_connections`
* PgBouncer
* Transaction pooling
* Thread pool vs DB pool mismatch
* Connection exhaustion

---

## H. Read Replicas and Replication Lag

Useful for scaling reads.

Covers:

* Primary/replica setup
* Replication lag
* Read-after-write consistency
* Leader/follower architecture
* Failover
* Logical replication

---

## I. Schema Design and Foreign Keys

Useful for DB design rounds.

Covers:

* Normalization
* Denormalization
* Foreign keys
* One-to-one
* One-to-many
* Many-to-many
* Audit tables
* Soft delete vs hard delete

---

## J. Large Write Ingestion

Useful for bulk data systems.

Covers:

* Single inserts
* Batch inserts
* `COPY`
* Staging tables
* Atomic merge
* `UPSERT`
* `MERGE`
* Bulk delete

---

## K. Multi-Tenant Database Design

Useful for SaaS/product systems.

Covers:

* `tenant_id` in every table
* Schema per tenant
* Database per tenant
* Noisy neighbor problem
* Tenant-level partitioning
* Tenant-level rate limiting

---

## L. Outbox / Inbox Pattern

Useful for microservices consistency.

Covers:

* Transactional outbox
* Inbox deduplication
* CDC
* Kafka publishing reliability
* Eventual consistency
* Retry safety

---

# Recommended Next Topic

Start next with:

## Transaction Isolation Problems

Reason:

This connects strongly to:

* Payments
* Inventory
* Booking systems
* Wallets
* Idempotency
* Race conditions
* Senior backend interviews

Example next question:

> Two users try to book the last available inventory item at the same time. Both requests read `quantity = 1`. How would you design the PostgreSQL transaction so only one succeeds?
