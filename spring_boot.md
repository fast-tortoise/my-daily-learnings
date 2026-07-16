# Spring Boot Interview Prep — Session Summary

## Progress So Far

Topics covered:

1. Dependency Injection (DI) & IoC
2. Spring Bean Lifecycle & ApplicationContext
3. Circular Dependencies
4. Spring Proxies & AOP
5. `@Transactional` Internals
6. `@Component` vs `@Bean` vs `@Configuration`

Overall assessment:
- Good practical Spring experience.
- Need deeper understanding of Spring internals (container, proxies, AOP, bean lifecycle), which are common in Senior Backend interviews.

---

# 1. Dependency Injection (DI)

## What is DI?

Dependency Injection is a design pattern used to implement **Inversion of Control (IoC)**.

Instead of a class creating its own dependencies using `new`, the Spring IoC Container creates and injects them.

```java
class OrderService {
    private final PaymentService paymentService;

    OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

## Benefits

- Loose coupling
- Easier unit testing
- Easier mocking
- Better maintainability
- Dependency Inversion Principle (SOLID)
- Explicit dependencies

---

## Constructor vs Field Injection

### Constructor Injection (Preferred)

Advantages

- Mandatory dependencies
- Immutable (`final`) fields
- Easier testing
- Recommended by Spring
- Prevents partially initialized objects

---

### Field Injection

Disadvantages

- Uses reflection
- Hidden dependencies
- Cannot use `final`
- Harder to unit test
- Generally discouraged

---

# 2. Spring Bean Lifecycle

Application starts:

```text
SpringApplication.run()
        ↓
ApplicationContext created
        ↓
Component Scan
        ↓
BeanDefinitions created
        ↓
Bean instantiation
        ↓
Dependency resolution
        ↓
Singleton cache
        ↓
Application ready
```

---

## ApplicationContext

ApplicationContext is Spring's IoC Container.

Responsibilities

- Stores beans
- Creates beans
- Injects dependencies
- Lifecycle management
- Events
- Configuration management

Conceptually

```text
ApplicationContext

OrderService
PaymentService
UserRepository
KafkaProducer
```

---

## BeanDefinition

During scanning Spring **does not create objects immediately**.

Instead it stores metadata.

Example

```text
BeanDefinition

Class = PaymentService

Scope = Singleton

Lazy = false
```

---

## Bean Creation

Spring later creates the actual object.

```java
new PaymentService();
```

---

## Singleton Cache

Singleton beans are stored in an internal cache.

Conceptually

```text
singletonObjects

PaymentService
OrderService
UserRepository
```

Whenever another bean needs PaymentService, Spring returns the existing instance instead of creating a new one.

---

# 3. Circular Dependency

Example

```java
@Service
class AService {
    AService(BService b){}
}

@Service
class BService {
    BService(AService a){}
}
```

---

## Constructor Injection

Fails.

Reason:

```text
Need A
↓

Need B
↓

Need A again
↓

Impossible to construct
```

Failure occurs during **bean creation**, not component scanning.

---

## Field Injection

Historically Spring could resolve some circular dependencies by:

```text
Create empty object
↓

Store early reference
↓

Inject fields later
```

However,

**Spring Boot 2.6+ disables circular references by default**, so field injection also commonly fails unless explicitly enabled.

---

## Solution

Use

```java
@Lazy
```

or redesign to remove the circular dependency.

---

# 4. Spring Proxy

## What is a Proxy?

Instead of client calling the bean directly,

```text
Client
    ↓
Proxy
    ↓
Target Bean
```

The proxy intercepts method calls.

---

## Why proxies?

To implement cross-cutting concerns:

- Transactions
- Logging
- Security
- Retry
- Caching
- Async execution
- Metrics

---

## Proxy execution

Instead of

```java
paymentService.pay();
```

Proxy performs

```java
before();

target.pay();

after();
```

---

## Are all beans proxied?

No.

Only beans requiring AOP features.

Examples

```java
@Transactional

@Async

@Cacheable

@PreAuthorize
```

Beans without such features are typically normal objects.

---

# 5. @Transactional Internals

Suppose

```java
@Transactional
public void methodA() {
    methodB();
}

@Transactional
public void methodB() {}
```

---

## External call

```text
Client
    ↓
Proxy
    ↓
methodA()
```

Proxy starts transaction.

---

## Internal call

Inside methodA

```java
methodB();
```

becomes

```java
this.methodB();
```

which bypasses the proxy.

Therefore:

- No new transaction interception
- No logging advice
- No cache advice
- No retry advice
- No security advice

This is called **Self Invocation**.

---

## REQUIRES_NEW

Even

```java
@Transactional(propagation = REQUIRES_NEW)
```

does **not** work through self invocation because the proxy is bypassed.

Correct approach:

Move transactional logic into another Spring bean.

---

# Self Injection

Example

```java
@Autowired
private UserService self;
```

Calling

```java
self.methodB();
```

goes through the proxy instead of `this`.

However,

This creates a circular dependency and is generally discouraged.

Preferred approach:

Move methodB into another service.

---

# Class-level @Transactional

Allowed.

```java
@Service
@Transactional
class UserService
```

All eligible public methods become transactional.

Method-level annotations override class-level configuration.

Common pattern

```java
@Transactional(readOnly = true)

find()

@Transactional

save()
```

---

# What gets injected?

Suppose another bean autowires UserService.

Spring usually injects:

```text
UserService Proxy
```

not the original object.

Internally

```text
Proxy
    ↓
Original Target Bean
```

The proxy delegates calls to the target.

---

# 6. @Component vs @Bean vs @Configuration

---

## @Component

Automatic bean discovery.

```java
@Component
class PaymentService {}
```

Used for application classes you own.

Examples

- Service
- Repository
- Controller
- Utility Components

---

## @Bean

Manual bean registration.

```java
@Bean
ObjectMapper mapper() {
    return new ObjectMapper();
}
```

Useful for

- Third-party classes
- SDK clients
- Custom object construction

Examples

- ObjectMapper
- ExecutorService
- KafkaProducer
- RedisClient
- DataSource

---

## Why not @Bean everywhere?

Technically possible.

But becomes verbose for hundreds of application classes.

Therefore

- `@Component` → automatic
- `@Bean` → manual

---

## @Configuration

Specialized configuration class.

Internally behaves like

```text
@Component
+
CGLIB Proxy
```

Purpose:

Ensure singleton semantics between `@Bean` methods.

Example

```java
@Bean
A a(){}

@Bean
B b(){
    return new B(a());
}
```

Without `@Configuration`

```java
a()
```

is a normal Java call and may create another object.

With `@Configuration`

Proxy intercepts the call and returns the existing singleton bean.

---

# Important Concepts Learned

- DI is an implementation of IoC.
- Spring first creates BeanDefinitions, then actual objects.
- ApplicationContext is the IoC container responsible for bean management.
- Singleton beans are stored in an internal singleton cache.
- Circular dependency failure occurs during bean creation, not scanning.
- Spring proxies implement transactions, AOP, caching, async execution, and security.
- Self invocation (`this.method()`) bypasses proxies.
- Other beans receive the proxy, not the original target bean.
- `@Configuration` uses a CGLIB proxy to preserve singleton behavior among `@Bean` methods.

---

# Areas to Improve

Need deeper understanding of:

- AOP internals
- Proxy creation (JDK vs CGLIB)
- Bean lifecycle callbacks (`@PostConstruct`, `InitializingBean`, `BeanPostProcessor`)
- Transaction propagation
- Spring Boot auto-configuration
- Bean scopes
- Conditional beans
- Spring Security internals
- Event mechanism
- Async execution internals
- Request lifecycle
- DispatcherServlet
- Auto-configuration internals
- Starter mechanism

These are good next topics for senior Spring Boot interview preparation.