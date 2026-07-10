# Server Threads vs Application Threads — Continuation Notes

## Goal

Understand the difference between Tomcat/server threads and application-managed threads, how they interact, how thread pools are sized, and how endpoint-level isolation can be implemented.

---

## Core Idea

Tomcat threads and application threads are not physically different.

Both are normal JVM/platform threads scheduled by the operating system.

The difference is ownership and responsibility:

- **Tomcat threads** are managed by the web server and process incoming HTTP requests.
- **Application threads** are managed by application-level executors such as `ExecutorService`, `@Async`, `CompletableFuture`, Kafka consumers, and schedulers.

---

## Synchronous Spring MVC Request Flow

```text
Client
  ↓
Tomcat request thread
  ↓
Controller
  ↓
Service
  ↓
Repository / downstream call
  ↓
Response
```

In a normal synchronous request, the same Tomcat thread usually remains occupied until the response is completed.

If Tomcat has 20 request threads, approximately 20 synchronous requests can execute concurrently. Additional requests wait in queues or the network backlog.

---

## Application Threads

Application threads are used for work such as:

- Background jobs
- Kafka consumption
- Scheduled tasks
- Parallel computation
- Asynchronous HTTP calls
- Email or file processing
- Long-running tasks

Example pools:

```text
Tomcat request pool: 20 threads
Background executor: 100 threads
Kafka consumers: 10 threads
Scheduler: 5 threads
```

These pools are independent, but they still compete for the same CPU, memory, database connections, and downstream resources.

---

## Asynchronous Request Flow

```text
Client
  ↓
Tomcat thread
  ↓
Submit work to application executor
  ↓
Application thread performs work
  ↓
Response is completed asynchronously
```

Moving work to another executor helps only when the Tomcat thread is actually released.

Simply calling another thread and then blocking with `join()` or `get()` still keeps the Tomcat thread occupied.

---

## Can Threads Be Configured Per Endpoint?

Tomcat normally uses one shared request-thread pool per connector.

Therefore, this is not directly supported:

```text
GET endpoints  → 100 Tomcat threads
POST endpoints → 20 Tomcat threads
```

All endpoints usually compete for the same Tomcat pool.

Endpoint isolation can instead be implemented using:

1. Separate application executors
2. Semaphore or bulkhead concurrency limits
3. Rate limiting
4. Separate services or deployments
5. Separate Tomcat connectors, although this is less common

Example:

```text
GET /reports   → report executor with 100 threads
POST /payment  → payment executor with 20 threads
```

A semaphore-based bulkhead is often better than creating many separate thread pools when the main requirement is only to limit concurrency.

---

## CPU-Bound Workloads

Examples:

- Compression
- Encryption
- Image processing
- Heavy calculations
- Large in-memory transformations

For CPU-bound work:

```text
Thread count ≈ number of CPU cores
```

Sometimes `cores + 1` is used.

Too many CPU-bound threads cause:

- Context switching
- CPU cache misses
- Scheduling overhead
- Higher latency
- Lower throughput

True CPU parallelism is limited by the available CPU cores.

---

## I/O-Bound Workloads

Examples:

- Database calls
- Redis calls
- External HTTP calls
- File or network I/O

An I/O-bound thread spends much of its time waiting, so thread count may be much higher than the CPU core count.

Approximation:

```text
Threads ≈ CPU cores × (1 + wait time / compute time)
```

Example:

```text
CPU cores = 8
Compute time = 5 ms
I/O wait time = 45 ms

Threads ≈ 8 × (1 + 45 / 5)
        ≈ 80
```

This is only a starting estimate. Production sizing must be validated through load testing.

---

## What Determines Tomcat Thread Count?

Tomcat thread count should not be based on CPU cores alone.

Important constraints include:

### 1. CPU

Too many runnable threads increase context switching and CPU contention.

### 2. Memory

Each platform thread has stack memory and may retain:

- Request objects
- Local variables
- Security context
- Logging MDC
- JSON buffers
- Hibernate entities

Large thread pools increase both native memory and heap pressure.

### 3. Database Connection Pool

Example:

```text
Tomcat threads = 200
DB connections = 20
```

If every request needs a database connection:

```text
20 requests use the database
180 requests wait for a connection
```

Increasing Tomcat threads does not increase database throughput.

### 4. Downstream Connection Pools

Examples:

- HTTP client max connections
- Redis connection pool
- External API limits
- Kafka producer constraints

The smallest constrained downstream pool may become the real bottleneck.

### 5. Request Latency

Using Little's Law:

```text
Concurrency ≈ throughput × latency
```

Example:

```text
Target throughput = 1,000 requests/second
Average latency = 200 ms = 0.2 seconds

Required concurrency ≈ 1,000 × 0.2
                     ≈ 200 requests
```

Higher latency requires more in-flight request capacity.

### 6. Queue and Backlog Size

When all Tomcat threads are busy, requests may wait in:

- Tomcat accept queue
- Operating-system socket backlog
- Load balancer queue
- API gateway queue

Very large queues can cause:

- High response latency
- Timeouts before execution starts
- Retry storms
- Memory pressure

Bounded queues and fast rejection are often safer.

### 7. Timeout Budgets

Thread-pool size and queue length must align with:

- Client timeout
- Load balancer timeout
- API gateway timeout
- Database timeout
- Downstream timeout

A request may time out at the caller while the server continues processing it.

---

## Concurrency vs Throughput

Thread count controls allowed concurrency, not guaranteed throughput.

```text
More threads
  ↓
More concurrent work
  ↓
More resource contention
  ↓
More waiting and context switching
  ↓
Higher latency
  ↓
Possibly lower throughput
```

The useful concurrency limit is normally determined by the slowest constrained resource.

---

## Practical Production Approach

Measure the following per endpoint:

- QPS
- p50, p95, and p99 latency
- CPU utilization
- CPU time vs waiting time
- Active Tomcat threads
- Tomcat queue depth
- Database pool usage
- HTTP or Redis pool usage
- Timeout rate
- Rejection rate
- Memory and GC pressure

Then apply:

- Timeouts
- Bounded queues
- Bulkheads
- Separate executors
- Rate limiting
- Circuit breakers
- Separate services where strong isolation is required

---

## Mental Model

```text
Tomcat threads
→ Front-door concurrency for HTTP requests

Application executors
→ Background, parallel, or isolated business work

CPU cores
→ Limit true CPU execution in parallel

DB / Redis / HTTP connection pools
→ Limit useful downstream concurrency

Queues
→ Absorb short bursts but increase latency

Bulkheads
→ Prevent one endpoint or dependency from exhausting the whole service
```

---

## Key Takeaways

- Tomcat threads and application threads are both JVM threads.
- Tomcat owns request-processing threads.
- The application owns executors used for background or asynchronous work.
- A synchronous request normally occupies one Tomcat thread for its entire lifetime.
- Tomcat does not normally provide separate request pools by endpoint or HTTP method.
- Use executors, semaphores, bulkheads, or separate services for endpoint isolation.
- CPU-bound pools should stay close to the number of CPU cores.
- I/O-bound pools can be larger because threads spend time waiting.
- Tomcat sizing depends on CPU, memory, latency, queues, database connections, downstream capacity, and timeout budgets.
- More threads increase concurrency only until another resource becomes the bottleneck.

---

## Suggested Next Discussion

Continue with a concrete Spring Boot example involving:

- Tomcat `max-threads`
- HikariCP database pool size
- One slow endpoint and one fast endpoint
- What happens under 1,000 concurrent requests
- How bulkheads and async executors change the behavior
- How virtual threads affect this model
