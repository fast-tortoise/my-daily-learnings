# Consistency Patterns in Microservices — Notes

## 1. Core Problem

In a monolith, multiple business changes can be handled inside one database transaction.

Example:

```sql
BEGIN;
UPDATE orders SET status = 'PAID';
UPDATE payments SET status = 'SUCCESS';
UPDATE inventory SET reserved_qty = reserved_qty + 1;
COMMIT;
```

In microservices, each service usually owns its own database:

```text
Order Service      -> orders_db
Payment Service    -> payments_db
Inventory Service  -> inventory_db
```

So one ACID transaction cannot safely cover all services.

The main problem becomes:

```text
How do we keep business state correct when different services update their own DBs independently?
```

---

## 2. Example Flow Discussed

Services:

```text
Order Service
Payment Service
Inventory Service
```

Flow:

```text
1. Order Service creates order.
2. Order Service calls Payment Service synchronously.
3. Payment succeeds.
4. PaymentSucceeded event is published to Kafka.
5. Inventory Service consumes event asynchronously.
6. Inventory Service reserves stock.
```

Failure case:

```text
Payment succeeds,
but Kafka publish fails.
```

Problem:

```text
Payment = SUCCESS
Inventory = NOT RESERVED
Order may remain stuck
```

---

## 3. Outbox Pattern

### Problem Solved

Outbox solves this failure:

```text
DB update succeeds,
but Kafka publish fails.
```

### Correct Design

When payment succeeds, Payment Service should update payment status and insert an event into an outbox table in the same PostgreSQL transaction.

```sql
BEGIN;

UPDATE payments
SET status = 'SUCCESS'
WHERE payment_id = ?;

INSERT INTO outbox_events(
    event_id,
    event_type,
    aggregate_id,
    payload,
    status,
    created_at
)
VALUES (
    ?,
    'PaymentSucceeded',
    ?,
    ?,
    'READY',
    now()
);

COMMIT;
```

A background publisher later reads the outbox table and publishes events to Kafka.

```text
Payment DB transaction:
payment success + event stored together
```

Even if Kafka is down, the event is not lost.

### Important Correction

Outbox does not retry the payment.

It only retries publishing the already-created event.

Bad mental model:

```text
Payment succeeded
Retry payment again
```

Good mental model:

```text
Payment succeeded
Outbox event exists
Retry publishing PaymentSucceeded event
```

---

## 4. Inventory Reservation and Stock Contention

Outbox solves event loss.

It does not directly solve stock contention.

Example problem:

```text
Only 1 item is left.
100 users paid at nearly the same time.
Only one reservation should succeed.
```

Inventory correctness should be handled inside Inventory Service using atomic DB operations.

Example:

```sql
UPDATE inventory
SET available_qty = available_qty - 1
WHERE sku_id = 'sku-iphone'
  AND available_qty >= 1;
```

Then check affected rows:

```text
affected rows = 1 -> reservation successful
affected rows = 0 -> out of stock
```

This prevents overselling under concurrency.

---

## 5. Saga Pattern

Saga handles cross-service business failure.

Example:

```text
Order created
Payment successful
Inventory reservation failed
```

Since we cannot rollback payment with a DB rollback, we use compensation.

```text
Inventory failed
-> publish InventoryReservationFailed
-> trigger refund
-> cancel order
```

Compensating actions:

```text
Payment success rollback = refund
Inventory reserved rollback = release stock
Order confirmed rollback = cancel order
```

Saga does not mean one distributed transaction.

It means multiple local transactions plus compensating actions.

---

## 6. FIFO / Inventory Booking Service Idea

You suggested using an inventory booking service or job table to process reservations in order.

This is useful when fairness matters.

Example:

```text
Only 1 item left.
Multiple paid orders compete for it.
Business wants first-come-first-serve allocation.
```

Possible design:

```text
1. Store reservation requests in a job/reservation table.
2. Order them by event time/payment success time/order created time.
3. Process in FIFO order.
4. Mark each reservation as RESERVED or FAILED.
```

### Pros

```text
Fairer allocation
More predictable ordering
Easier to reason about business priority
```

### Cons

```text
Higher latency
Failures are detected later
More operational complexity
Earlier stuck jobs can delay later jobs
Needs retry and reconciliation design
```

Important point:

```text
Strict FIFO is a business requirement, not always a default technical requirement.
```

Many systems prioritize:

```text
No overselling
Idempotent reservation
Retry safety
Clear compensation
```

over strict global FIFO.

---

## 7. Duplicate Kafka Event Problem

Problem:

```text
Inventory Service consumes PaymentSucceeded event.
It reserves stock successfully.
Consumer crashes before Kafka offset commit.
Kafka delivers the same event again.
```

If not handled, stock may be reserved twice.

---

## 8. Inbox Pattern

### Problem Solved

Inbox prevents duplicate event processing.

The consumer stores the consumed event ID in an inbox table.

```sql
CREATE TABLE inventory_inbox (
    event_id TEXT NOT NULL,
    consumer_name TEXT NOT NULL,
    processed_at TIMESTAMP DEFAULT now(),
    PRIMARY KEY (event_id, consumer_name)
);
```

Processing flow:

```sql
BEGIN;

INSERT INTO inventory_inbox(event_id, consumer_name)
VALUES ('evt-123', 'inventory-service');

UPDATE inventory
SET available_qty = available_qty - 1
WHERE sku_id = 'sku-iphone'
  AND available_qty >= 1;

INSERT INTO inventory_reservations(order_id, sku_id, quantity, status)
VALUES ('ord-1', 'sku-iphone', 1, 'RESERVED');

COMMIT;
```

If the same event comes again:

```sql
INSERT INTO inventory_inbox(event_id, consumer_name)
VALUES ('evt-123', 'inventory-service');
```

The unique constraint fails, so the service skips processing.

---

## 9. Important Atomicity Rule

The inbox insert, inventory decrement, and reservation insert must happen in the same DB transaction.

Bad:

```text
1. Insert inbox event.
2. Commit.
3. Update inventory.
4. Crash.
```

Problem:

```text
Event is marked processed,
but inventory was never reserved.
```

Good:

```text
BEGIN;
insert inbox event;
decrement inventory;
insert reservation;
COMMIT;
```

Either all succeed, or all rollback.

---

## 10. Business-Level Idempotency

Inbox prevents duplicate event processing by `event_id`.

But sometimes two different events may represent the same business action.

Example:

```text
evt-123 -> reserve inventory for order ord-1
evt-456 -> reserve inventory for order ord-1
```

Event IDs are different, but the business intent is same.

So we also need business-level uniqueness.

```sql
CREATE UNIQUE INDEX uq_inventory_reservation_order_sku
ON inventory_reservations(order_id, sku_id);
```

This prevents duplicate reservation for the same order and SKU.

Production systems usually use both:

```text
Inbox unique(event_id, consumer_name)
-> prevents duplicate event processing

Reservation unique(order_id, sku_id)
-> prevents duplicate business effect
```

---

## 11. Kafka Offset Crash Case

Scenario:

```text
1. Inventory Service consumes event evt-123.
2. DB transaction succeeds.
3. Inventory is reserved.
4. Consumer crashes before Kafka offset commit.
5. Kafka redelivers evt-123 after restart.
```

This is safe if inbox/idempotency is correct.

Why?

```text
The second processing attempt tries to insert event_id = evt-123 again.
The unique constraint rejects it.
The consumer skips the event.
Inventory is not reserved twice.
```

This is why Kafka consumers should be designed for at-least-once delivery.

Assume duplicate delivery can happen.

Protect correctness using idempotency.

---

## 12. Key Mental Model

Separate responsibilities clearly:

```text
Outbox pattern
-> prevents lost producer events

Inbox pattern
-> prevents duplicate consumer processing

DB constraints
-> protect business invariants

Atomic inventory update
-> prevents overselling

Saga pattern
-> handles cross-service failure and compensation

Reconciliation jobs
-> fix stuck/inconsistent states later
```

---

## 13. Production-Grade Order Flow

A better full design:

```text
1. Order Service creates order = PENDING_PAYMENT.

2. Payment Service charges user synchronously.

3. Payment Service DB transaction:
   - payment = SUCCESS
   - insert PaymentSucceeded event into outbox

4. Outbox publisher publishes PaymentSucceeded to Kafka.

5. Inventory Service consumes PaymentSucceeded.

6. Inventory Service DB transaction:
   - insert event_id into inbox
   - atomically decrement inventory
   - insert reservation row

7. If reservation succeeds:
   - publish InventoryReserved event using outbox

8. If reservation fails:
   - publish InventoryReservationFailed event using outbox

9. Saga handles:
   - confirm order if inventory reserved
   - refund payment and cancel order if inventory failed
```

---

## 14. Interview-Ready Answer

In microservices, I would not rely on one distributed transaction across Order, Payment, and Inventory services. I would keep strong consistency inside each service using PostgreSQL transactions, constraints, and atomic updates. For cross-service consistency, I would use eventual consistency with saga, outbox, inbox, retries, idempotency, and reconciliation.

For example, when payment succeeds, Payment Service should update payment status and insert a PaymentSucceeded event into an outbox table in the same DB transaction. A background publisher reliably publishes the event to Kafka. Inventory Service consumes the event idempotently using an inbox table with a unique event ID. The inbox insert, inventory decrement, and reservation insert should happen in one transaction. I would also add a business unique constraint like unique(order_id, sku_id) to prevent duplicate reservations even if two different events represent the same intent. If inventory fails, the saga triggers compensation like refunding payment and cancelling the order.

---

## 15. Topics Covered So Far

```text
Consistency in microservices
Eventual consistency
Saga pattern
Outbox pattern
Inbox pattern
Kafka duplicate delivery
Kafka offset commit crash
Idempotent consumers
Inventory reservation
Atomic stock decrement
Business-level uniqueness
FIFO reservation idea
Compensation and refund flow


- Indexing and query planner 
- Transactions and isolation levels 
- Locks, deadlocks, SKIP LOCKED, NOWAIT, advisory locks 
- Idempotency and uniqueness 
- PostgreSQL job queue production design 
- Partitioning and archival 
- Connection pooling and DB overload 
- Read replicas and replication lag 
- Schema design and foreign keys 
- COPY/staging/UPSERT bulk ingestion 
- Multi-tenant database design 
- Outbox/inbox pattern 
- Consistency patterns in microservices
```

---

## 16. Good Next Topics

Continue with one of these:

```text
1. Transaction isolation problems
2. Lost update in PostgreSQL
3. SELECT FOR UPDATE
4. SKIP LOCKED job queue
5. NOWAIT locks
6. Deadlocks
7. Idempotency key design
8. Outbox table production design
9. Inventory reservation table design
10. Reconciliation job design
```

Recommended next topic:

```text
Transaction isolation problems
```
