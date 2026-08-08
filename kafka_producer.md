# Kafka Producer – Message Size, Batching & Compression

## 1. `batch.size`

* Controls the target batch size for records going to the **same partition**.
* A batch cannot contain records from multiple partitions.
* It is **not a hard maximum message size**.

Example:

```properties
batch.size=16384        # 16 KB
max.request.size=1048576 # 1 MB
```

If a single message is 100 KB, it can still be sent because:

```text
100 KB message > 16 KB batch.size
but
100 KB message < 1 MB max.request.size
```

Kafka creates a larger batch for that record.

---

## 2. `batch.size` vs `max.request.size`

```text
batch.size
→ size of a batch for ONE partition

max.request.size
→ maximum size of the entire producer request
```

One ProduceRequest can contain batches for multiple partitions:

```text
ProduceRequest
├── Partition 0 batch
├── Partition 1 batch
└── Partition 2 batch
```

---

## 3. `linger.ms`

Controls how long the producer waits to collect more records before sending a batch.

Example:

```properties
batch.size=65536
linger.ms=2
```

Low traffic:

```text
1 message
→ wait briefly
→ send small batch
```

Burst traffic:

```text
many messages arrive quickly
→ batch fills quickly
→ send larger/full batch
```

Kafka does **not** wait until `batch.size` is full.

A useful combination for latency-sensitive systems can be:

```text
moderately large batch.size
+
small linger.ms
```

This gives:

```text
low traffic → low latency
burst traffic → better batching
```

---

## 4. Is larger `batch.size` always better?

No.

Benefits:

* fewer network requests
* better compression
* higher throughput
* lower per-message overhead

Costs:

* more producer memory
* potentially larger network bursts
* little benefit if traffic is low

A very large value such as `1 MB` is more useful for high-throughput pipelines than normal low-volume services.

---

## 5. `message.max.bytes`

Broker-side setting:

```properties
message.max.bytes
```

Controls the largest record batch the broker will accept.

Topic-level equivalent:

```properties
max.message.bytes
```

Why is this needed if the producer has `max.request.size`?

Because:

```text
Producer config
→ what the client is willing to send

Broker config
→ what Kafka is willing to accept
```

The broker cannot trust every producer configuration.

It protects Kafka from accidentally receiving extremely large records that could cause:

* memory pressure
* network pressure
* disk I/O spikes
* replication overhead
* consumer issues

Typical relationship:

```text
Producer
max.request.size
        ↓
Broker
message.max.bytes
        ↓
Topic
max.message.bytes
```

All relevant limits must allow the record/batch.

---

# Compression

Kafka supports:

```text
none
gzip
snappy
lz4
zstd
```

Compression happens on the **record batch**, not individually for every message.

Therefore, fuller batches usually compress better.

---

## LZ4

Characteristics:

```text
Compression speed     → Excellent
Decompression speed   → Excellent
CPU usage             → Low
Compression ratio     → Good
```

Good when:

* latency matters
* CPU is limited
* traffic is bursty
* producer throughput is very high
* network is not the main bottleneck

Typical choice for latency-sensitive systems.

---

## Zstd

Characteristics:

```text
Compression speed     → Good
Decompression speed   → Good
CPU usage             → Moderate
Compression ratio     → Excellent
```

Good when:

* network bandwidth matters
* storage cost matters
* replication traffic is high
* messages compress well
* high-throughput Kafka pipelines

Zstd is often a strong general-purpose production choice.

---

## Gzip

Characteristics:

```text
Compression ratio → Very good
CPU usage         → High
Compression speed → Slow
```

Good when maximum compression is more important than CPU or latency.

Usually less attractive than Zstd for modern high-throughput Kafka workloads.

---

# LZ4 vs Zstd vs Gzip

| Algorithm | CPU    | Speed     | Compression Ratio | Best For                             |
| --------- | ------ | --------- | ----------------- | ------------------------------------ |
| LZ4       | Low    | Very Fast | Good              | Low latency / CPU constrained        |
| Zstd      | Medium | Fast      | Excellent         | High throughput / network efficiency |
| Gzip      | High   | Slow      | Very Good         | Maximum compression                  |
| Snappy    | Low    | Fast      | Moderate          | Older/general workloads              |

---

# Burst Traffic

During burst traffic:

```text
Normal traffic:
few messages
→ small batches

Burst:
many messages arrive quickly
→ batches naturally fill
→ fewer ProduceRequests
→ better compression
→ higher throughput
```

You normally do **not** need to dynamically change `batch.size` during a burst.

### Compression choice during bursts

If CPU is the bottleneck:

```text
LZ4
```

If network/disk bandwidth is the bottleneck:

```text
Zstd
```

Avoid making Gzip the default for latency-sensitive bursty systems because compression can become CPU-heavy.

---

# Practical Mental Model

```text
Application messages
        ↓
same-partition records grouped
        ↓
batch.size
        ↓
wait up to linger.ms
        ↓
compression
        ↓
ProduceRequest
        ↓
max.request.size
        ↓
Kafka Broker
        ↓
message.max.bytes
        ↓
Topic
max.message.bytes
```

For latency-sensitive systems such as payments, a reasonable starting idea is:

```properties
batch.size=65536
linger.ms=1-5
compression.type=lz4

acks=all
enable.idempotence=true
```

For high-throughput event pipelines:

```properties
compression.type=zstd
```

is often worth benchmarking first.
