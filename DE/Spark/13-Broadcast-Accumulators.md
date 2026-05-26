# Broadcast & Accumulators

## Broadcast Variable Internals

- __Broadcast variable__ - a read-only variable cached on every executor once; eliminates repeated per-task serialization of large shared data (lookup tables, ML models, configuration maps)
- Without broadcast - a closure capturing a large object serializes it with EVERY task; $1000$ tasks × $100 MB$ object = $100 GB$ of serialization traffic
- With broadcast - object serialized ONCE; distributed to executors; each executor caches one copy; all tasks on that executor share it

### Broadcast Creation

- Python -
```python
    lookup = {"US": "United States", "GB": "Great Britain"}
    bc = spark.sparkContext.broadcast(lookup)

    # Use inside task
    result = rdd.map(lambda x: bc.value.get(x, "Unknown"))

    # DataFrame UDF usage
    from pyspark.sql.functions import udf
    lookup_udf = udf(lambda x: bc.value.get(x, "Unknown"))
    df.withColumn("country", lookup_udf(col("country_code")))
```

- Scala -
```scala
    val lookup = Map("US" -> "United States", "GB" -> "Great Britain")
    val bc = spark.sparkContext.broadcast(lookup)

    // Use inside task
    val result = rdd.map(x => bc.value.getOrElse(x, "Unknown"))

    // Destroy when done
    bc.destroy()
```

### Broadcast in SQL / DataFrame Engine

- `BroadcastHashJoin` - the join small side is automatically broadcast by the engine
- `BroadcastExchangeExec` - the physical operator that collects data to Driver and broadcasts
- `HashedRelation` - the in-memory hash map built from broadcast data; specialized for join probe
- User-created broadcasts via `sc.broadcast()` are different from engine-managed join broadcasts - different lifecycle and storage path

### Broadcast Size Limits

- `spark.broadcast.blockSize` (default $4 MB$) - chunk size for `TorrentBroadcast` distribution
- Practical limit - broadcast data must fit in executor memory (storage memory pool)
- No hard size limit in Spark code; executor OOM is the natural limit
- For very large broadcasts ($>$ a few GB) - consider if a join or lookup table approach via shuffle is more appropriate

---

## TorrentBroadcast

- __`TorrentBroadcast`__ - Spark's default and only production broadcast implementation; uses a BitTorrent-style peer-to-peer distribution protocol instead of Driver-to-all-executors point-to-multicast
- Addresses the Driver bandwidth bottleneck - with $1000$ executors, Driver-to-all would require $1000×$ serial transfers; TorrentBroadcast distributes the load

### Why Not Direct Driver Broadcast

- Driver has limited network bandwidth
- $N$ executors requesting the same large object simultaneously → Driver becomes bottleneck
- Single-point-of-failure for broadcast data
- With $100$ executors and $1 GB$ broadcast - Driver must send $100 GB$ total; saturates network

### TorrentBroadcast Protocol

1. __Serialization and chunking__ -
    - Driver serializes the broadcast value using `SparkEnv.serializer`
    - Compressed using `spark.io.compression.codec`
    - Split into chunks of `spark.broadcast.blockSize` (default $4 MB$)
    - Each chunk stored as a `BroadcastBlockId(broadcastId, "piece" + chunkIndex)` in Driver's `BlockManager`

2. __Chunk distribution__ -
    - Driver's `BlockManager` holds all chunks initially
    - First executor to request fetches ALL chunks from Driver
    - Each fetched chunk immediately stored in the executor's `BlockManager`
    - Second executor fetches some chunks from Driver, some from first executor (peer)
    - As more executors join, more peers available; Driver load decreases geometrically
    - Eventually most executors fetch from peers; Driver serves only a fraction of total requests

3. __Reassembly__ -
    - Each executor receives all chunks
    - Deserialized and decompressed into the broadcast value
    - Stored in executor's `BlockManager` as `BroadcastBlockId(broadcastId)` (the full object)
    - Deserialized object cached as a Java object in `MemoryStore` - `MEMORY_AND_DISK` storage level

### Fetch Strategy

- `TorrentBroadcast.readBlocks()` -
    - Queries `BlockManagerMaster` for locations of each chunk
    - Fetches chunks in parallel from available locations (Driver + any executor peers that have them)
    - Uses `BlockManager.getRemoteBytes` for remote chunks; `getLocalBytes` if chunk already local
    - `spark.broadcast.checksum` (default `true`) - validates chunk integrity via checksum

### Storage on Executors

- Broadcast value stored at two levels -
    - __Serialized chunks__ - `BroadcastBlockId("piece0")`, `("piece1")`, etc. in `MemoryStore` or `DiskStore`
    - __Deserialized object__ - the final Java object cached in memory for fast task access
- `bc.value` - returns the deserialized Java object; if evicted from memory, re-reads and deserializes from chunks
- Chunks stored at `MEMORY_AND_DISK` - if memory pressure, chunks spill to disk; deserialized object re-created on next `bc.value` access

---

## Broadcast Lifecycle & Cleanup

### Creation

1. `SparkContext.broadcast(value)` called on Driver
2. Value serialized + chunked + stored in Driver's `BlockManager`
3. `Broadcast` object returned to user; holds `broadcastId` and reference to value
4. No executor-side action yet - lazy; chunks distributed on first access from a task

### First Access (Lazy Distribution)

1. Task references `bc.value`
2. Executor checks local `BlockManager` - broadcast not found
3. Executor fetches chunks from Driver + peers (TorrentBroadcast protocol)
4. Assembles, deserializes, caches deserialized value
5. Returns value to task

### Subsequent Accesses (Same Executor)

- `bc.value` hits in-memory deserialized cache - zero network I/O
- If deserialized object evicted by LRU - re-read from local chunks (disk) or re-fetch chunks from peers

### Cleanup Methods

- `bc.unpersist()` -
    - Removes broadcast blocks from all EXECUTOR `BlockManager`s
    - Driver copy retained - can be re-broadcast on next access
    - Asynchronous by default; `bc.unpersist(blocking=True)` waits for completion
    - Use when broadcast no longer needed by running tasks but may be needed again

- `bc.destroy()` -
    - Removes broadcast blocks from ALL `BlockManager`s including Driver
    - Broadcast is permanently invalidated - any future access throws exception
    - Call when broadcast will never be used again
    - Memory freed immediately on Driver

- Python -
```python
    bc = sc.broadcast(large_dict)

    # Use in tasks
    result = df.rdd.map(lambda x: bc.value[x])

    # Clean up executor copies (keep driver copy)
    bc.unpersist()

    # OR permanently destroy
    bc.destroy()
```

### Automatic Cleanup

- `ContextCleaner` - background thread on Driver that tracks broadcast references
- When `Broadcast` object becomes unreachable (GC'd on Driver) → `ContextCleaner` schedules cleanup
- Cleanup is async and best-effort - not guaranteed if Driver JVM dies before cleanup runs
- In long-running applications (streaming, notebooks) - always call `bc.destroy()` explicitly; GC-based cleanup may be delayed

### Broadcast in Structured Streaming

- Broadcast variables work in Structured Streaming but with caveats -
    - Broadcast created once; value fixed at creation time
    - Cannot update broadcast value mid-stream
    - Pattern for updateable lookups - re-create broadcast on each micro-batch (expensive but correct)
    - Alternative - use a streaming join or foreachBatch with fresh broadcast per batch

---

## Accumulator Internals

- __Accumulator__ - a variable that tasks can only ADD TO; the Driver reads the final value; provides a way to aggregate information from tasks back to the Driver without returning full result sets
- Classic use cases - counting events, summing metrics, collecting diagnostic information

### Why Accumulators Exist

- Normal variables captured in task closures are serialized per task; updates in tasks are LOCAL; Driver never sees them
- Accumulators provide a WRITE channel from executors to Driver with defined merge semantics
- The only safe way for task code to communicate aggregate information back to the Driver

### AccumulatorV2 Architecture

- `AccumulatorV2[IN, OUT]` - the base class for all accumulators since Spark 2.0
    - `IN` - type of values added by tasks
    - `OUT` - type of the final value read by Driver
- Each executor holds a LOCAL copy of the accumulator
- Tasks call `acc.add(value)` on the local copy
- On task completion, executor sends the local accumulator value to Driver
- Driver merges all received values into the global accumulator using `merge(other)`

### Value Flow

```
Driver creates Accumulator (initialValue)
        ↓ serialized with task
Executor Task 1 → local copy → add(v1), add(v2) → localValue = f(v1, v2)
Executor Task 2 → local copy → add(v3)           → localValue = f(v3)
Executor Task 3 → local copy → add(v4), add(v5)  → localValue = f(v4, v5)
        ↓ task completion; each executor sends localValue to Driver
Driver merges: globalValue = merge(merge(localValue1, localValue2), localValue3)
Driver reads: acc.value → final merged result
```

### Internal Registration

- Every accumulator registered with `AccumulatorContext` on creation -
    - Assigned unique `accumulatorId: Long`
    - Stored in `AccumulatorContext.originals: Map[Long, WeakReference[AccumulatorV2[_, _]]]`
- `TaskContext` holds all accumulators active for a task -
    - `TaskContext.taskMetrics.accumulators()` - list of accumulators for this task
- `TaskRunner` serializes accumulator updates into `TaskResult` and sends to Driver
- Driver's `DAGScheduler.updateAccumulators(taskResult)` merges updates into originals

---

## AccumulatorV2

- __`AccumulatorV2[IN, OUT]`__ - abstract base class; must implement 6 methods

### Required Methods

- `isZero: Boolean` - true if the accumulator is in its initial state; zero/empty
- `copy(): AccumulatorV2[IN, OUT]` - creates a fresh copy for a new task; each task starts with a copy of the initial state
- `reset(): Unit` - resets to zero state; used when re-running tasks
- `add(v: IN): Unit` - adds a value to the local accumulator; called by task code
- `merge(other: AccumulatorV2[IN, OUT]): Unit` - merges another accumulator's value into this; called on Driver when combining task results
- `value: OUT` - returns the current accumulated value; only meaningful on Driver

### Built-in Accumulator Types

- `LongAccumulator` - `IN=Long`, `OUT=Long`; add long values; `sum`, `count`, `avg` available
```python
    acc = sc.accumulator(0)                     # LongAccumulator
    df.foreach(lambda row: acc.add(1))
    print(acc.value)
```
- `DoubleAccumulator` - `IN=Double`, `OUT=Double`; add double values
- `CollectionAccumulator[T]` - `IN=T`, `OUT=List[T]`; collects values into a list
    - Warning - all collected values sent to Driver; OOM risk for large collections
    - Use only for debugging/sampling; never for production data collection

### SparkContext Accumulator API

- Python -
```python
    # Long accumulator
    counter = sc.accumulator(0)
    error_count = sc.accumulator(0L)

    # Double accumulator
    total_amount = sc.accumulator(0.0)

    # Use in task
    def process(row):
        if row["status"] == "error":
            error_count.add(1)
        total_amount.add(row["amount"])

    df.foreach(process)
    print(f"Errors: {error_count.value}, Total: {total_amount.value}")
```

- Scala -
```scala
    val counter = sc.longAccumulator("EventCounter")
    val totalAmount = sc.doubleAccumulator("TotalAmount")

    df.foreach { row =>
        counter.add(1L)
        totalAmount.add(row.getDouble(0))
    }

    println(s"Count: ${counter.value}, Total: ${totalAmount.value}")
```

### Named Accumulators

- `sc.accumulator(0, "MyCounter")` - named accumulator visible in Spark UI
- Named accumulators appear in Spark UI → Stages → Accumulators section
- Named accumulators show per-task updates and final merged value
- Use names for all accumulators in production - essential for debugging

---

## Custom Accumulators

- Extend `AccumulatorV2[IN, OUT]` for domain-specific aggregation
- Most common use - collecting structured metrics, building histograms, tracking per-category counts

### Custom Accumulator Pattern

- Scala -
```scala
    import org.apache.spark.util.AccumulatorV2

    // Map accumulator - tracks count per category
    class MapAccumulator extends AccumulatorV2[String, Map[String, Long]] {
        private var _map: scala.collection.mutable.Map[String, Long] =
            scala.collection.mutable.Map.empty

        override def isZero: Boolean = _map.isEmpty

        override def copy(): MapAccumulator = {
            val newAcc = new MapAccumulator
            _map.synchronized {
                newAcc._map = _map.clone()
            }
            newAcc
        }

        override def reset(): Unit = _map.clear()

        override def add(category: String): Unit = {
            _map(category) = _map.getOrElse(category, 0L) + 1L
        }

        override def merge(other: AccumulatorV2[String, Map[String, Long]]): Unit = {
            other match {
                case o: MapAccumulator =>
                    o._map.foreach { case (k, v) =>
                        _map(k) = _map.getOrElse(k, 0L) + v
                    }
            }
        }

        override def value: Map[String, Long] = _map.toMap
    }

    // Registration and use
    val categoryCounter = new MapAccumulator
    sc.register(categoryCounter, "CategoryCounts")

    df.foreach { row =>
        categoryCounter.add(row.getString(0))   // add category string
    }

    println(categoryCounter.value)   // Map("A" -> 1000, "B" -> 500, ...)
```

- Python -
```python
    from pyspark.accumulators import AccumulatorParam

    class DictAccumulatorParam(AccumulatorParam):
        def zero(self, initial_value):
            return {}

        def addInPlace(self, d1, d2):
            for k, v in d2.items():
                d1[k] = d1.get(k, 0) + v
            return d1

    category_counter = sc.accumulator({}, DictAccumulatorParam())

    def count_categories(row):
        category_counter.add({row["category"]: 1})

    df.foreach(count_categories)
    print(category_counter.value)
```

### Thread Safety Requirements

- `add()` called concurrently by multiple threads on the same executor (multiple tasks in same executor JVM)
- `copy()` and `merge()` also called from background threads
- All mutable state in custom accumulator must be thread-safe -
    - Use `synchronized` blocks in Scala
    - Use `java.util.concurrent` collections
    - Python accumulators handled by single-threaded Python process - thread safety less critical but GIL protects

### Serialization Requirements

- `AccumulatorV2` must be serializable (Java serialization)
- `copy()` creates a NEW instance for each task - the copy is what gets serialized into task closure
- Driver holds the "original" in `AccumulatorContext` - receives merged updates from tasks
- Accumulator state that is not Java-serializable must be handled carefully in `copy()` and `merge()`

---

## Accumulator Exactly-Once Guarantee

- __The fundamental challenge__ - Spark may re-run tasks (retries, speculation); a naive accumulator implementation would count the same work multiple times
- __Exactly-once semantics__ - Spark provides exactly-once accumulator updates for SUCCESSFUL tasks and job-level aggregation; but with important caveats

### How Exactly-Once Works (in Theory)

- `TaskSetManager` tracks which task attempts have been registered
- When a task succeeds, `DAGScheduler.handleTaskCompletion` calls `updateAccumulators(taskResult)` -
    - Merges accumulator updates from the successful task attempt
    - Marks task as completed; no future attempts will be merged for this task index
- If a task is retried - the previous failed attempt's accumulator updates are NOT merged (failure updates discarded)
- If speculative execution - when the winner is determined, only the winner's updates are merged; loser's updates discarded

### The Reality - At-Least-Once for Actions

- Accumulator updates are merged during the action (job) that computes them
- __Failed tasks__ - accumulator updates from failed task attempts are discarded; only successful attempt's updates merged → correct
- __Speculative tasks__ - winner's updates merged; loser killed before sending updates → correct
- __Retried stages__ - if a stage is re-run (shuffle data loss), all task accumulator updates re-executed → accumulator DOUBLE-COUNTED for re-run tasks
- __Cached RDDs__ - if an RDD is cached and a task reads from cache on retry, accumulator not incremented again → UNDER-COUNTED if previous attempt had incremented

### The Transformation vs Action Distinction

- __Inside an action__ - accumulator updates reliable; each task index counted once
- __Inside a transformation__ - accumulator updates NOT guaranteed; transformations can be re-evaluated multiple times without the engine knowing

- Python -
```python
    counter = sc.accumulator(0)

    # WRONG - transformation; may be evaluated multiple times
    # counter may over-count if partitions recomputed
    rdd = input.map(lambda x: (counter.add(1), x)[1])
    rdd.cache()
    count1 = rdd.count()    # counter incremented here
    count2 = rdd.count()    # counter NOT incremented again (cache hit)
    # But if cache evicted between count1 and count2:
    # count2 recomputes → counter incremented AGAIN → double count

    # CORRECT - accumulator in action; each task counted exactly once
    def process_and_count(x):
        counter.add(1)
        return x

    rdd.foreach(process_and_count)   # action; exactly-once per task index
    print(counter.value)             # correct count
```

### Accumulator Updates in Failed Jobs

- If a job fails (exception, OOM, user cancellation) -
    - Accumulator updates from completed tasks within that job ARE merged before failure
    - Partial accumulator updates visible on Driver even though job failed
    - This is intentional - useful for diagnosing how far a job got before failing

### Accumulator Visibility Guarantee

- Updates are visible on the Driver ONLY after an action completes
- Updates from tasks of a running action are NOT visible until action finishes
- Exception - named accumulators shown in Spark UI during execution (best-effort, may lag)
- Python -
```python
    counter = sc.accumulator(0)
    rdd.foreach(lambda x: counter.add(1))  # action
    # Only AFTER foreach completes:
    print(counter.value)   # reliable
```

### Practical Guidelines

- Use accumulators ONLY in actions (`foreach`, `foreachPartition`); NOT in transformations (`map`, `filter`)
- Do not use accumulators for correctness-critical counting - use `df.count()` or `df.groupBy().count()` instead
- Accumulators are best for -
    - Diagnostic metrics (error counts, warning counts, skipped record counts)
    - Progress monitoring
    - Collecting small samples for debugging
- Never rely on accumulator values for business logic that affects output data

> [!NOTE]
> The exactly-once guarantee is often misunderstood. The precise guarantee is - for a successfully completed action, each successfully completed task contributes its accumulator update exactly once. This breaks down for re-run stages (shuffle data loss) and for accumulators inside transformations. Design accumulators for monitoring, not for correctness.

> [!TIP]
> For production diagnostic accumulators -
> ```python
> # Pattern - accumulate in foreachPartition for efficiency
> # (one add per partition instead of one per row)
> error_counter = sc.accumulator(0)
>
> def process_partition(rows):
>     local_errors = 0
>     for row in rows:
>         try:
>             process(row)
>         except Exception:
>             local_errors += 1
>     error_counter.add(local_errors)   # one add per partition; thread-safe
>
> df.foreachPartition(process_partition)
> print(f"Total errors: {error_counter.value}")
> ```
> `foreachPartition` reduces accumulator `add()` calls from O(rows) to O(partitions); more efficient and reduces contention on the local accumulator copy.
