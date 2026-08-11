# Apache Fluss — Additional Notes

## 1. Fluss Core Idea

Fluss can provide both:

- Kafka-like streaming/changelog data
- Latest state lookup using a primary key

Example:

Changes:

product 100 → inventory 50
product 100 → inventory 48
product 100 → inventory 46

Streaming view:

50 → 48 → 46

Latest-state view:

product 100 → 46

So conceptually:

Fluss
=
streaming log
+
current primary-key state

---

## 2. Flink Still Handles Computation

Fluss does not replace Flink for processing logic.

Flink is still used for:

- Aggregations
- Windows
- Joins
- Deduplication
- Transformations
- Fraud rules
- Stateful computation

Example:

Fluss
→ Flink
→ calculate sales per product for last 5 minutes

So:

Fluss
=
storage

Flink
=
computation

---

## 3. Why Tiering Exists

Fluss is designed mainly for hot/recent streaming data.

Keeping years of historical data in fast streaming storage is unnecessary and expensive.

So data can be split into:

HOT DATA
→ Fluss

COLD / HISTORICAL DATA
→ Iceberg
→ Blob / S3

Example:

Last 1 day
→ Fluss

Older data
→ Iceberg
→ Azure Blob

The process of moving/syncing older data from Fluss into the lakehouse is called:

Tiering

So:

Fluss
→ Tiering
→ Iceberg
→ Blob

---

## 4. Tiering vs Normal Flink Processing

There can be two different Flink jobs involved.

### Business Flink Job

Used for application computation.

Example:

Fluss
→ Flink

Tasks:

- Aggregate sales
- Join order + inventory
- Detect fraud
- Calculate rolling metrics

---

### Fluss Tiering Job

Used internally for storage lifecycle management.

Fluss
→ Tiering Service
→ Iceberg

Its job is:

- Copy older Fluss data into the lakehouse
- Keep Fluss and Iceberg synchronized
- Handle retries/consistency
- Manage the hot → cold transition

The current Fluss tiering service itself can run as a Flink job, but this is infrastructure work, not business computation.

---

## 5. Traditional vs Fluss Architecture

### Traditional

Kafka
→ Flink
→ Redis / DB

and separately:

Kafka
→ Flink
→ Iceberg
→ Blob

Flink/application pipelines may need to maintain multiple outputs.

---

### With Fluss

Application
→ Fluss

Fluss provides:

- Streaming data
- Latest PK state
- Hot storage

Then:

Fluss
→ Tiering
→ Iceberg
→ Blob

And separately:

Fluss
→ Flink
→ business computation

This reduces custom plumbing needed to keep streaming storage and historical storage synchronized.

---

## 6. Hot vs Cold Storage

Think of it like:

RAM
→ SSD
→ HDD/archive

Similarly:

Fluss
→ hot/recent/fast data

Iceberg + Blob
→ older/cheap historical data

This is the main purpose of tiering.

---

## 7. Union Read

Physically, one logical table may be split across:

Recent data
→ Fluss

Historical data
→ Iceberg

Example:

2022–Aug 10, 2026
→ Iceberg

Aug 11, 2026
→ Fluss

But logically you still want:

SELECT *
FROM orders;

Union Read allows the system to combine:

Iceberg historical data
+
Fluss recent data
=
one logical dataset

So the caller does not necessarily need to query both systems separately and merge the results manually.

---

## Final Mental Model

Fluss
=
hot streaming storage
+
latest primary-key state

Flink
=
business computation

Tiering
=
move/sync old data from hot storage to cold storage

Iceberg
=
manage historical data as tables

Blob / S3
=
cheap physical storage

Union Read
=
read historical Iceberg data + recent Fluss data as one logical table