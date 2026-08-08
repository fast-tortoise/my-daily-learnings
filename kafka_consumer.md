# Kafka Consumer – Fetching, Polling, Timeouts & Offsets

## 1. Consumer Flow

Kafka consumer works roughly like this:

```text
Kafka Broker
    ↓
FetchRequest
    ↓
Consumer internal buffer
    ↓
poll()
    ↓
Your application
```

Important:

```text
fetch != poll
```

Kafka may fetch much more data from the broker than it returns to your application in one `poll()`.

---

# 2. `fetch.min.bytes`

Controls the **minimum amount of data** the broker should try to accumulate before returning a fetch response.

```properties
fetch.min.bytes=1
```

Default is very small.

Low value:

```text
less waiting
↓
lower latency
↓
more fetch requests
```

Higher value:

```text
wait for more data
↓
larger fetch responses
↓
fewer requests
↓
better throughput
```

Example:

```properties
fetch.min.bytes=1048576   # 1 MB
```

Conceptually:

```text
Consumer asks broker for data
        ↓
Is ~1 MB available?
        |
       yes
        ↓
respond
```

If enough data is not available, `fetch.max.wait.ms` limits how long the broker waits.

---

# 3. `fetch.max.wait.ms`

Works together with `fetch.min.bytes`.

Example:

```properties
fetch.min.bytes=1048576
fetch.max.wait.ms=500
```

Meaning:

```text
Try to accumulate 1 MB
        ↓
if 1 MB becomes available
        ↓
respond immediately

otherwise
        ↓
wait at most 500 ms
        ↓
respond with whatever is available
```

Trade-off:

```text
higher fetch.min.bytes
+
higher fetch.max.wait.ms

→ better throughput
→ fewer requests
→ potentially higher latency
```

For latency-sensitive consumers, keep these relatively low.

---

# 4. `max.partition.fetch.bytes`

Controls how much data the broker normally returns **per partition** in a fetch.

Example:

```properties
max.partition.fetch.bytes=1048576   # ~1 MB
```

If the consumer owns:

```text
Partition 0
Partition 1
Partition 2
```

you can think of it roughly as:

```text
P0 → up to ~1 MB
P1 → up to ~1 MB
P2 → up to ~1 MB
```

This is important for consumer memory.

If one consumer owns many partitions:

```text
20 partitions
×
1 MB per partition
```

the consumer can potentially have significant fetched data buffered.

Important exception:

Kafka can still return the first oversized record batch so that the consumer does not get permanently stuck on a record larger than this configured value.

---

# 5. `fetch.max.bytes`

Controls the approximate maximum amount of data returned in an entire fetch request.

Example:

```properties
fetch.max.bytes=52428800   # 50 MB
```

Relationship:

```text
max.partition.fetch.bytes
→ limit per partition

fetch.max.bytes
→ limit across the whole fetch
```

Example:

```text
FetchResponse

P0 → 4 MB
P1 → 6 MB
P2 → 3 MB
P3 → 5 MB

Total = 18 MB
```

`fetch.max.bytes` limits the overall response.

Like `max.partition.fetch.bytes`, this is not an absolute hard limit for the first oversized batch because Kafka must allow the consumer to make progress.

---

# 6. `max.poll.records`

Controls how many records your application receives from one `poll()`.

Example:

```properties
max.poll.records=500
```

Important:

```text
max.poll.records
DOES NOT control how much Kafka fetches from broker.
```

Kafka might do:

```text
Broker
  ↓
fetch 10 MB
  ↓
Consumer internal buffer

[2000 records]

  ↓ poll()

500 records

  ↓ poll()

next 500

  ↓ poll()

next 500
```

So:

```text
fetch settings
→ network + consumer buffering

max.poll.records
→ application processing batch size
```

This distinction is very important.

---

# 7. Is higher `max.poll.records` better?

Not automatically.

Higher value:

```text
more records per poll
↓
better batch-processing efficiency
↓
higher throughput
```

But:

```text
more records
↓
more processing time
↓
higher memory usage
↓
greater risk of exceeding max.poll.interval.ms
```

Example:

```properties
max.poll.records=500
```

If every message takes:

```text
100 ms
```

then processing 500 sequentially takes roughly:

```text
500 × 100 ms
= 50 seconds
```

That may be fine.

But if every message takes:

```text
1 second
```

then:

```text
500 records
≈ 500 seconds
```

which can become a consumer-group problem.

---

# 8. `max.poll.interval.ms`

Controls the maximum allowed time between calls to `poll()`.

Example:

```properties
max.poll.interval.ms=300000
```

≈ 5 minutes.

Conceptually:

```text
poll()
  ↓
application processes records
  ↓
must call poll() again before
max.poll.interval.ms
```

If processing takes too long:

```text
poll()
 ↓
processing...
 ↓
processing...
 ↓
max.poll.interval.ms exceeded
 ↓
Kafka assumes consumer is unhealthy/stuck
 ↓
consumer-group rebalance
```

This setting is therefore closely related to:

```text
max.poll.records
×
processing time per record
```

For slow processing:

* reduce `max.poll.records`
* increase processing parallelism carefully
* increase `max.poll.interval.ms` if long processing is expected

---

# 9. `session.timeout.ms`

Controls how long Kafka waits without receiving consumer liveness/heartbeats before considering the consumer dead.

Conceptually:

```text
Consumer
   ↓ heartbeat
Broker

Consumer crashes
   ↓
no heartbeat
   ↓
session.timeout.ms expires
   ↓
consumer declared dead
   ↓
partitions reassigned
```

This is primarily about:

```text
Is the consumer process alive?
```

---

# 10. `heartbeat.interval.ms`

Controls how frequently the consumer sends heartbeats under the classic consumer-group protocol.

Conceptually:

```text
heartbeat
   ↓
wait
   ↓
heartbeat
   ↓
wait
   ↓
heartbeat
```

It must be lower than `session.timeout.ms`.

Mental model:

```text
heartbeat.interval.ms
→ how frequently I say "I'm alive"

session.timeout.ms
→ how long broker waits before declaring me dead
```

Note: with Kafka's newer consumer group protocol, heartbeat timing can be broker-controlled instead of using this client property.

---

# 11. `max.poll.interval.ms` vs `session.timeout.ms`

Very important interview distinction.

```text
session.timeout.ms
→ detects DEAD consumer

max.poll.interval.ms
→ detects STUCK/SLOW application
```

Example:

Consumer JVM is running and sending heartbeats:

```text
heartbeat ✅
heartbeat ✅
heartbeat ✅
```

but your application gets stuck processing records and doesn't call `poll()`.

Then:

```text
session.timeout.ms
→ may be fine

max.poll.interval.ms
→ exceeded
→ rebalance
```

Think:

```text
session timeout = process liveness

poll timeout = application processing liveness
```

---

# 12. `enable.auto.commit`

Controls whether Kafka automatically commits consumed offsets.

```properties
enable.auto.commit=true
```

means Kafka periodically commits the consumer's position automatically.

For critical processing pipelines, it is common to prefer explicit/manual offset handling.

Why?

Imagine:

```text
read message
    ↓
offset committed
    ↓
application processing fails
```

The message may not be processed successfully even though the offset has advanced.

So critical processing often follows:

```text
read
 ↓
process successfully
 ↓
commit offset
```

instead of:

```text
read
 ↓
automatic commit
 ↓
process
```

---

# 13. `auto.offset.reset`

Controls where a consumer starts when **there is no valid committed offset**.

Common options:

```properties
auto.offset.reset=earliest
```

Start from the oldest available retained record.

```properties
auto.offset.reset=latest
```

Start from newly arriving records.

Conceptually:

```text
Consumer group has committed offset?
        |
       yes
        ↓
continue from committed offset

       no
        ↓
auto.offset.reset
```

Important:

`latest` does NOT mean:

```text
always consume latest message
```

It matters primarily when the consumer has no valid committed offset.

---

# 14. `isolation.level`

Relevant when Kafka transactions are being used.

Options:

```properties
isolation.level=read_uncommitted
```

Consumer can see transactional records even before transaction outcome filtering.

Or:

```properties
isolation.level=read_committed
```

Consumer only returns transactional records from committed transactions.

For transactional pipelines:

```text
Producer transaction

record A
record B
record C

COMMIT
```

With:

```properties
isolation.level=read_committed
```

the consumer sees the committed transaction.

Aborted transactional records are not returned to the application.

---

# 15. Consumer Size Configuration Mental Model

```text
                Kafka Broker

                     ↓

               FetchRequest

                     ↓

        fetch.min.bytes
        fetch.max.wait.ms

                     ↓

        ┌─────────────────────┐
        │ Fetch Response      │
        │                     │
        │ P0 → data           │
        │ P1 → data           │
        │ P2 → data           │
        └─────────────────────┘

        ↑                 ↑
max.partition.       fetch.max.bytes
fetch.bytes          overall fetch
per partition

                     ↓

          Consumer Internal Buffer

                     ↓

                   poll()

                     ↓

             max.poll.records

                     ↓

             Your Application
```

---

# 16. Most Important Consumer Properties

For interviews, remember these first:

```text
fetch.min.bytes
fetch.max.wait.ms
fetch.max.bytes
max.partition.fetch.bytes
max.poll.records
max.poll.interval.ms
session.timeout.ms
heartbeat.interval.ms
enable.auto.commit
auto.offset.reset
```

---

# 17. Throughput vs Latency

## Latency-sensitive consumer

Prefer roughly:

```text
low fetch.min.bytes
low fetch.max.wait.ms
reasonable max.poll.records
```

Effect:

```text
data available
↓
return quickly
↓
process quickly
```

---

## High-throughput consumer

Can use:

```text
higher fetch.min.bytes
larger fetch sizes
larger max.poll.records
```

Effect:

```text
larger network responses
↓
more records processed together
↓
less request overhead
↓
higher throughput
```

Trade-off:

```text
higher throughput
vs
latency + memory + processing time
```

---

# 18. Burst Traffic

During a burst:

```text
Broker has lots of records
        ↓
consumer fetches larger responses
        ↓
internal consumer buffer fills
        ↓
poll() returns max.poll.records
        ↓
application processes records
```

The important tuning relationship becomes:

```text
Kafka fetch speed
        >
application processing speed
```

If this continues:

```text
consumer internal buffering
+
consumer lag
+
memory pressure
```

can increase.

So don't blindly increase:

```text
fetch.max.bytes
max.partition.fetch.bytes
max.poll.records
```

The application must actually be able to process the increased load.

---

# 19. Practical Spring Boot Example

```properties
# Consumer identity
spring.kafka.consumer.group-id=payment-consumer

# Fetching
spring.kafka.consumer.properties.fetch.min.bytes=1
spring.kafka.consumer.properties.fetch.max.wait.ms=500
spring.kafka.consumer.properties.fetch.max.bytes=52428800
spring.kafka.consumer.properties.max.partition.fetch.bytes=1048576

# Application processing batch
spring.kafka.consumer.max-poll-records=500

# Poll processing timeout
spring.kafka.consumer.properties.max.poll.interval.ms=300000

# Offset handling
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.consumer.auto-offset-reset=latest
```

The exact values should be load-tested rather than treated as universal production defaults.

---

# Quick Cheat Sheet

```text
fetch.min.bytes
→ minimum data broker tries to return

fetch.max.wait.ms
→ maximum wait for fetch.min.bytes

max.partition.fetch.bytes
→ data fetched per partition

fetch.max.bytes
→ data fetched across entire request

max.poll.records
→ records returned to application per poll

max.poll.interval.ms
→ maximum time application can take before polling again

heartbeat.interval.ms
→ frequency of consumer heartbeats

session.timeout.ms
→ how long before consumer is considered dead

enable.auto.commit
→ automatic vs application-controlled offset commits

auto.offset.reset
→ where to start when no valid committed offset exists

isolation.level
→ whether transactional consumers see only committed records
```

## One-line Mental Model

```text
FETCH controls broker → consumer data transfer.

POLL controls consumer → application data transfer.

HEARTBEAT/SESSION controls consumer liveness.

MAX.POLL.INTERVAL controls processing liveness.

OFFSET settings control where consumption resumes.
```
