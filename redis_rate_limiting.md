# Redis Rate Limiting

## Goal

Continue the discussion on **Redis-based rate limiting** for **100k+ QPS production systems**, focusing on architecture and production design (not code yet).

---

# Current Understanding

## High-level flow

```text
Client Request
      ↓
API Gateway / Rate Limiter Service
      ↓
Match request to rate-limit rule
      ↓
Generate Redis key
      ↓
Execute Redis Lua script
      ↓
Allowed → Forward request
Rejected → Return 429 / Queue / Drop
```

Responsibilities are split as follows:

* **Rule matching** → Gateway / Rate Limiter Service
* **Rule configuration** → Config service / DB / Gateway config (cached in memory)
* **Runtime state** → Redis
* **Atomic algorithm execution** → Redis Lua
* **Allow / Reject decision** → Gateway / Rate Limiter Service

---

# Rule Configuration

Redis generally **does not store the rate-limit rules**.

Example rules:

```
POST /login  -> 5 req/min/user
GET /search -> 100 req/min/user
POST /payment -> 10 req/min/user
```

These rules usually live in:

* Gateway configuration
* Config Service
* Database (cached in memory)
* Admin-managed rule engine

The gateway performs **path matching** before Redis is called.

Priority is generally:

1. Exact path
2. Route template (`/users/{id}`)
3. Prefix / wildcard
4. Global default

---

# Redis Stores Runtime State

Redis stores only dynamic information.

Example:

```
rl:login:user:123

tokens = 3
last_refill = timestamp
TTL = 2 minutes
```

Different APIs generate different keys.

Example:

```
rl:login:user:123
rl:search:user:123
```

Therefore `/login` and `/search` have completely independent buckets.

---

# Redis Concepts Used

## 1. Keys

Used to identify each independent limiter.

Example:

```
rl:<rule>:<identity_type>:<identity>
```

Examples:

```
rl:login:user:123
rl:search:user:123
rl:payment:tenant:amazon
```

---

## 2. TTL

Used to automatically remove inactive buckets.

No cleanup job is usually required.

---

## 3. Atomic Operations

Simple counters use Redis atomic commands.

Examples:

* INCR
* DECR
* EXPIRE

Complex algorithms require Lua.

---

## 4. Lua Scripts

Lua executes **inside Redis**.

Instead of:

```
GET
calculate
SET
EXPIRE
```

the gateway performs one `EVAL`.

Lua executes atomically:

```
Read state
Calculate refill
Check limit
Update state
Return allow/reject
```

Benefits:

* One network round trip
* Atomic
* No race conditions
* No distributed locking

---

## 5. Redis Time

Rate limiting depends on timestamps.

Production systems often use Redis/server-side time to avoid clock drift across application servers.

---

## 6. Redis Data Structures

### String

Typically used for fixed-window counters.

```
count = 7
```

---

### Hash

Useful for token bucket.

Stores:

```
tokens
last_refill_timestamp
```

---

### Sorted Set

Useful for Sliding Window Log.

Stores every request timestamp.

Flow:

* Remove expired timestamps
* Count remaining requests
* Add current request
* Decide allow/reject

More accurate than token bucket but more memory-intensive.

---

## 7. Redis Cluster

Large-scale systems distribute limiter keys across shards.

Good:

```
rl:user:1
rl:user:2
rl:user:3
```

Bad (hot key):

```
rl:global
```

Global rate limits require additional design because all requests hit the same key.

---

## 8. Pipelining

Useful when checking multiple independent limits.

Example:

* User limit
* IP limit
* Tenant limit

Lua already minimizes round trips for a single limiter.

---

# Token Bucket Refill

A major clarification:

There is **usually no background Java scheduler** continuously refilling Redis buckets.

Instead, refill is **lazy**.

When a request arrives:

```
elapsed = current_time - last_refill
tokens += elapsed × refill_rate
tokens = min(capacity, tokens)
```

Then:

```
if tokens >= cost
    decrement
    allow
else
    reject
```

Therefore:

* No Redis scan
* No scheduled refill job
* No unnecessary writes

Refill is simply a mathematical calculation performed when a request arrives.

---

# Multi-Level Rate Limiting

Production systems often evaluate multiple limits.

Example:

* Per User
* Per IP
* Per API Key
* Per Tenant
* Global

Request is allowed only if **all** configured limits pass.

---

# Questions Already Answered

### 1. Where does constant refill happen?

Usually nowhere.

Buckets are lazily refilled during request processing using elapsed time.

---

### 2. Why Lua?

Lua executes the entire rate-limit algorithm atomically inside Redis:

```
Read
Refill
Check
Update
Return
```

avoiding race conditions.

---

### 3. Where are path-wise rules stored?

Usually outside Redis.

Gateway matches request → selects rule → constructs Redis key → invokes Redis.

---

### 4. Where is path matching performed?

Inside the Gateway / Rate Limiter Service.

Usually using in-memory route configuration.

---

### 5. Where is the rate-limiting algorithm executed?

Split responsibilities:

```
Gateway:
    Match rule
    Build Redis key
    Invoke Redis

Redis Lua:
    Execute token/sliding-window algorithm
    Update state atomically

Gateway:
    Allow or reject request
```

---

# Next Discussion

Continue with **implementation concepts only (no code initially)**:

1. Token Bucket algorithm in Redis (internal state transitions)
2. Sliding Window algorithm in Redis
3. Comparison:

    * Fixed Window
    * Sliding Window Counter
    * Sliding Window Log
    * Token Bucket
    * Leaky Bucket
4. Redis Lua lifecycle:

    * Script loading
    * EVAL vs EVALSHA
    * Replication
    * Cluster behavior
5. Redis Cluster considerations:

    * Hot keys
    * Hash slots
    * Multi-key Lua limitations
6. Failure scenarios:

    * Redis unavailable
    * Cluster failover
    * Network partition
    * Fail-open vs Fail-closed strategies
7. Production optimizations at 100k+ QPS:

    * Local + Redis hybrid rate limiting
    * Multi-level limit evaluation
    * Caching rule configurations
    * Gateway implementations (NGINX, Envoy, Spring Cloud Gateway, etc.)
