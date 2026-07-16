# Kafka Broker Sizing — Continuation Notes

## Core Principle

**Broker count is NOT decided by partition count.**

Partitions determine:

- Parallelism
- Ordering units
- Consumer concurrency

Brokers provide:

- CPU
- Disk
- Memory
- Network bandwidth
- Failure tolerance

Final broker count is whichever resource becomes the bottleneck first.

---

# Broker Sizing Formula

Choose brokers based on the maximum requirement among:

- Availability
- Storage
- Producer throughput (Ingress)
- Consumer throughput (Egress)
- Failure recovery capacity
- Partition distribution (rarely the bottleneck)

Conceptually:

```
Required Brokers =
max(
    availability,
    storage,
    ingress,
    egress,
    recovery headroom
)
```

---

# Industry Standard Starting Point

Typical production defaults:

```
Brokers              : 3
Replication Factor   : 3
min.insync.replicas  : 2
acks                 : all
Unclean leader election : disabled
```

Reason:

- tolerate one broker failure
- rolling upgrades
- safe writes

---

# 24 Partition Example

Topic

```
24 partitions
RF = 3
```

Total replicas

```
24 × 3 = 72 replicas
```

Replica distribution

| Brokers | Leaders/Broker | Replicas/Broker |
|----------|----------------|-----------------|
|3|8|24|
|4|6|18|
|6|4|12|
|12|2|6|

Observation:

24 partitions **do not** require 24 brokers.

Usually:

- 3 brokers → normal production
- 6 brokers → higher throughput / better failure handling
- 12+ brokers → storage or throughput driven

---

# Factor 1 — Availability

Minimum brokers depend on replication factor.

Example

```
RF = 3

Need at least 3 brokers.
```

Otherwise Kafka cannot place replicas on different brokers.

---

# Factor 2 — Producer Ingress

Ingress means:

**Data entering Kafka from producers.**

Example

```
1000 messages/sec

100 KB each

Ingress

=100 MB/sec
```

Ingress affects

- Broker network receive
- Disk writes
- Replication traffic
- CPU
- Compression
- Page cache

---

# Factor 3 — Consumer Egress

Egress means:

**Data leaving Kafka to consumers.**

Suppose

```
Producer writes

100 MB/sec
```

Consumer Groups

```
Group A
Group B
Group C
```

Each group reads the entire topic.

Then

```
Kafka sends

100
+100
+100

=300 MB/sec
```

Important distinction

Consumers **inside the same consumer group**

```
24 partitions

24 consumers

Still only

100 MB/sec
```

because each message is consumed once per group.

Different consumer groups each receive their own copy.

---

# Ingress vs Egress

Ingress

```
Producer

------>

Kafka
```

Egress

```
Kafka

------>

Consumers
```

Example

```
Ingress

200 MB/sec
```

Consumer Groups

```
Notification

Analytics

Fraud

Search

ML

History

Backup

Recommendation
```

Eight consumer groups

```
Egress

8 × 200

=1600 MB/sec
```

Even though producers only write

```
200 MB/sec
```

Kafka sends

```
1.6 GB/sec
```

Broker network is often limited by **consumer egress**, not producer ingress.

---

# Factor 4 — Storage

Storage calculation

```
Incoming Data

×

Retention

×

Replication Factor

+

20–30% headroom
```

If storage requirement exceeds broker disks,

Increase:

- broker count
- disk size

Storage often determines broker count before partitions do.

---

# Factor 5 — Failure Capacity

Cluster should survive one broker failure without overload.

Example

```
3 brokers

Lose 1

Remaining 2 brokers handle all traffic.
```

Normal utilization should leave headroom.

Typical operational target:

```
Normal CPU

≈50–60%
```

so remaining brokers can absorb traffic during failures.

---

# Factor 6 — Hot Partitions

Broker count cannot fix a hot partition.

Example

```
Partition 0

100 MB/sec

Remaining 23 partitions

20 MB/sec
```

One broker still owns the leader.

Solutions

- Better partition key
- More partitions
- Key salting (if ordering permits)
- Stronger broker

---

# Practical Rules

Small production

```
24 partitions

RF = 3

Low traffic

=> 3 brokers
```

Medium production

```
Higher throughput

Several consumer groups

More storage

=> 3–6 brokers
```

Large production

```
Hundreds MB/sec+

TBs of retention

Many consumers

=> 6–12+ brokers
```

---

# Interview Answer

Do not size Kafka brokers from partition count.

Instead discuss:

1. Replication factor
2. Producer ingress
3. Consumer egress
4. Storage and retention
5. Failure tolerance
6. Future growth

Then choose broker count based on whichever resource becomes the bottleneck first.

---

# Key Takeaways

- Partition count ≠ broker count.
- 24 partitions generally start with **3 brokers**, not 24.
- Ingress = producer write rate into Kafka.
- Egress = total data Kafka sends to all consumer groups.
- Multiple consumers **within the same group** do **not** increase egress.
- Multiple **consumer groups** multiply egress.
- Broker sizing is primarily driven by:
    - Throughput
    - Storage
    - Failure headroom
    - Network bandwidth
- Storage and consumer egress are often the real reasons clusters grow beyond three brokers.