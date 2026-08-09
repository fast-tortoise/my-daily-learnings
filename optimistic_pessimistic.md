# Optimistic vs Pessimistic Locking

## 1. What problem do they solve?

Both are used when **multiple transactions try to update the same database row concurrently**.

Example:

```text
Stock = 10

Request A reads stock = 10
Request B reads stock = 10

Both try to update it.
```

Without proper concurrency control, one update may overwrite the other.

---

# 2. Optimistic Locking

Optimistic locking assumes:

> Conflicts are rare. Allow concurrent access, but detect if someone modified the row before updating it.

In JPA, it is commonly implemented using `@Version`.

```java
@Entity
public class Product {

    @Id
    private Long id;

    private int stock;

    @Version
    private Long version;
}
```

Suppose:

```text
stock = 10
version = 5
```

Two requests read the same row:

```text
Request A -> stock = 10, version = 5
Request B -> stock = 10, version = 5
```

Request A updates first.

Hibernate effectively executes:

```sql
UPDATE product
SET stock = 9,
    version = 6
WHERE id = 1
  AND version = 5;
```

Now Request B tries to update using version `5`.

```sql
UPDATE product
SET stock = 8,
    version = 6
WHERE id = 1
  AND version = 5;
```

But the row is already at version `6`.

Therefore:

```text
0 rows updated
```

Hibernate detects this and throws an optimistic locking exception.

Typical exceptions:

```java
OptimisticLockException
ObjectOptimisticLockingFailureException
```

The application can then:

```text
retry
OR
return conflict/error
```

### Optimistic locking flow

```text
A reads version = 5
B reads version = 5

A updates
version -> 6

B tries update with version = 5
        ↓
fails because version changed
```

---

# 3. Pessimistic Locking

Pessimistic locking assumes:

> Conflicts are likely. Lock the row before modifying it.

Typical SQL:

```sql
SELECT *
FROM product
WHERE id = 1
FOR UPDATE;
```

Spring Data JPA:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select p from Product p where p.id = :id")
Optional<Product> findByIdForUpdate(Long id);
```

Usage:

```java
@Transactional
public void reduceStock(Long productId) {

    Product product =
        repository.findByIdForUpdate(productId)
                  .orElseThrow();

    product.setStock(product.getStock() - 1);
}
```

Flow:

```text
Request A -> locks row

Request B -> tries same row
          -> waits

A updates
A commits
lock released

B gets lock
B reads latest value
B continues
```

Example:

```text
Stock = 10

A locks row
A: 10 -> 9
A commits

B gets lock
B sees 9
B: 9 -> 8
```

---

# 4. Optimistic vs Pessimistic

| Optimistic Locking                   | Pessimistic Locking                    |
| ------------------------------------ | -------------------------------------- |
| Assume conflicts are rare            | Assume conflicts are likely            |
| Does not explicitly lock during read | Locks the row                          |
| Usually uses `@Version`              | Usually uses `SELECT FOR UPDATE`       |
| Detect conflict during update        | Prevent concurrent modification        |
| Conflict causes failure/retry        | Other transaction waits                |
| Higher concurrency                   | Lower concurrency                      |
| Good for normal CRUD                 | Useful for highly contested operations |
| No long-running row locks            | Locks remain until transaction ends    |

Easy way to remember:

```text
Optimistic:
"Go ahead. I'll check later if someone changed it."

Pessimistic:
"Nobody touch this until I'm done."
```

---

# 5. Why Not Use Pessimistic Locking Everywhere?

Pessimistic locking works, but introduces several production problems.

## Reduced Concurrency

Requests touching the same row become serialized.

```text
Request 1 -> gets lock

Request 2 -> waits
Request 3 -> waits
Request 4 -> waits
...
```

Instead of processing concurrently:

```text
A
B
C
D
```

you effectively get:

```text
A -> B -> C -> D
```

This becomes a major problem for **hot rows**.

---

## Increased Latency

Suppose each transaction holds the lock for:

```text
20 ms
```

and 100 requests want the same row.

Approximately:

```text
100 × 20 ms
= 2 seconds
```

The requests become serialized around that resource.

With 1000 requests:

```text
1000 × 20 ms
= ~20 seconds
```

So a hot database row can become a bottleneck.

---

# 6. Connection Pool Exhaustion

This is one of the more dangerous consequences.

Suppose:

```text
Hikari connection pool = 50
```

One transaction holds a row lock.

Another 49 requests are waiting for that lock.

Those waiting transactions can still occupy database connections.

```text
1 connection -> doing work

49 connections -> waiting for lock
```

Now:

```text
DB connection pool exhausted
```

Other unrelated APIs may be unable to get a database connection.

This creates a cascading problem:

```text
hot row
   ↓
lock contention
   ↓
requests wait
   ↓
DB connections occupied
   ↓
connection pool exhausted
   ↓
other APIs become slow
```

---

# 7. Deadlocks

Pessimistic locking also increases the possibility of deadlocks.

Example:

```text
Transaction A:
locks Row 1
then needs Row 2

Transaction B:
locks Row 2
then needs Row 1
```

Now:

```text
A waits for B

B waits for A
```

This is a deadlock.

The database detects it and usually aborts one of the transactions.

A common prevention technique is:

```text
Always acquire locks in a consistent order.
```

For example:

```text
Always lock smaller ID first.
```

---

# 8. Lock Timeouts

A transaction may wait too long for another transaction to release a lock.

Example:

```text
A holds lock for 5 seconds

B waits
C waits
D waits
```

Eventually some requests may hit:

```text
lock timeout
transaction timeout
request timeout
```

and fail.

---

# 9. Long Transactions Are Dangerous

A pessimistic lock exists until the database transaction finishes.

Therefore this is dangerous:

```java
@Transactional
public void updateOrder(Long id) {

    Order order = repository.findByIdForUpdate(id);

    callExternalService();

    order.setStatus(COMPLETED);
}
```

Suppose:

```text
External API call = 2 seconds
```

The database row stays locked for those 2 seconds.

If the external service becomes slow:

```text
2 sec
↓
5 sec
↓
10 sec
```

the database lock is also held longer.

This can produce:

```text
long transactions
+
lock waits
+
connection exhaustion
+
request timeout
```

General rule:

```text
Do not perform slow network calls while holding DB locks.
```

---

# 10. Optimistic Locking Under Contention

Optimistic locking behaves differently.

Suppose 100 requests read:

```text
version = 5
```

They can all read concurrently.

One transaction updates successfully:

```text
version 5 -> 6
```

The others trying to update version `5` fail.

So instead of:

```text
99 requests waiting
```

you may have:

```text
1 succeeds
99 receive conflict/retry
```

This provides higher concurrency but introduces **retry handling**.

Therefore optimistic locking is best when:

```text
conflicts are relatively rare
```

because continuously retrying under very high contention can also become inefficient.

---

# 11. Atomic SQL Update Can Be Better Than Both

For simple operations, you may not need explicit optimistic or pessimistic locking.

Example: decrement stock.

Instead of:

```text
read stock
↓
lock/check version
↓
calculate
↓
write stock
```

use an atomic SQL operation:

```sql
UPDATE product
SET stock = stock - 1
WHERE id = ?
  AND stock > 0;
```

Then check affected rows.

```text
rows updated = 1
    -> success

rows updated = 0
    -> out of stock
```

Spring example:

```java
@Modifying
@Query("""
    UPDATE Product p
    SET p.stock = p.stock - 1
    WHERE p.id = :id
      AND p.stock > 0
""")
int decrementStock(Long id);
```

Usage:

```java
int updated = repository.decrementStock(id);

if (updated == 0) {
    throw new OutOfStockException();
}
```

The database performs the condition and update atomically.

This is often better for:

```text
counters
inventory decrement
state transitions
quota decrement
```

---

# 12. Which One Should You Use?

A useful rule of thumb:

```text
Normal CRUD
Low probability of concurrent conflicts
        ↓
Optimistic Locking
```

```text
Simple counter / conditional update
        ↓
Atomic SQL UPDATE
```

```text
High contention
+
operation requires reading current state
+
several related checks/updates must happen together
        ↓
Pessimistic Locking
```

---

# 13. Example: Concert Ticket

Suppose:

```text
Only 1 ticket remains.
```

## Optimistic locking

```text
A reads:
tickets = 1
version = 10

B reads:
tickets = 1
version = 10
```

A updates:

```text
tickets = 0
version = 11
```

B tries:

```text
WHERE version = 10
```

Update fails.

B receives:

```text
ticket no longer available
```

---

## Pessimistic locking

```text
A locks ticket row.

B waits.

A:
tickets 1 -> 0
COMMIT

B acquires lock.

B sees:
tickets = 0

B cannot book.
```

Both approaches protect correctness, but the concurrency behavior is different.

---

# Final Mental Model

```text
OPTIMISTIC LOCKING

Concurrent work allowed
        ↓
detect conflict later
        ↓
one succeeds
others retry/fail
```

```text
PESSIMISTIC LOCKING

Acquire DB lock first
        ↓
others wait
        ↓
execute operation
        ↓
commit
        ↓
release lock
```

Main trade-off:

```text
Optimistic
= more concurrency + possible retries

Pessimistic
= fewer update conflicts + more blocking
```

The main reason **not to use pessimistic locking everywhere** is:

```text
Lock contention
    ↓
waiting requests
    ↓
higher latency
    ↓
occupied DB connections
    ↓
possible pool exhaustion
    ↓
deadlocks / timeouts
    ↓
cascading system slowdown
```

So pessimistic locking is a tool for cases where serialization is actually required, not a default solution for every concurrent update.
