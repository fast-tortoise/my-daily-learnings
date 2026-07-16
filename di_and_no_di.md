# Dependency Injection (DI) vs No DI

## Core Idea

Dependency Injection is **not about annotations or reducing code**.

It is a design principle:

> A class should **not create its own dependencies**. Instead, dependencies should be provided (injected) from outside.

Spring is simply a framework that automates this process.

---

# 1. No Dependency Injection

```java
class PaymentService {
    private final PaymentGateway gateway = new StripeGateway();
}
```

Here, `PaymentService` is responsible for creating its own dependency.

### Characteristics

- Tight coupling between `PaymentService` and `StripeGateway`
- Changing implementation requires modifying the class itself.
- Difficult to unit test because dependencies cannot easily be replaced.
- Business logic and object creation are mixed together.
- Lifecycle of the dependency is controlled by the class.

Object graph:

```
PaymentService
      |
      ---> new StripeGateway()
```

---

# 2. Manual Dependency Injection

Instead of creating dependencies internally:

```java
class PaymentService {

    private final PaymentGateway gateway;

    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

Someone else creates the dependency:

```java
PaymentGateway gateway = new StripeGateway();
PaymentService service = new PaymentService(gateway);
```

### Characteristics

- `PaymentService` no longer knows which implementation it receives.
- Object creation is separated from business logic.
- Easy to replace dependencies.
- Easy to mock during testing.

This is **Dependency Injection**, even without Spring.

---

# 3. Spring Dependency Injection

Spring performs the same wiring automatically.

Instead of writing:

```java
PaymentGateway gateway = new StripeGateway();
PaymentService service = new PaymentService(gateway);
```

Spring:

1. Scans classes (`@Component`, `@Service`, etc.)
2. Creates objects.
3. Resolves constructor dependencies.
4. Injects them.
5. Manages their lifecycle.

Business code remains focused only on business logic.

---

# DI vs No DI

| No DI | Dependency Injection |
|--------|----------------------|
| Class creates dependencies | Dependencies provided externally |
| Tight coupling | Loose coupling |
| Difficult to test | Easy to mock and test |
| Object creation mixed with business logic | Object creation separated from business logic |
| Hard to replace implementations | Easy to swap implementations |
| Class manages dependency lifecycle | Container/bootstrap code manages lifecycle |

---

# Why DI is Better

## Loose Coupling

Business classes depend on interfaces rather than concrete implementations.

Today:

```
StripeGateway
```

Tomorrow:

```
RazorpayGateway
```

Only wiring changes; business code remains unchanged.

---

## Better Testability

Without DI:

```java
new StripeGateway();
```

Cannot replace with a mock.

With DI:

```java
PaymentGateway mock = Mockito.mock(...);

PaymentService service = new PaymentService(mock);
```

No database, Kafka, or external services are required for testing.

---

## Separation of Responsibilities

Business class:

- Performs business logic.

Bootstrap/Spring:

- Creates objects.
- Resolves dependencies.
- Wires everything together.
- Manages lifecycle.

Each component has a single responsibility.

---

## Lifecycle Management

Without Spring:

Every class can create its own instance.

```
new DBConnection()
new DBConnection()
new DBConnection()
```

Potentially hundreds of unnecessary objects.

With Spring Singleton scope:

```
One DBConnection
      ↓
Shared across all services
```

Spring creates and manages the instance.

---

# Spring IoC Container

Spring's IoC (Inversion of Control) container:

```
Scan classes
      ↓
Create objects
      ↓
Resolve dependencies
      ↓
Inject dependencies
      ↓
Manage lifecycle
      ↓
Destroy beans on shutdown
```

Dependency Injection is just one capability provided by the IoC container.

---

# Important Distinction

**Dependency Injection** is a design principle.

**Spring** is a framework that automates Dependency Injection.

DI exists even if Spring does not.

---

# Interview Definition

> Dependency Injection is a design principle where a class receives its dependencies from an external source instead of creating them itself. This separates object creation from business logic, resulting in loose coupling, better testability, easier replacement of implementations, and allowing frameworks like Spring to manage object lifecycle and dependency wiring automatically.