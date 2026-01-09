
# Trigger Modes in Apache Spark Structured Streaming

This section explains how different trigger modes behave with respect to **checkpointing**, **offset handling**, and **startingOffsets**, based on practical implementation experience.


| Trigger Mode       | Checkpoint Location         | Offset Read Behavior                               | Restart Behavior                       | Typical Use Case                             |
| ------------------ | --------------------------- | -------------------------------------------------- | -------------------------------------- | -------------------------------------------- |
| **Once**           | ❌ Not specified (temporary) | Reads from `startingOffsets` every run             | Treated as first run on every restart  | One-time exploratory runs, testing           |
| **Once**           | ✅ Specified                 | Reads only unprocessed offsets from checkpoint     | Resumes from last committed offset     | Backfills, batch-on-stream jobs              |
| **AvailableNow**   | ❌ Not specified (temporary) | Reads from `startingOffsets` every run             | Full reprocessing on restart           | Ad-hoc batch execution                       |
| **AvailableNow**   | ✅ Specified                 | Reads all available new data since last checkpoint | Safe restart, no duplicate reads       | Large backfills, incremental batch pipelines |
| **ProcessingTime** | ❌ Not specified (temporary) | Tracks offsets only during runtime                 | Offsets lost on restart → reprocessing | Local testing, debugging                     |
| **ProcessingTime** | ✅ Specified                 | Incremental read of new offsets per batch          | Safe resume from last offset           | Long-running real-time pipelines             |
| **Continuous**     | ❌ / ⚠️ Limited              | Internal offset tracking (not micro-batch based)   | Possible duplicate processing          | Low-latency experiments                      |
| **Continuous**     | ⚠️ Limited support          | Limited recovery guarantees                        | No strong exactly-once semantics       | Rare / non-production use                    |


---

## 1. `trigger = Once` / `trigger = AvailableNow`

> `AvailableNow` is the **preferred and optimized replacement** for `Once` in newer Spark versions.
> Both process **all available data and then stop**.

---

### 1.1 Without `checkpointLocation`

* If `checkpointLocation` is **not explicitly specified**, Spark creates a **temporary (auto-generated) checkpoint location** internally.
* This temporary checkpoint:

  * Is different for every run
  * Is not durable
  * Is lost when the job finishes or restarts
* Because of this:

  * Every new run is treated as a **fresh run**
  * Spark re-reads data based on `startingOffsets`
  * This behaves like a **full load on every execution**

> Effectively, this is similar to running a batch job repeatedly on the same source.

---

### 1.2 With `checkpointLocation`

* Spark reads the **last committed offsets** from the checkpoint directory.
* Only **new/unprocessed data** is read from the source.
* Data is processed incrementally and typically **appended** to the sink (sink-dependent).
* This enables **restart-safe and idempotent execution**.

---

### 1.3 `startingOffsets` behavior

* `earliest`

  * Reads data from the beginning **only on the first run**
* `latest`

  * Reads only data that arrives **after the query starts**
  * For `Once / AvailableNow`, if no new data arrives during execution, the result may be **empty (null output)**

> `startingOffsets` is used **only when no valid checkpoint exists**.

---

## 2. `trigger = ProcessingTime`

This trigger runs the query **continuously** at a fixed time interval.

---

### 2.1 Processing behavior

* Spark triggers a micro-batch every configured interval
  (e.g., every 10 seconds).
* Each batch processes **all unprocessed offsets** since the previous batch.
* If a batch takes longer than the interval, the next batch waits (no overlap).

---

### 2.2 Without `checkpointLocation`

* Spark creates a **temporary internal checkpoint**.
* Offsets are tracked **only for the lifetime of the running query**.
* On application restart:

  * Offsets are lost
  * Spark reprocesses data again from `startingOffsets`
* This can result in **duplicate processing**.

---

### 2.3 With `checkpointLocation`

* Spark stores offsets durably in the checkpoint directory.
* On restart, Spark resumes from the **last committed offset**.
* This is the **recommended and production-safe configuration**.

---

### 2.4 `startingOffsets` behavior

* `earliest`

  * First micro-batch (batch 0) reads from the beginning
* `latest`

  * Batch 0 is created with the current latest offset (no data)
  * Subsequent batches read only newly arriving data

> Once a checkpoint exists, `startingOffsets` is ignored.

---

## 3. `trigger = Continuous`

> Continuous mode is **rarely used in production**.

---

### Characteristics

* Data is processed **continuously (record-at-a-time)** instead of micro-batches.
* The configured interval (e.g., `Continuous("10 seconds")`) controls:

  * Progress reporting
  * Epoch markers
  * **Not traditional micro-batch checkpoints**
* Offset tracking and recovery guarantees are **limited**.

---

### Failure considerations

* If the job fails before progress is durably recorded:

  * Some data may be **reprocessed**
  * Exactly-once guarantees are **not fully supported**
* Due to limited operator, source, and sink support, this mode is generally avoided.

---

## Key Takeaways

* **Triggers control *when* Spark processes data**
* **Checkpoints control *what has already been processed***
* **`startingOffsets` applies only on the first run without a checkpoint**
* Spark may work without an explicit `checkpointLocation`, but this relies on **temporary checkpoints and is not safe for production**

