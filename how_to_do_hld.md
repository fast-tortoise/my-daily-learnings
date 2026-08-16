Use this as your **default HLD interview flow**:

```text
1. Clarify scope
        ↓
2. Functional requirements
        ↓
3. Non-functional requirements
        ↓
4. Rough scale estimation
        ↓
5. Core entities + external APIs
        ↓
6. Build high-level architecture
        ↓
7. Identify logical services / boundaries
        ↓
8. Decide communication on each interaction
        ↓
9. Decide data ownership / storage
        ↓
10. Walk critical flow end-to-end
        ↓
11. Identify hard problems / bottlenecks
        ↓
12. Deep dive into 1–2 important areas
        ↓
13. Add required infra/patterns
        ↓
14. Consistency + concurrency
        ↓
15. Failure handling
        ↓
16. Scaling
        ↓
17. Trade-offs + final summary
```

### The important way to think about it

**1. Scope first**

Example:

> “For Uber, I’ll focus on requesting a ride, finding a driver, driver acceptance, trip tracking and completion.”

Don't design everything Uber has.

**2. Requirements**

```text
Functional:
- Request ride
- Find nearby driver
- Accept ride
- Track ride
- Complete ride

NFR:
- Low latency matching
- Highly available
- Scalable location updates
- Correct driver assignment
```

**3. Estimate only useful scale**

```text
Concurrent drivers
Ride requests/sec
Location updates/sec
```

Use these numbers later to justify design decisions.

**4. Entities + APIs**

```text
Rider
Driver
Ride
Location

POST /rides
POST /rides/{id}/accept
POST /drivers/location
```

Keep this short.

---

### 5–9 are really one big architecture phase

Start drawing the system.

```text
App
 |
Gateway
 |
Ride Service
 |
Matching Service
 |
Location Service
```

While drawing, answer four questions for every component:

```text
What does it do?
What data does it own?
Who calls it?
How do they communicate?
```

For example:

```text
Ride Service
- owns ride lifecycle
- owns Ride DB
- receives HTTP requests

Matching Service
- finds drivers
- talks to Location Service

Location Service
- owns latest driver location
- high-write workload
```

Then label communication:

```text
HTTP / gRPC
```

when the caller needs an immediate response.

```text
Kafka / queue
```

when asynchronous processing, buffering, fan-out or eventual consistency makes sense.

**Don't start by declaring 15 microservices.**

Think:

```text
Requirement
   ↓
Responsibility
   ↓
Logical component
   ↓
Data ownership
   ↓
Communication
```

---

### 10. Walk the main flow

For Uber:

```text
Rider
  ↓
POST /rides
  ↓
Ride Service
  ↓
Create ride
  ↓
Matching Service
  ↓
Location Service
  ↓
Find nearby drivers
  ↓
Notify driver
  ↓
Driver accepts
  ↓
Assign ride
```

This often exposes the actual design problems.

---

### 11–12. Find the hard parts and deep dive

For Uber:

```text
How do I find nearby drivers?

How do I process millions of location updates?

What if 2 rides choose the same driver?

What if 2 drivers accept?

What if matching takes too long?
```

Pick the most important 1–2 and go deep.

---

### 13. Introduce infrastructure only when justified

Don't say:

```text
Kafka
Redis
Cassandra
Elasticsearch
```

just because you're doing HLD.

Instead:

```text
Problem:
Matching requests can spike and Matching Service may temporarily fail.

Solution:
Queue/Kafka between ride creation and matching.
```

Or:

```text
Problem:
Driver location has huge write volume and needs geo lookup.

Solution:
Dedicated geo/location store.
```

So:

```text
Problem → requirement → technology
```

not:

```text
Technology → find somewhere to use it
```

---

### 14–16. Senior-level discussion

Now cover:

```text
Consistency
Concurrency
Idempotency
Retries
Timeouts
Partial failures
Duplicate events
Service crashes
DB failures
Scaling
Partitioning
Caching
Hotspots
```

For example:

```text
Driver AVAILABLE

Ride A → tries to assign
Ride B → tries to assign
```

Only one should succeed.

That becomes a concurrency/consistency discussion.

---

### 17. Finish with trade-offs

End with 2–3 important decisions:

> Driver location is eventually consistent because slight staleness is acceptable.

> Driver assignment requires stronger consistency because a driver cannot be assigned to two rides.

> Location processing is separated because its write workload is dramatically different from normal ride transactions.

---

## The short version to memorize

```text
Scope
 ↓
Requirements
 ↓
NFR + Scale
 ↓
Entities + APIs
 ↓
Architecture
 ↓
Services + Data Ownership
 ↓
Sync vs Async Communication
 ↓
Main Flow
 ↓
Hard Problems
 ↓
Deep Dive
 ↓
DB / Cache / Kafka / Patterns
 ↓
Consistency + Concurrency
 ↓
Failures
 ↓
Scaling
 ↓
Trade-offs
```

And the most important interview rule:

> **First make a simple working design. Then let scale, failures and requirements force you to make it more sophisticated.**

That prevents both under-designing and throwing Kafka/Redis/microservices at the problem prematurely.
