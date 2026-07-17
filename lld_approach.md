# LLD Interview Framework (45 Minutes)

## Biggest Learning

My weakness is **not backend engineering**, it is **starting from APIs and databases instead of the business domain**.

Current thought process:

```text
Requirements
    ↓
API
    ↓
Database
    ↓
Service
    ↓
Controller
```

Expected interview thought process:

```text
Requirements
    ↓
Business Flow
    ↓
Domain Objects
    ↓
Responsibilities
    ↓
Services
    ↓
APIs
    ↓
Persistence
    ↓
Concurrency
    ↓
Patterns / Extensibility
```

---

# 1. Clarify Requirements (5 min)

Ask:

* Functional requirements
* Non-functional requirements
* Scale
* Assumptions
* Out of scope

Goal:

> Understand the problem completely before designing.

---

# 2. Explain Main Flows (3 min)

Describe only the business flow.

Example:

```text
Entry

Vehicle arrives
    ↓
Allocate Spot
    ↓
Generate Ticket
    ↓
Park
```

```text
Exit

Ticket scanned
    ↓
Calculate Fee
    ↓
Release Spot
    ↓
Exit
```

Goal:

> Show understanding of the business process.

---

# 3. Domain Modeling (10 min)

Forget:

* Spring Boot
* REST APIs
* Database

Ask:

> **If there were no database, what business objects would still exist?**

Example:

```text
ParkingLot
 ├── ParkingFloor
 │      ├── ParkingSpot
 │
 ├── EntryGate
 ├── ExitGate
 ├── Vehicle
 └── Ticket
```

Think about:

* State
* Relationships
* Ownership

This is **Domain Modeling**, not database design.

---

# Domain Modeling vs Database Modeling

## Domain Modeling

Question:

> What exists in the business?

Focus on:

* Objects
* State
* Behavior
* Relationships

Example:

```java
ParkingSpot

Fields:
- spotNumber
- vehicleType
- status

Methods:
- occupy()
- release()
- canFit(vehicle)
```

## Database Modeling

Question:

> How do I persist data?

Focus on:

* Tables
* Primary keys
* Foreign keys
* Indexes
* Queries

Example:

```sql
parking_spot

spot_id
floor_id
vehicle_type
available
```

Important:

> Domain objects exist even if there is no database.

---

# 4. Responsibilities (7 min)

Ask:

> **Who should own this behavior?**

Example:

Allocate Spot

↓

SpotAllocationService

Calculate Price

↓

PricingService

Generate Ticket

↓

TicketService

Do **not** create services first.

Services should emerge naturally from responsibilities.

---

# 5. APIs (3 min)

Expose the business operations.

Example:

```text
POST /entry

POST /exit
```

Keep APIs simple.

---

# 6. Persistence (5 min)

Only now think about storage.

Design:

* Tables
* Keys
* Relationships
* Indexes

Question:

> What needs to survive a restart?

---

# 7. Concurrency (5 min)

Think:

* Race conditions
* Transactions
* Locks
* Idempotency

Parking Lot example:

Multiple gates can allocate the same spot.

Discussion should include:

* Atomic allocation
* Row-level locking
* `SKIP LOCKED`
* Transactions

Goal:

> Ensure correctness under concurrent requests.

---

# 8. Extensibility (5 min)

Only after the core design.

Examples:

Dynamic Pricing

↓

Strategy Pattern

Multiple Allocation Algorithms

↓

Strategy Pattern

Multiple Notification Channels

↓

Observer Pattern

Never introduce patterns unless a requirement demands them.

---

# Layered Thinking

## Layer 1

Business

"What exists?"

↓

Domain Objects

---

## Layer 2

Behavior

"Who should do this?"

↓

Responsibilities

---

## Layer 3

Patterns

"Will multiple implementations exist?"

↓

Strategy / Observer / Factory etc.

---

## Layer 4

Framework

↓

Spring Boot

Controllers

Repositories

---

# One Service vs Multiple Services

Both are acceptable if justified.

## Simple Version

```text
ParkingService

reserveSpot()

releaseSpot()

calculatePrice()
```

Good answer:

> "For V1, the logic is simple, so I'll keep a single service. If pricing or allocation rules become more complex, I'll extract dedicated services."

Avoid unnecessary abstraction.

## Separate Services

Use when responsibilities evolve independently.

Example:

```text
SpotAllocationService

PricingService

TicketService
```

Reason:

* Allocation changes independently.
* Pricing changes independently.

The interviewer evaluates the reasoning, not the number of services.

---

# Three Questions to Ask Continuously

## 1. Domain

> What exists?

Example:

* ParkingLot
* ParkingSpot
* Vehicle
* Ticket

---

## 2. Responsibility

> Who should do this?

Example:

Calculate Price

↓

PricingService

---

## 3. Persistence

> What needs to be stored?

Example:

* ParkingSpot
* Ticket

---

# Biggest Mindset Shift

Do **not** start with:

```text
API

↓

Database

↓

Service
```

Instead start with:

```text
Business

↓

Objects

↓

Behavior

↓

Persistence
```

---

# Final LLD Flow

```text
1. Clarify Requirements
2. Explain Business Flow
3. Discover Domain Objects
4. Assign Responsibilities
5. Design Services
6. Expose APIs
7. Design Database
8. Handle Concurrency
9. Apply Patterns (only if required)
10. Discuss Future Extensions
```

---

# Key Takeaway

> **Don't ask "How do I build it?"**

Instead ask:

> **"What exists in the business, who owns the behavior, and only then how should it be implemented?"**

Everything else follows naturally.
