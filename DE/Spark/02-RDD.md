# RDD

- __RDD (Resilient Distributed Dataset)__ -
    - The core distributed data abstraction in Spark
    - An immutable, partitioned collection of records distributed across executor JVMs
- __Immutable__ -
    - Once created, an RDD cannot be modified
    - Every transformation produces a new RDD
    - This is what makes lineage-based fault tolerance possible
- __Lazy__ -
    - Transformations record intent in the DAG
    - Nothing materializes until an action forces execution
- __Typed__ -
    - `RDD[T]` carries the element type at compile time (Scala/Java)
    - Python erases this at the JVM boundary
- __Schema-less__ -
    - Unlike DataFrames, RDDs have no column metadata, no Catalyst optimization, no predicate pushdown
    - You own the data layout entirely
- __Low-level API__ -
    - Gives direct control over partitioning, serialization, locality
    - Used when DataFrame/Dataset abstractions are insufficient (custom binary formats, iterative ML algorithms, graph processing)

> [!NOTE]
> `DataFrame` is `Dataset[Row]` - a Dataset where the type parameter is an untyped `Row`.
>
> Under the hood, a DataFrame/Dataset compiles down to an `RDD` of `InternalRow` for execution. The optimizer operates above RDD level - by the time tasks run, it's RDD mechanics all the way down.

- When to use RDDs -
    - Custom partitioners that require business logic unavailable in DataFrame API
    - Processing non-tabular data - binary records, variable-length blobs, custom serialization formats
    - Iterative algorithms where you cache intermediate results and explicitly control recomputation
    - Interfacing with legacy Hadoop `InputFormat`/`OutputFormat` directly
    - Fine-grained control over data locality or task-level side effects

## RDD Creation

- __From collection__ - `sc.parallelize(seq, numSlices)` - partition count defaults to `sc.defaultParallelism`
- __From file__ - `sc.textFile(path, minPartitions)` - one partition per HDFS block by default
- __From Hadoop format__ - `sc.hadoopFile`, `sc.newAPIHadoopFile` - any `InputFormat` supported
- __From another RDD__ -
    - Transformation returns a new RDD
    - Child holds reference to parent in lineage
- __From DataFrame__ - `df.rdd` -
    - Converts to `RDD[Row]`
    - Triggers schema erasure, loses Tungsten optimizations

- Python -
    ```
    sc = spark.sparkContext

    rdd = sc.parallelize([1, 2, 3, 4], numSlices=4)
    rdd = sc.textFile("hdfs://path/data.txt", minPartitions=10)
    rdd = sc.sequenceFile("hdfs://path/", keyClass="...", valueClass="...")
    rdd = df.rdd                                                            # DataFrame → RDD[Row]
    ```

- Scala -
    ```
    val rdd = sc.parallelize(Seq(1,2,3,4), numSlices = 4)
    val rdd = sc.textFile("hdfs://path/data.txt", minPartitions = 10)
    val rdd = df.rdd                                                        // RDD[Row]
    ```

## RDD Partitions

- __Partition__ -
    - The atomic unit of parallelism
    - One task processes exactly one partition
    - A contiguous slice of the RDD's data living on one executor
- __Partition count = parallelism__ - more partitions → more tasks can run concurrently (bounded by total executor cores)
- Every partition has an index ($0$-based) and a set of preferred locations for data locality

### Partition Count Rules

- `sc.parallelize` -
    - Defaults to `sc.defaultParallelism` (= total executor cores)
    - Override with `numSlices`
- `sc.textFile` -
    - One partition per HDFS block (default $128 MB$)
    - `minPartitions` sets a floor
    - Spark may split further but never merges blocks
- __Shuffle output__ -
    - Controlled by `spark.sql.shuffle.partitions` (SQL/DataFrame) or `rdd.partitionBy(n)` / operation `numPartitions` arg (RDD API)
- `coalesce(n)` -
    - Reduces to `n` without shuffle
    - May produce skewed partition sizes if reduction is large
- `repartition(n)` -
    - Always full shuffle
    - Even distribution guaranteed

### Partition Sizing Guidelines

- Target $~128 MB$ per partition - matches HDFS block size, balances task overhead vs memory pressure
- Partition count ≈ $2–4×$ total executor cores -
    - Ensures all cores stay busy even when some tasks are slow
    - Last wave of tasks uses all slots
- __Too few__ - large partitions → executor OOM, no parallelism, stragglers dominate
- __Too many__ - tiny partitions → scheduler overhead, excessive small shuffle files, high Driver metadata load, GC pressure

### Partitioner

- Determines which partition a key maps to in key-value RDDs
- Only `RDD[(K, V)]` has a `Partitioner`; plain RDDs have `partitioner = None`
- __`HashPartitioner(n)`__ -
    - `partition = hash(key) % n`
    - Default for all key-based operations
    - Requires consistent `key.hashCode()`
- __`RangePartitioner(n, rdd)`__ -
    - Samples the RDD to determine range boundaries
    - Assigns keys to partitions by sorted range
    - Used by `sortByKey`
    - Sampling cost at construction

### Custom Partitioner

- Python -
    ```python
    from pyspark import Partitioner

    class DomainPartitioner(Partitioner):
        def __init__(self, num_parts):
            self.num_parts = num_parts

        def numPartitions(self):
            return self.num_parts

        def getPartition(self, key):
            domain = key.split("@")[-1]
            return hash(domain) % self.num_parts

    rdd.partitionBy(DomainPartitioner(20))
    ```

- Scala -
    ```scala
    class DomainPartitioner(n: Int) extends Partitioner {
        override def numPartitions: Int = n
        override def getPartition(key: Any): Int =
            key.toString.split("@").last.hashCode.abs % n
    }

    rdd.partitionBy(new DomainPartitioner(20))
    ```

### Inspecting Partitions

```python
    rdd.getNumPartitions()                          # partition count
    rdd.glom().collect()                            # collect each partition as a list - debug only, small data
    rdd.mapPartitionsWithIndex(
        lambda idx, it: [(idx, x) for x in it]
    ).collect()                                     # see which element is in which partition
```

### Partition Locality

- `DAGScheduler` computes preferred locations for each task - the set of nodes where that partition's data already resides
- `PROCESS_LOCAL` - data in same JVM (cached RDD partition on this executor)
- `NODE_LOCAL` - data on the same physical node (HDFS block local, or cached on co-located executor)
- `RACK_LOCAL` - data on a node in the same rack
- `ANY` - data anywhere in the cluster
- `TaskScheduler` delay-schedules -
    - Waits briefly for a better locality slot before falling back
    - Controlled by `spark.locality.wait` (default $3s$ per level)

## Transformations vs Actions

- __Transformation__ -
    - Returns a new RDD
    - Lazy - recorded as a DAG node, zero execution
    - Eg - `map`, `filter`, `flatMap`, `groupByKey`, `join`, `distinct`, `repartition`, `sortBy`, `mapPartitions`, `mapPartitionsWithIndex`, `zip`, `cogroup`
- __Action__ -
    - Triggers DAG execution
    - Returns a value to the Driver or writes to storage
    - Eg - `count`, `collect`, `take`, `first`, `reduce`, `fold`, `aggregate`, `foreach`, `foreachPartition`, `saveAsTextFile`, `saveAsObjectFile`, `countByKey`, `lookup`

## RDD Iterator Model & Compute Chain

- __Iterator model__ -
    - The fundamental execution model inside a Spark task
    - Each RDD exposes a `compute(partition, context)` method that returns an `Iterator[T]`
    - Transformations wrap iterators around iterators
    - Data flows element-by-element through the chain without intermediate materialization
- __Pull-based__ -
    - The terminal consumer (shuffle writer or action) pulls elements from the outermost iterator
    - Each `next()` call cascades down the chain through every transformation
    - Only one element at a time in memory per chain link
- __Zero intermediate materialization within a stage__ -
    - `filter → map → flatMap` on the same partition is one iterator chain
    - No intermediate array is built between transformations

### `compute()` Mechanics

- Every concrete RDD subclass overrides `compute`
- `iterator(partition, context)` -
    - The public entry point
    - Checks `BlockManager` first (is this partition cached?)
    - If not, delegates to `compute()`

- Scala -
    ```scala
    // MappedRDD - wraps parent iterator, applies f lazily
    override def compute(split: Partition, context: TaskContext): Iterator[B] =
        parent.iterator(split, context).map(f)

    // FilteredRDD
    override def compute(split: Partition, context: TaskContext): Iterator[T] =
        parent.iterator(split, context).filter(pred)

    // HadoopRDD - actual I/O, produces elements from a RecordReader
    override def compute(split: Partition, context: TaskContext): Iterator[(K, V)] =
        new HadoopIterator(split.asInstanceOf[HadoopPartition], context)
    ```

### Full Iterator Chain Example

- `sc.textFile("hdfs://...")` → HadoopRDD → `Iterator[String]` (reads HDFS blocks)
    - `.filter(_.contains("ERROR"))` → `FilteredRDD` → wraps `HadoopRDD` iterator
    - `.map(line => parse(line))` → `MappedRDD` → wraps `FilteredRDD` iterator
    - `.mapPartitions(iter => aggregate)` → `MapPartitionsRDD` → wraps `MappedRDD` iterator

- When a task runs, it calls `mapPartitionsRDD.iterator(partition, ctx)` -
    - `MapPartitionsRDD.compute` calls `MappedRDD.iterator`
    - `MappedRDD.compute` calls `FilteredRDD.iterator`
    - `FilteredRDD.compute` calls `HadoopRDD.iterator`
    - `HadoopRDD.compute` opens HDFS block, yields first record
    - Record flows up through `filter` → `map` → `mapPartitions` in one pull

### `map` vs `mapPartitions` vs `foreachPartition`

- __`map(f)`__ -
    - Applies `f` element-wise
    - Per-element function call overhead
    - `f` called once per record
- __`mapPartitions(f)`__ -
    - `f` receives the entire partition iterator
    - Amortizes setup cost once per partition, not once per record (DB connections, model loading)
    - GC friendlier for stateful processing
- __`foreach(f)`__ -
    - Applies `f` to each element for side effects
    - Implemented as `mapPartitions` + drain
    - Same pull model
- __`foreachPartition(f)`__ -
    - `f` receives the full iterator
    - Use for partition-level side effects (writing to external systems, flushing buffers)
    - Same amortization benefit as `mapPartitions`

- Python -
    ```python
    # Bad - opens connection per record
    rdd.map(lambda x: write_to_db(connect(), x))

    # Good - opens connection once per partition
    def write_partition(records):
        conn = connect()
        for r in records:
            conn.write(r)
            yield r
        conn.close()

    rdd.mapPartitions(write_partition)
    ```

### Task Execution & Iterator Lifecycle

- Executor thread calls `runTask()` on the task
- Task calls `rdd.iterator(partition, context)` on the final RDD in the stage
- __Shuffle map stage__ -
    - Iterator consumed by `ShuffleWriter`
    - Elements flow through the chain and are written directly to the shuffle file
    - No intermediate collection
- __Result stage__ - iterator consumed by the action (`count()` increments a counter per element, `collect()` builds a local array)
- `TaskContext` -
    - Carries task metadata (attempt ID, partition ID, stage ID)
    - Allows registering completion callbacks
    - Accessible inside any `compute()` call

> [!NOTE]
> The iterator model is why Spark can process a $100 GB$ partition on an executor with $4 GB$ heap - only a handful of records are in-flight through the chain at any moment.
>
> Memory pressure comes from aggregation (building hash tables for `groupBy`) and caching, not from the pipeline itself.

## RDD Lineage & DAG

- __Lineage__ -
    - The complete chain of transformations from source to a given RDD
    - Stored as a directed graph of RDD dependencies
    - Not data, just a recipe
- __DAG__ -
    - The lineage graph is a DAG (Directed Acyclic Graph)
    - Acyclic because transformations always produce new RDDs, never mutate existing ones
- __Why lineage replaces replication__ -
    - Instead of copying partitions to multiple nodes, Spark recomputes any lost partition by replaying its lineage chain from the nearest reliable source (file, checkpoint, or cached RDD)
- __Recomputation cost__ -
    - Replaying a long lineage chain is expensive
    - This is why `persist()` and `checkpoint()` exist - to truncate lineage and avoid full recomputation

### Lineage Graph Structure

- Each RDD holds a list of `Dependency` objects pointing to parent RDDs
- `toDebugString` prints the lineage tree with indentation showing dependency depth

- Python -
    ```python
    rdd = sc.textFile("hdfs://data/") \
        .filter(lambda x: x.startswith("2024")) \
        .map(lambda x: (x.split(",")[0], 1)) \
        .reduceByKey(lambda a, b: a + b)

    print(rdd.toDebugString())
    # (8) PythonRDD[4] at RDD at PythonRDD.scala:53 []
    #  |  MapPartitionsRDD[3] ...
    #  |  PythonRDD[2] ...
    #  |  MapPartitionsRDD[1] ...
    #  |  data.txt MapPartitionsRDD[0] ...
    #      HadoopRDD[...] ...
    ```

### DAGScheduler & Stage Cutting

- `DAGScheduler` walks the RDD lineage bottom-up from the final RDD on which an action was called
- Cuts a stage boundary at every `ShuffleDependency` - each stage is a maximal set of RDDs connected only by `NarrowDependency`
- Within a stage, all transformations pipeline on a single partition - one pass, zero network I/O
- Between stages, a shuffle materializes data to disk and redistributes it across the network

## NarrowDependency vs ShuffleDependency

- __`NarrowDependency`__ -
    - Each child partition depends on a fixed, bounded set of parent partitions (usually exactly one)
    - Data never crosses the network
    - Pipelines within a stage
- __`ShuffleDependency`__ (wide dependency) -
    - Each output partition may depend on all input partitions
    - Requires full network redistribution
    - Forces a stage boundary
    - Shuffle write to disk before the boundary, shuffle read after

### NarrowDependency Subtypes

- __`OneToOneDependency`__ -
    - Child partition `i` maps to exactly parent partition `i`
    - Used by `map`, `filter`, `flatMap`, `mapPartitions`
    - Partition count preserved
- __`RangeDependency`__ -
    - Used by `union`
    - Child partition `i` maps to a contiguous range in one parent RDD
    - Still narrow because mapping is deterministic and local

### ShuffleDependency Internals

- Records the shuffle ID, partitioner, serializer, and map-side aggregator (if any)
- At stage boundary, map tasks write shuffle output files to local disk
- Reduce tasks in the next stage fetch their partition slice from every map task output via HTTP (`BlockManager`)
- Tracked by `MapOutputTracker` - map tasks register output locations, reduce tasks query for them

### Co-partitioning Eliminates Shuffles

- If two RDDs share the same `Partitioner` (same type, same `numPartitions`), a join between them is narrow - matching partitions are already co-located
- Two RDDs are co-partitioned only if both used the same `HashPartitioner` instance with the same `numPartitions`

- Python -
    ```python
    # Both use HashPartitioner(10) - join is narrow, no shuffle
    rdd1 = pairs.partitionBy(10)
    rdd2 = other_pairs.partitionBy(10)
    joined = rdd1.join(rdd2)                        # no shuffle - co-partitioned

    # rdd2 uses default partitioner - join triggers full shuffle
    joined = rdd1.join(other_pairs)                 # shuffle on other_pairs side
    ```

> [!NOTE]
> Cache the co-partitioned RDD after `partitionBy` - otherwise `partitionBy` itself re-shuffles on every action. The shuffle investment amortizes across multiple joins on the same dataset.

### Failure Recovery

- __Narrow dependency failure__ -
    - Only the lost partition is recomputed
    - Only the parent partitions feeding that one child partition are reprocessed
- __Wide dependency failure__ -
    - If a reduce-side partition is lost, all map-side partitions that contributed to it may need to re-run
    - If map-side shuffle output was also lost (no External Shuffle Service), the entire parent stage reruns

## RDD Persistence & Storage Levels

- __Persistence__ - explicitly materializing an RDD's partitions to memory/disk so subsequent actions reuse materialized data instead of replaying the full lineage chain
- __Without persistence__ -
    - Every action on an RDD retriggers its entire lineage from source
    - Catastrophically expensive for iterative algorithms or RDDs used multiple times
- `persist()` and `cache()` mark the RDD -
    - Actual materialization happens on the first action that computes each partition
    - Subsequent reads hit the `BlockManager` instead of recomputing

### `cache()` vs `persist()`

- __`cache()`__ - shorthand for `persist(StorageLevel.MEMORY_ONLY)`
- __`persist(storageLevel)`__ - explicit control over where and how partitions are stored

- Python -
    ```python
    rdd.cache()                                     # MEMORY_ONLY
    rdd.persist()                                   # also MEMORY_ONLY
    rdd.persist(StorageLevel.MEMORY_AND_DISK)
    rdd.persist(StorageLevel.MEMORY_ONLY_SER)
    rdd.persist(StorageLevel.DISK_ONLY)
    rdd.persist(StorageLevel.OFF_HEAP)

    rdd.unpersist()                                 # remove from cache, blocking=False by default
    rdd.unpersist(blocking=True)                    # wait until eviction completes
    ```

| Storage Level         | Memory | Disk | Serialized | Replicated | Notes                                                                                   |
| --------------------- | ------ | ---- | ---------- | ---------- | --------------------------------------------------------------------------------------- |
| `MEMORY_ONLY`         | ✓      | ✗    | ✗          | ✗          | Deserialized JVM objects; fastest access; OOM risk on large RDDs                        |
| `MEMORY_ONLY_SER`     | ✓      | ✗    | ✓          | ✗          | Serialized bytes in memory; smaller footprint; CPU cost to deserialize on read          |
| `MEMORY_AND_DISK`     | ✓      | ✓    | ✗          | ✗          | Spills partitions that don't fit in memory to disk; deserialized                        |
| `MEMORY_AND_DISK_SER` | ✓      | ✓    | ✓          | ✗          | Serialized in memory, serialized on disk; best space efficiency with spill              |
| `DISK_ONLY`           | ✗      | ✓    | ✓          | ✗          | All on disk; avoids OOM; high read latency                                              |
| `DISK_ONLY_2`         | ✗      | ✓    | ✓          | ✓          | Disk with 2× replication across nodes                                                   |
| `MEMORY_ONLY_2`       | ✓      | ✗    | ✗          | ✓          | In-memory with 2× replication; fault tolerant but 2× memory cost                        |
| `OFF_HEAP`            | ✓      | ✗    | ✓          | ✗          | Tungsten off-heap memory; bypasses JVM GC; requires `spark.memory.offHeap.enabled=true` |

> [!TIP]
> Choosing a storage level -
>   - Fast reuse, data fits in memory → `MEMORY_ONLY`
>   - Data fits in memory but GC pressure is high → `MEMORY_ONLY_SER` (Kryo strongly recommended)
>   - Data may not fit in memory → `MEMORY_AND_DISK`
>   - Critical RDD, can afford $2×$ memory → `MEMORY_ONLY_2`
>   - Checkpoint substitute without lineage severance → `DISK_ONLY`

### `BlockManager` & Eviction

- Cached partitions are stored as blocks in the executor's `BlockManager` under `RDDBlockId(rddId, partitionIndex)`
- __`MemoryStore`__ - holds deserialized objects or serialized byte arrays in JVM heap (or off-heap)
- __`DiskStore`__ - writes serialized bytes to local disk under `spark.local.dir`
- __Eviction policy__ - LRU -
    - When `MemoryStore` is full, least recently used blocks evicted to disk (if `MEMORY_AND_DISK`) or dropped (if `MEMORY_ONLY`)
    - Dropped blocks recomputed from lineage on next access
- `BlockManagerMaster` on the Driver tracks which executor holds which block -
    - If an executor dies, Driver knows which cached blocks are lost and recomputes on next access

### Serialization Impact on Persistence

- `MEMORY_ONLY` stores deserialized Java/Python objects -
    - Fast to read, no deserialization cost
    - Large memory footprint (object headers, references, boxing)
- `MEMORY_ONLY_SER` stores a byte array -
    - Often $5–10×$ smaller with Kryo
    - CPU cost on every read
- __Kryo vs Java serialization__ -
    - Kryo is $3–10×$ faster and produces $3–5×$ smaller output
    - Configure via `spark.serializer=org.apache.spark.serializer.KryoSerializer`

- Python -
    ```python
    spark = SparkSession.builder \
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
        .config("spark.kryo.registrationRequired", "false") \
        .getOrCreate()
    ```

### `unpersist()` & Memory Management

- `unpersist()` -
    - Instructs `BlockManagerMaster` to remove all cached blocks for this RDD from all executors
    - Asynchronous by default
- Spark does NOT automatically unpersist RDDs when they go out of scope -
    - Memory held until explicit `unpersist()` or LRU eviction

> [!NOTE]
> In long-running applications (streaming, notebooks), forgetting to `unpersist()` intermediate RDDs is a common cause of progressive memory degradation.

### Checkpoint vs Lineage Truncation

- __`persist()`__ -
    - Caches the RDD in memory/disk but does NOT truncate lineage
    - If the cached partition is lost, Spark still replays lineage to recompute it
- __`checkpoint()`__ -
    - Materializes RDD to a reliable store (HDFS) AND severs the lineage
    - The checkpointed RDD has no parents
    - Recomputation starts from the checkpoint file
    - Requires a checkpoint directory set via `sc.setCheckpointDir("hdfs://...")`

> [!NOTE]
> Always `persist()` before `checkpoint()` - otherwise Spark recomputes the RDD twice: once to checkpoint, once for the job.
>
> Checkpointing is critical for iterative algorithms (ML, graph processing) where lineage grows unboundedly across iterations. Without it, a failure in iteration 100 replays all 100 iterations.

- Python -
    ```
    sc.setCheckpointDir("hdfs://checkpoints/")
    rdd.persist()
    rdd.checkpoint()                                # lineage severed after this materializes
    rdd.count()                                     # force materialization NOW before lineage is severed
    ```

| Feature                   | `persist()`                               | `checkpoint()`                                       |
| ------------------------- | ----------------------------------------- | ---------------------------------------------------- |
| Lineage truncated         | No                                        | Yes                                                  |
| Survives executor failure | Only with replication (`_2` levels)       | Yes                                                  |
| Survives Driver failure   | No                                        | Yes                                                  |
| Storage                   | Executor local memory/disk                | Reliable distributed store                           |
| Recompute on eviction     | Yes - from lineage                        | No - reads from checkpoint                           |
| Use case                  | Reuse within one job                      | Iterative algorithms, long lineage                   |
| Cost                      | Low - no extra write if already computing | High - forces a full extra pass if not pre-persisted |
