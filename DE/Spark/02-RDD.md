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

---

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
    ```python
    sc = spark.sparkContext

    rdd = sc.parallelize([1, 2, 3, 4], numSlices=4)
    rdd = sc.textFile("hdfs://path/data.txt", minPartitions=10)
    rdd = sc.sequenceFile("hdfs://path/", keyClass="...", valueClass="...")
    rdd = df.rdd                                                            # DataFrame → RDD[Row]
    ```

- Scala -
    ```scala
    val rdd = sc.parallelize(Seq(1,2,3,4), numSlices = 4)
    val rdd = sc.textFile("hdfs://path/data.txt", minPartitions = 10)
    val rdd = df.rdd                                                        // RDD[Row]
    ```

---

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

---

## Transformations vs Actions

- __Transformation__ -
    - Returns a new RDD
    - Lazy - recorded as a DAG node, zero execution
    - Eg - `map`, `filter`, `flatMap`, `groupByKey`, `join`, `distinct`, `repartition`, `sortBy`, `mapPartitions`, `mapPartitionsWithIndex`, `zip`, `cogroup`
- __Action__ -
    - Triggers DAG execution
    - Returns a value to the Driver or writes to storage
    - Eg - `count`, `collect`, `take`, `first`, `reduce`, `fold`, `aggregate`, `foreach`, `foreachPartition`, `saveAsTextFile`, `saveAsObjectFile`, `countByKey`, `lookup`

---

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

---

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

---

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

---

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
    ```python
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

---

## PairRDD & Specialized RDDs

### PairRDD Operations

- __PairRDD__ - an `RDD[(K, V)]`; no special class, just a Scala implicit enrichment (`PairRDDFunctions`) that activates when the element type is a 2-tuple
- All key-based operations live in `PairRDDFunctions` - 
    - Automatically available on any `RDD[(K, V)]` in Scala/Java
    - In Python, PySpark detects tuple pairs and exposes the same API
- __Implicit conversion__ - 
    - In Scala, `import org.apache.spark.SparkContext._` brings `PairRDDFunctions` into scope
    - This is done automatically by `SparkContext` - you never need to import it explicitly in normal usage

### Aggregation Operations

- __`reduceByKey(f, numPartitions)`__ -
    - Combines values per key using associative, commutative `f`
    - Map-side pre-aggregation (combiner) runs within each partition before shuffle - drastically reduces shuffle data volume
    - Output type must be same as input value type - constraint that `aggregateByKey` lifts
    - Wide dependency, triggers shuffle
    - Internally implemented via `combineByKey` with `createCombiner = identity`, `mergeValue = f`, `mergeCombiners = f`
- __`aggregateByKey(zeroValue)(seqOp, combOp, numPartitions)`__ -
    - `zeroValue` - initial accumulator per partition per key; must be a zero element for `combOp` (not mutated - cloned per key)
    - `seqOp(acc, value)` - merges one value into the accumulator within a partition; called once per record
    - `combOp(acc1, acc2)` - merges two partition-level accumulators after shuffle; must be associative
    - Output type can differ from input value type - key difference from `reduceByKey`
    - Classic use - compute per-key average: `seqOp` accumulates `(sum, count)`, `combOp` merges tuples, then `mapValues` divides
- __`combineByKey(createCombiner, mergeValue, mergeCombiners, numPartitions, mapSideCombine)`__ -
    - Most general key aggregation primitive - `reduceByKey` and `aggregateByKey` are thin wrappers over it
    - `createCombiner(v)` - called once per key per partition on the first value seen; creates the initial combiner
    - `mergeValue(acc, v)` - merges subsequent values into the combiner within the same partition
    - `mergeCombiners(acc1, acc2)` - merges combiners from different partitions after shuffle
    - `mapSideCombine` (default `true`) - controls whether map-side pre-aggregation runs; set to `false` only when combining is more expensive than shuffling raw data
    - `numPartitions` - controls output partition count; defaults to `spark.sql.shuffle.partitions`
- __`groupByKey(numPartitions)`__ -
    - Shuffles ALL values for each key to one partition with no pre-aggregation
    - Returns `RDD[(K, Iterable[V])]`
    - Avoid unless you genuinely need all values together - causes massive shuffle vs `reduceByKey`
    - Memory risk on reduce side - all values for a key must fit in executor memory simultaneously
    - Legitimate use cases - when you need the full value list (eg - building inverted index, deduplication with full context)
- __`foldByKey(zeroValue)(f, numPartitions)`__ -
    - Like `reduceByKey` but with an explicit zero value applied at the start of each partition's aggregation
    - `f` must be associative
    - `zeroValue` must be a true identity element for `f` - wrong zero value produces wrong results silently
- __`countByKey()`__ -
    - Action - returns `Map[K, Long]` to Driver
    - Internally a `map(_ => 1)` + `reduceByKey(_ + _)` + `collect` + convert to map
    - Risk of Driver OOM if key cardinality is high - use `reduceByKey` + `saveAsTextFile` for large cardinality

> [!NOTE]
> `reduceByKey` vs `groupByKey` - the most common Spark interview question at every level.
>
> `reduceByKey` runs `mergeValue` on the map side - each partition emits at most one record per key. `groupByKey` emits every raw record. For 1B records with 1M unique keys, `reduceByKey` shuffles ~1M records; `groupByKey` shuffles 1B. The difference is not just performance - `groupByKey` can OOM executors when value lists are large; `reduceByKey` cannot (it aggregates in a fixed-size buffer).

### Join Operations

- __`join(other, numPartitions)`__ -
    - Inner join - only keys present in both RDDs appear in output
    - Implemented as `cogroup` + `flatMap` to cross-product matching value pairs
    - Wide dependency on both sides unless co-partitioned
- __`leftOuterJoin(other, numPartitions)`__ -
    - All keys from left RDD
    - Right side wrapped in `Some(v)` if present, `None` if absent
    - Implemented as `cogroup` + `flatMap` with `Option` wrapping
- __`rightOuterJoin(other, numPartitions)`__ -
    - All keys from right RDD
    - Left side wrapped in `Option`
- __`fullOuterJoin(other, numPartitions)`__ -
    - All keys from both sides
    - Both sides wrapped in `Option`
- __`cogroup(other, numPartitions)`__ -
    - Groups values from both RDDs by key into `(Iterable[V1], Iterable[V2])`
    - Basis for all join types - all joins are implemented on top of `cogroup`
    - Returns empty iterables (not `None`) for missing keys on either side
    - Supports up to 4 RDDs - `rdd1.cogroup(rdd2, rdd3, rdd4)`
- __`subtractByKey(other, numPartitions)`__ -
    - Returns keys in `this` that are NOT present in `other`
    - Implemented via `cogroup` - keeps keys where right iterable is empty
- Join shuffle behavior -
    - Both sides shuffled if neither has a known partitioner
    - One side skips shuffle if it already uses the same `HashPartitioner(n)` as the join's `numPartitions`
    - If both sides are already co-partitioned with the same partitioner, join is narrow - zero shuffle
    - Smaller side can be broadcast via `sc.broadcast` + `mapPartitions` to avoid shuffle entirely (manual broadcast join at RDD level - DataFrame API does this automatically via `BroadcastHashJoin`)

- Python -
```python
    rdd1 = sc.parallelize([("a", 1), ("b", 2)])
    rdd2 = sc.parallelize([("a", 10), ("c", 30)])

    rdd1.join(rdd2).collect()
    # [("a", (1, 10))]

    rdd1.leftOuterJoin(rdd2).collect()
    # [("a", (1, 10)), ("b", (2, None))]

    rdd1.fullOuterJoin(rdd2).collect()
    # [("a", (Some(1), Some(10))), ("b", (Some(2), None)), ("c", (None, Some(30)))]

    rdd1.cogroup(rdd2).collect()
    # [("a", ([1], [10])), ("b", ([2], [])), ("c", ([], [30]))]

    rdd1.subtractByKey(rdd2).collect()
    # [("b", 2)]
```

### Key/Value Manipulation

- __`mapValues(f)`__ -
    - Applies `f` to values only
    - Preserves partitioner - critical optimization; downstream joins remain narrow
    - Narrow dependency - no shuffle even on a partitioned RDD
    - Does not reserialize the key - slightly more efficient than `map` on the full tuple
- __`flatMapValues(f)`__ -
    - Like `mapValues` but `f` returns an iterable; output has one record per element of the iterable
    - Preserves partitioner
    - Use for one-to-many value expansions without disturbing key-based partitioning
- __`keys()`__ - returns `RDD[K]`; loses partitioner (result is plain RDD)
- __`values()`__ - returns `RDD[V]`; loses partitioner
- __`mapKeys` (no dedicated method)__ -
    - Use `rdd.map(lambda kv: (f(kv[0]), kv[1]))` - always loses partitioner
    - Spark cannot prove the new key maintains the same partition assignment
- __`sortByKey(ascending=True, numPartitions=None)`__ -
    - Assigns `RangePartitioner` - range-partitions data then sorts within each partition
    - Result is globally sorted when partitions are read in order
    - Wide dependency
    - `numPartitions` controls output partition count; more partitions = finer-grained sort but more tasks
- __`partitionBy(partitioner)`__ -
    - Repartitions the RDD using the given `Partitioner`
    - Assigns the partitioner to the resulting RDD - critical for co-partitioning detection on downstream joins
    - Always `persist()` after `partitionBy` if the RDD is reused - otherwise the shuffle repeats on every action
    - The partitioner is stored as metadata on the RDD; Spark checks it during join planning

> [!NOTE]
> `mapValues` preserves the partitioner because keys don't change - Spark knows the partition assignment is still valid. `map` always loses the partitioner even if you don't touch the key, because Spark cannot statically prove the key is unchanged without inspecting the function body.

### Lookup & Sampling

- __`lookup(key)`__ -
    - Action - returns `Seq[V]` for a given key
    - If RDD has a known partitioner, only the relevant partition is scanned - O(partition_size) not O(total_size)
    - Without a partitioner, scans all partitions - full O(n) scan
    - Not suitable for high-frequency lookups - each call is a Spark job; use `collectAsMap` + local map for repeated lookups on small RDDs
- __`sampleByKey(withReplacement, fractions, seed)`__ -
    - Stratified sample - each key sampled at its own specified rate
    - `fractions` is `Map[K, Double]` - key to sampling fraction
    - `withReplacement=False` uses Bernoulli sampling; `True` uses Poisson sampling
    - Does not guarantee exact counts - use `sampleByKeyExact` for exact counts (more expensive - makes two passes)

### Key-Level Statistics

- __`countByKey()`__ -
    - Action - collects `Map[K, Long]` to Driver
    - Use `reduceByKey(_ + _)` + `saveAsTextFile` for large key cardinality to avoid Driver OOM
- __`countByValue()`__ -
    - On plain RDD - counts occurrences of each distinct element
    - Equivalent to `map(x => (x, 1)).reduceByKey(_ + _).collectAsMap()`

### Numeric PairRDD Operations

- __`sumByKey`__ - not a built-in; use `reduceByKey(_ + _)`
- __`meanByKey`__ - not a built-in; use `aggregateByKey((0.0, 0))((acc, v) => (acc._1 + v, acc._2 + 1), (a, b) => (a._1 + b._1, a._2 + b._2)).mapValues(x => x._1 / x._2)`
- __`collectAsMap()`__ -
    - Action - collects `RDD[(K, V)]` into a `Map[K, V]` on the Driver
    - If duplicate keys exist, last value wins
    - OOM risk on large RDDs - use only for small lookup tables

---

## CoGroupedRDD

- __`CoGroupedRDD`__ - the internal RDD that backs all multi-RDD key-based operations (`join`, `cogroup`, `subtractByKey`, `intersection`)
- Groups values from N parent RDDs by key into a single partition - produces `RDD[(K, Array[Iterable[_]])]` internally
- `cogroup` on two RDDs returns `RDD[(K, (Iterable[V1], Iterable[V2]))]`
- `join` is `cogroup` + `flatMap` to cross-product and unwrap non-empty iterables
- Not usually constructed directly - created internally by `cogroup`

### How CoGroupedRDD Works

- Examines the partitioners of all parent RDDs at construction time -
    - If a parent already has the target partitioner → `NarrowDependency` for that parent (data stays local)
    - If a parent has no partitioner or a different one → `ShuffleDependency` for that parent (data shuffled)
- This means a `cogroup` of two co-partitioned RDDs produces zero shuffles - pure narrow dependency
- Each partition of `CoGroupedRDD` builds an in-memory `ExternalAppendOnlyMap` per parent RDD -
    - Keys are hashed into buckets
    - Values appended to per-key lists
    - Map spills to disk if memory pressure exceeds threshold (controlled by `spark.shuffle.spill`)
- After all parent data is collected into the map, iterates over keys and emits `(key, (iter1, iter2, ...))`

### Dependency Resolution Detail

- `CoGroupedRDD.getDependencies` logic -
    - For each parent RDD, check if `parent.partitioner == Some(targetPartitioner)`
    - If yes → `OneToOneDependency` (narrow)
    - If no → `ShuffleDependency` with the target partitioner applied to parent
- Target partitioner defaults to `HashPartitioner(spark.sql.shuffle.partitions)` unless overridden

### Memory Implications

- All values for every key landing in a given `CoGroupedRDD` partition must fit in executor memory
- `ExternalAppendOnlyMap` spills to disk when memory is exhausted - prevents OOM but causes disk I/O
- This is why `groupByKey` has OOM risk -
    - `groupByKey` is `cogroup` with one parent
    - All values per key land in one iterable in memory before being returned
    - `reduceByKey` never builds this full list - it merges values incrementally
- `join` on two RDDs where one key has millions of values will OOM even if `join` looks safe - the cross product of matching values is the actual memory consumer

### ExternalAppendOnlyMap

- Spark's spill-aware hash map used inside `CoGroupedRDD` and `aggregateByKey`
- Appends values to per-key lists; when memory threshold exceeded, sorts by key and spills to disk
- On iteration, merges in-memory map with all spill files using a merge-sort
- Controlled by `spark.shuffle.spill.initialMemoryThreshold` and `spark.shuffle.spill.batchSize`

- Python -
```python
    rdd1 = sc.parallelize([("a", 1), ("a", 2), ("b", 3)])
    rdd2 = sc.parallelize([("a", 10), ("b", 20), ("b", 30)])

    # cogroup - basis for all joins
    rdd1.cogroup(rdd2).mapValues(lambda x: (list(x[0]), list(x[1]))).collect()
    # [("a", ([1, 2], [10])), ("b", ([3], [20, 30]))]

    # join is cogroup + flatMap
    rdd1.join(rdd2).collect()
    # [("a", (1, 10)), ("a", (2, 10)), ("b", (3, 20)), ("b", (3, 30))]

    # subtractByKey via cogroup
    rdd1.subtractByKey(rdd2).collect()
    # [] - all keys in rdd1 exist in rdd2
```

- Scala -
```scala
    val rdd1 = sc.parallelize(Seq(("a", 1), ("a", 2), ("b", 3)))
    val rdd2 = sc.parallelize(Seq(("a", 10), ("b", 20), ("b", 30)))

    rdd1.cogroup(rdd2).mapValues { case (v1, v2) => (v1.toList, v2.toList) }.collect()
    // Array((a,(List(1, 2),List(10))), (b,(List(3),List(20, 30))))

    // 3-way cogroup
    val rdd3 = sc.parallelize(Seq(("a", 100)))
    rdd1.cogroup(rdd2, rdd3).collect()
    // Array((a,(List(1,2), List(10), List(100))), (b,(List(3), List(20,30), List())))
```

> [!NOTE]
> `cogroup` supports up to 4 RDDs in a single call. Beyond that, chain manually. The dependency type (narrow vs shuffle) is determined independently per parent - cogrouping 3 RDDs where 2 are co-partitioned results in 2 narrow deps and 1 shuffle dep.

---

## ShuffledRDD

- __`ShuffledRDD`__ - the concrete RDD class produced by any wide transformation that requires redistributing data across the network
    - Has exactly one parent and exactly one `ShuffleDependency`
- Produced by - `reduceByKey`, `groupByKey`, `aggregateByKey`, `combineByKey`, `sortByKey`, `repartition`, `partitionBy`, `distinct`, `coalesce(shuffle=True)`
- Not usually constructed directly - Spark internals create it; but understanding it is essential for shuffle tuning

### ShuffledRDD Fields

- `prev` - the parent RDD
- `part` - the `Partitioner` applied to output
- `keyOrdering` - optional `Ordering[K]` for sorted output within partitions
- `aggregator` - optional `Aggregator[K, V, C]` encapsulating `createCombiner`, `mergeValue`, `mergeCombiners`
- `mapSideCombine` - whether to apply the aggregator on the map side before writing shuffle files

### Map Side - Shuffle Write

- Each map task writes shuffle output for ALL output partitions
- `SortShuffleManager` (default since Spark 1.2, replacing `HashShuffleManager`) manages write path selection
- __`BypassMergeSortShuffleWriter`__ -
    - Conditions - no map-side aggregation AND `numPartitions ≤ spark.shuffle.sort.bypassMergeThreshold` (default $200$)
    - Writes one temporary file per output partition, then concatenates all into one data file + one index file
    - No sort within partitions - fastest for small partition counts
    - Memory usage - one open file handle per output partition; can exhaust file descriptors at high partition counts
- __`UnsafeShuffleWriter`__ -
    - Conditions - serializer supports object relocation (Kryo or Spark's own unsafe serializer) AND no map-side aggregation AND `numPartitions ≤ 16777216`
    - Sorts serialized binary records by partition ID using `ShuffleExternalSorter` (off-heap sort buffer)
    - Never deserializes records during sort - operates on raw bytes
    - Most CPU-efficient write path for medium-to-large partition counts
- __`SortShuffleWriter`__ -
    - Default fallback for all other cases (map-side aggregation required, or serializer doesn't support relocation)
    - Uses `ExternalSorter` - deserializes records, inserts into `PartitionedAppendOnlyMap` (if aggregating) or `PartitionedPairBuffer` (if not)
    - Spills to disk when memory exceeds threshold, merges spill files at the end
    - Output is one sorted data file per mapper + one index file

### Map Side - Output Files

- Each mapper produces exactly 2 files regardless of partition count (since `SortShuffle`) -
    - `shuffle_{shuffleId}_{mapId}_{reducerId}.data` - the actual data, all partitions concatenated
    - `shuffle_{shuffleId}_{mapId}_{reducerId}.index` - byte offsets for each partition within the data file
- Old `HashShuffleManager` produced `numMappers × numReducers` files - caused millions of small files at scale; replaced for this reason
- With `External Shuffle Service` - these files are served by the node daemon, not the executor JVM
    - Executor can die and be relaunched without losing shuffle output

### Reduce Side - Shuffle Read

- Each reduce task fetches its partition slice from every map task's output file
- `BlockStoreShuffleReader` orchestrates the fetch -
    - Queries `MapOutputTrackerMaster` on Driver for locations of all map outputs for this shuffle
    - `MapOutputTrackerMaster` returns `MapStatus` array - one per map task, containing executor location and partition sizes
    - Fetches blocks from remote `BlockManager`s via HTTP (Netty)
    - Fetches local blocks directly from local `BlockManager` without network
- `spark.reducer.maxSizeInFlight` (default $48 MB$) - total data in-flight per reduce task across all fetch requests
    - Controls memory pressure during fetch
    - Higher value = more parallelism in fetching but more memory consumed
- `spark.reducer.maxReqsInFlight` (default `Int.MaxValue`) - max simultaneous fetch requests per reduce task
- `spark.reducer.maxBlocksInFlightPerAddress` (default `Int.MaxValue`) - max blocks fetched from one executor at a time
    - Tune to avoid overwhelming a single executor that holds many map outputs

### Shuffle Read - Aggregation and Sort

- After fetching, `ExternalSorter` or `ExternalAppendOnlyMap` processes records -
    - If aggregation required - inserts into `ExternalAppendOnlyMap`, applies `mergeCombiners`
    - If sort required - sorts by key using `keyOrdering`
    - If neither - records passed through directly
- Spill behavior same as map side - controlled by `spark.shuffle.spill` and memory thresholds

### Shuffle Persistence and Cleanup

- Shuffle files persist on executor local disk until the job completes or the shuffle is explicitly unregistered
- `MapOutputTrackerMaster` tracks all shuffle registrations
- On job completion, `DAGScheduler` calls `mapOutputTracker.unregisterShuffle(shuffleId)` which triggers cleanup
- Shuffle files NOT cleaned up if executor dies without External Shuffle Service - this causes stage recomputation

> [!TIP]
> Shuffle tuning levers -
>   - `spark.sql.shuffle.partitions` - output partition count (default $200$, almost always wrong; target $128 MB$ per partition)
>   - `spark.shuffle.compress` - compress shuffle files (default `true`; use `lz4` codec)
>   - `spark.shuffle.file.buffer` - map-side write buffer per output partition (default $32 KB$; increase to $1 MB$ for large shuffles)
>   - `spark.reducer.maxSizeInFlight` - reduce-side fetch buffer (default $48 MB$; increase for fast networks)
>   - `spark.shuffle.sort.bypassMergeThreshold` - bypass writer threshold (default $200$; lower if file descriptor limit is hit)
>   - `spark.shuffle.io.maxRetries` - fetch retry count (default $3$; increase for flaky networks)
>   - `spark.shuffle.io.retryWait` - wait between retries (default $5s$)

---

## UnionRDD

- __`UnionRDD`__ - combines N parent RDDs into one logical RDD without any data movement
    - Partitions from all parents are concatenated in order
- `sc.union(rdds)` or `rdd1.union(rdd2)` -
    - `sc.union` more efficient for many RDDs - creates one `UnionRDD` with N parents
    - `rdd1.union(rdd2).union(rdd3)` chains binary `UnionRDD`s - creates a deep lineage tree; avoid for large N
- Partition count = sum of all parent partition counts
- All `NarrowDependency` (`RangeDependency` for each parent) - zero network transfer, zero shuffle

### UnionRDD Internals

- `UnionRDD.getPartitions` - concatenates partition arrays from all parents; assigns each a `UnionPartition` with a parent index and parent partition index
- `UnionRDD.compute(split)` - delegates to `parent.iterator(parentPartition, context)` for the relevant parent
- `UnionRDD.getDependencies` - returns one `RangeDependency` per parent -
    - `RangeDependency(parent, inStart, outStart, length)` - maps child partitions `[outStart, outStart+length)` to parent partitions `[inStart, inStart+length)`
- Preferred locations - delegates to parent RDD's preferred locations for each partition

### UnionRDD Behavior

- Does NOT deduplicate -
    - `union` is a multiset (bag) union
    - Use `distinct()` after if set semantics needed (triggers shuffle)
- Does NOT rebalance partitions -
    - If parent RDDs have very different partition sizes, `UnionRDD` inherits the imbalance
    - Use `repartition` after `union` if downstream tasks need balanced partitions
- Partitioner is always `None` on the result -
    - Even if all parents share the same partitioner, `UnionRDD` loses it
    - Reason - partitioner implies a mapping from key to partition index; after union, partition indices are remapped (offset by parent partition counts), breaking the mapping
    - Downstream joins on a `UnionRDD` result will trigger a full shuffle even if all inputs were identically partitioned

> [!NOTE]
> The partitioner loss is the most dangerous `UnionRDD` gotcha in production. Pattern -
> ```python
> rdd1 = large_rdd.partitionBy(100)   # HashPartitioner(100)
> rdd2 = other_rdd.partitionBy(100)   # HashPartitioner(100)
> union = rdd1.union(rdd2)            # partitioner = None
> union.join(third_rdd)               # full shuffle - NOT narrow
> ```
> Fix - `union.partitionBy(100).join(third_rdd)` - re-apply partitioner after union.

### Performance Characteristics

- `union` itself is free - no computation, no data movement
- The cost is paid downstream -
    - If downstream operations need a shuffle (join, groupBy), they shuffle the full unioned dataset
    - If downstream operations are narrow (filter, map), they run on each partition independently - still free
- `sc.union` on thousands of small RDDs creates a `UnionRDD` with thousands of partitions -
    - Scheduler overhead becomes significant
    - Use `coalesce` to reduce partition count before further processing

### When to Use

- Combining sharded input files read as separate RDDs before aggregation
- Merging results of parallel independent computations (eg - per-date processing) into one RDD
- Building a single RDD from heterogeneous sources before a `reduceByKey`
- Incremental processing - union new data RDD with existing RDD before re-aggregating

- Python -
```python
    rdd1 = sc.parallelize([("a", 1), ("b", 2)], 2)
    rdd2 = sc.parallelize([("c", 3), ("d", 4)], 3)
    rdd3 = sc.parallelize([("e", 5)], 1)

    # Efficient - single UnionRDD with 3 parents
    union = sc.union([rdd1, rdd2, rdd3])
    union.getNumPartitions()                # 6 (2 + 3 + 1)
    union.partitioner                       # None

    # Inefficient for large N - chained binary unions
    bad = rdd1.union(rdd2).union(rdd3)      # nested UnionRDDs

    # Re-partition after union if needed
    union.partitionBy(10).join(other_rdd)   # now join is co-partitioned
```

---

## HadoopRDD

- __`HadoopRDD`__ - the leaf RDD that reads data from any Hadoop `InputFormat`
    - Every `sc.textFile`, `sc.sequenceFile`, `sc.hadoopFile` call produces a `HadoopRDD` (or `NewHadoopRDD`) under the hood
- Uses the old MapReduce API (`org.apache.hadoop.mapred`)
- `NewHadoopRDD` - equivalent for the new API (`org.apache.hadoop.mapreduce`); used by `sc.newAPIHadoopFile`
- Both are leaf RDDs - they have no parent RDDs; they read from external storage

### HadoopRDD Internals

- Each partition is a `HadoopPartition` wrapping one `InputSplit` -
    - For `TextInputFormat` - one `InputSplit` per HDFS block (default $128 MB$)
    - For `CombineTextInputFormat` - multiple blocks combined into one split (reduces task count for small files)
    - Split boundaries honored - records spanning HDFS block boundaries are read by the task owning the start of the record
- `compute(split, context)` -
    - Constructs a `JobConf` for the split
    - Calls `InputFormat.getRecordReader(split, conf, reporter)` to open a `RecordReader`
    - Returns a `NextIterator` - calls `reader.next(key, value)` on each `next()` invocation
    - Registers a `TaskCompletionListener` to close the `RecordReader` when task completes (or fails)
    - True streaming - only the current record is in memory; one `RecordReader` open per task
- Preferred locations -
    - Calls `InputFormat.getSplits` to get splits, then `split.getLocations()` for each split
    - HDFS returns the 3 replica node hostnames
    - `DAGScheduler` uses these to schedule tasks on data-local nodes

### HadoopRDD Serialization Problem

- `HadoopRDD` holds a reference to `JobConf` which is not serializable in all cases
- More critically, `RecordReader` holds open file handles and cannot be serialized
- Task closures captured over a `HadoopRDD` directly will fail to serialize
- This is safe -
    ```python
    # Safe - closure captures the mapped RDD, not the HadoopRDD
    rdd = sc.textFile("hdfs://...")
    mapped = rdd.map(lambda x: x.upper())   # mapped is a MappedRDD
    mapped.cache()                          # fine
    ```

- This will fail -
    ```python
    # Unsafe - trying to serialize the HadoopRDD itself
    sc.broadcast(sc.textFile("hdfs://..."))  # fails - RecordReader not serializable
    ```

### Partition Count Control

- `minPartitions` in `sc.textFile` -
    - Calls `TextInputFormat.getSplits(job, minSplits)` - `minSplits` is a hint, not a guarantee
    - HDFS may return more splits if individual blocks are small
    - Cannot produce fewer partitions than HDFS blocks - use `coalesce` after reading to reduce
- Large number of small files -
    - Each file produces at least one partition regardless of size (even 1-byte files)
    - 10,000 small files = 10,000 partitions = 10,000 tasks = massive scheduler overhead
    - Solutions -
        - `sc.wholeTextFiles(path)` - reads each file as one record `(filename, content)`; reduces partition count but loads entire file into memory
        - `CombineTextInputFormat` - combine multiple small files into one split; set via `hadoopConfiguration`
        - Compact files upstream before ingestion (preferred)

### Configuration Access

- `HadoopRDD` exposes `hadoopConfiguration` for tuning `InputFormat` behavior

- Python -
```python
    # textFile
    rdd = sc.textFile("hdfs://path/data.txt", minPartitions=10)

    # SequenceFile
    rdd = sc.sequenceFile("hdfs://path/",
                          keyClass="org.apache.hadoop.io.Text",
                          valueClass="org.apache.hadoop.io.IntWritable")

    # Custom InputFormat (old API)
    rdd = sc.hadoopFile("hdfs://path/",
                        inputFormatClass="com.example.MyInputFormat",
                        keyClass="org.apache.hadoop.io.LongWritable",
                        valueClass="org.apache.hadoop.io.Text")

    # Custom InputFormat (new API)
    rdd = sc.newAPIHadoopFile("hdfs://path/",
                              inputFormatClass="com.example.MyNewInputFormat",
                              keyClass="org.apache.hadoop.io.LongWritable",
                              valueClass="org.apache.hadoop.io.Text")

    # With Hadoop configuration overrides
    conf = {"mapreduce.input.fileinputformat.split.maxsize": "67108864"}  # 64 MB splits
    rdd = sc.newAPIHadoopFile("hdfs://path/", ..., conf=conf)

    # wholeTextFiles - entire file as one record
    rdd = sc.wholeTextFiles("hdfs://path/small_files/", minPartitions=100)
    # RDD[(filename, file_content)]
```

- Scala -
```scala
    // textFile
    val rdd = sc.textFile("hdfs://path/data.txt", minPartitions = 10)

    // SequenceFile
    val rdd = sc.sequenceFile[Text, IntWritable]("hdfs://path/")

    // Custom InputFormat (old API)
    val rdd = sc.hadoopFile[LongWritable, Text, MyInputFormat]("hdfs://path/")

    // Custom InputFormat (new API)
    val rdd = sc.newAPIHadoopFile[LongWritable, Text, MyNewInputFormat]("hdfs://path/")

    // With configuration overrides
    sc.hadoopConfiguration.set("mapreduce.input.fileinputformat.split.maxsize", "67108864")
    val rdd = sc.textFile("hdfs://path/")
```

> [!TIP]
> `HadoopRDD` and `NewHadoopRDD` reuse the same `Writable` key/value objects across records for efficiency - the `RecordReader` mutates the same object on each `next()` call. If you call `collect()` on a `HadoopRDD` without mapping first, all collected records will be the same object (the last record). Always `map` or `mapValues` to extract or copy values before collecting.

---

## HashPartitioner

- __`HashPartitioner(numPartitions)`__ - maps keys to partitions via `partition = nonNegativeMod(key.hashCode(), numPartitions)`
- Default partitioner for all key-based shuffle operations - `reduceByKey`, `groupByKey`, `aggregateByKey`, `combineByKey`, `join`, `cogroup`, `partitionBy` without explicit partitioner
- `nonNegativeMod(x, mod)` - `((x % mod) + mod) % mod` - ensures non-negative result for negative `hashCode()` values

### Correctness Requirements

- Keys must satisfy the Java `hashCode`/`equals` contract -
    - `a.equals(b)` implies `a.hashCode() == b.hashCode()`
    - Violating this sends equal keys to different partitions - `reduceByKey` produces wrong results silently
- Mutable keys are dangerous -
    - If a key object is mutated after being placed in a partition, its hash may change
    - Key becomes unfindable in the partition's hash structure
    - Always use immutable keys
- Python -
    - Uses Python's `hash()` function via `portable_hash` (handles cross-platform hash consistency)
    - Strings, ints, tuples are safe
    - Custom objects need `__hash__` and `__eq__`; `__hash__` must be consistent with `__eq__`
- Scala case classes -
    - `hashCode` and `equals` auto-generated from constructor fields
    - Safe by default
- Java/Scala Arrays -
    - Use object identity for `hashCode` (memory address)
    - Two arrays with identical content hash differently
    - Use `Seq`, `List`, or `Vector` as keys instead
- `None`/`null` keys -
    - Java `null` has `hashCode() == 0` → always goes to partition `0`
    - All null-keyed records pile up on one partition - guaranteed skew

### Co-partitioning Detection

- Two RDDs are co-partitioned if `rdd1.partitioner == rdd2.partitioner` (Scala `Option` equality)
- `HashPartitioner.equals` is defined as `other.isInstanceOf[HashPartitioner] && other.numPartitions == this.numPartitions`
- Implication - any two RDDs partitioned with `HashPartitioner(n)` for the same `n` are co-partitioned, regardless of which job created them
- Join co-partitioning check -
    - `DAGScheduler` checks `leftRDD.partitioner == rightRDD.partitioner` before planning join stage
    - If equal → narrow dependency for the already-partitioned side; only the other side shuffles (or neither if both match)
    - If not equal → both sides shuffled to `HashPartitioner(numPartitions)` where `numPartitions = spark.sql.shuffle.partitions`

### Data Skew with HashPartitioner

- Hot keys -
    - One key with disproportionately many values → one partition gets all of them
    - One task runs for hours; all others finish in seconds
    - Classic symptom - Spark UI shows 99% tasks complete but job stalls on 1 task
- Diagnosing skew -
    - Check partition sizes via `rdd.mapPartitionsWithIndex(lambda i, it: [(i, sum(1 for _ in it))]).collect()`
    - Check Spark UI task metrics - look for one task with much higher input size or shuffle read size
- Solutions -
    - __Salting__ -
        - Append random suffix to hot key - `(key + "_" + random(0, n), value)`
        - Distribute records for one logical key across `n` partitions
        - Requires two-phase aggregation - first aggregate salted keys, then strip suffix and aggregate again
    - __Custom partitioner__ - assign known hot keys to dedicated partitions or spread them explicitly
    - __Broadcast join__ - if one RDD is small enough, broadcast it and avoid shuffle entirely
    - __Skew join__ - handle hot keys separately with broadcast, merge with non-hot key join result

- Python -
```python
    from pyspark.rdd import portable_hash

    rdd = pairs.partitionBy(10)             # HashPartitioner(10) applied
    rdd.partitioner                         # HashPartitioner(10)

    # Verify co-partitioning
    rdd2 = other_pairs.partitionBy(10)
    rdd.partitioner == rdd2.partitioner     # True - join will be narrow

    # Salting for skew mitigation
    import random
    n_salt = 10
    salted = skewed_rdd.map(lambda kv: (
        (kv[0], random.randint(0, n_salt - 1)), kv[1]
    ))
    partial = salted.reduceByKey(lambda a, b: a + b)
    result = partial.map(lambda kv: (kv[0][0], kv[1])).reduceByKey(lambda a, b: a + b)
```

---

## RangePartitioner

- __`RangePartitioner(numPartitions, rdd, ascending=True, samplePointsPerPartitionHint=20)`__ -
    - Assigns keys to partitions by sorted range
    - Keys in partition `i` are all ≤ keys in partition `i+1`
    - Result is globally ordered across partitions when read sequentially
- Used internally by `sortByKey` - the only standard way to produce a globally sorted RDD
- Requires keys to be `Ordered` (Scala `scala.math.Ordering`) or `Comparable` (Java); Python uses natural comparison

### Sampling Phase - How Boundaries Are Determined

- `RangePartitioner` construction triggers a real Spark job on the parent RDD -
    - This is not lazy - constructing the object executes computation
- Two-phase sampling process -
    - __Phase 1 - sketch__ - each partition independently samples `math.ceil(samplePointsPerPartitionHint * numPartitions / rdd.partitions.length)` records using reservoir sampling
        - Sample size per partition bounded to avoid Driver OOM
        - Reservoir sampling ensures uniform random sample without knowing partition size in advance
    - __Phase 2 - determine boundaries__ - all samples collected to Driver, sorted, then `numPartitions - 1` quantile points selected as boundaries
        - Total sample size = `math.min(samplePointsPerPartitionHint * numPartitions, 1e6)`
        - More partitions = more samples = more accurate boundaries = more Driver memory
- Sampling accuracy -
    - With enough samples, partition sizes will be roughly equal by record count
    - Skewed data with non-uniform key distribution → unequal partition sizes even with good sampling
    - Increase `samplePointsPerPartitionHint` for more accurate boundaries at cost of Driver memory

### Boundary Array and getPartition

- After sampling, boundaries stored as a sorted array of `numPartitions - 1` keys
- `getPartition(key)` -
    - Binary searches the boundary array
    - O(log numPartitions)
    - Returns index of the first boundary ≥ key
- Keys exactly equal to a boundary go to the lower partition (left-inclusive ranges)

### RangePartitioner Equality

- Two `RangePartitioner` instances are equal only if -
    - Same `numPartitions`
    - Same boundary array (element-wise equality)
- Since boundaries come from sampling, two independently constructed `RangePartitioner`s on the same data will almost certainly have different boundaries and will NOT be equal
- Implication - you cannot co-partition two RDDs with independently constructed `RangePartitioner`s; use the same instance

- Python -
```python
    pairs = sc.parallelize([(i, i) for i in range(1000)], 10)

    # Via sortByKey - creates RangePartitioner internally
    sorted_rdd = pairs.sortByKey(ascending=True, numPartitions=4)
    sorted_rdd.partitioner                      # RangePartitioner

    # Verify global sort - each partition's max key < next partition's min key
    sorted_rdd.mapPartitionsWithIndex(
        lambda i, it: [(i, list(it))]
    ).collect()
```

- Scala -
```scala
    import org.apache.spark.RangePartitioner

    val pairs = sc.parallelize((0 until 1000).map(i => (i, i)), 10)

    // Explicit construction - triggers sampling job immediately
    val rp = new RangePartitioner(4, pairs)
    println(rp.numPartitions)               // 4

    // Partition an RDD using the same instance - no sampling again
    val partitioned1 = pairs.partitionBy(rp)
    val partitioned2 = other_pairs.partitionBy(rp)    // co-partitioned with partitioned1

    // Via sortByKey
    val sorted = pairs.sortByKey(ascending = true, numPartitions = 4)
    sorted.partitioner                      // Some(RangePartitioner)
```

> [!NOTE]
> `RangePartitioner` vs `HashPartitioner` for joins - never use `RangePartitioner` for joins unless you explicitly need sorted output. The sampling cost is paid at construction, and two independently constructed instances won't be equal. `HashPartitioner(n)` instances with the same `n` are always equal - much safer for co-partitioning.

---

## Custom Partitioner

- __Custom Partitioner__ - extend `org.apache.spark.Partitioner`
    - Override `numPartitions: Int` - total number of output partitions
    - Override `getPartition(key: Any): Int` - maps key to partition index in `[0, numPartitions)`
    - Override `equals(other: Any): Boolean` - REQUIRED for co-partitioning detection
    - Override `hashCode: Int` - REQUIRED; must be consistent with `equals`
- Use when `HashPartitioner` causes skew on known hot keys, or when domain-specific routing is needed

### When Custom Partitioner is the Right Choice

- __Known hot keys__ - route hot keys to dedicated partitions or spread them across multiple partitions explicitly
- __Locality requirements__ - co-locate related keys on the same partition to enable partition-local joins (avoid shuffle entirely for downstream operations)
- __External system alignment__ - match an external system's sharding strategy (eg - Cassandra token ranges, HBase region boundaries, Kafka partition assignments)
- __Avoiding re-partitioning__ - pre-partition input data once to match all downstream join keys; pay shuffle cost once, all subsequent joins are narrow
- __Bucketing equivalent__ - RDD-level equivalent of DataFrame bucketing; partition on join key so all future joins on that key are shuffle-free

### Implementation Requirements

- `getPartition(key)` -
    - Must return value in `[0, numPartitions)` - values outside this range cause `ArrayIndexOutOfBoundsException`
    - Must be deterministic - same key must always return same partition
    - Must be fast - called once per record on the map side; expensive logic here multiplies by total record count
    - Must handle all possible key types - `key` is `Any`/`Object`; always cast safely
- `equals(other)` -
    - Two partitioners are equal iff they assign every possible key to the same partition
    - Spark uses `partitioner1 == partitioner2` to decide if a join is narrow
    - Failing to implement `equals` means every two instances are unequal → always shuffles
- `hashCode()` -
    - Must satisfy `a == b` implies `a.hashCode == b.hashCode`
    - Used when partitioners are stored in hash-based collections

### Hot Key Partitioner Pattern

- Python -
```python
    from pyspark import Partitioner

    class TenantPartitioner(Partitioner):
        def __init__(self, num_partitions, hot_tenants):
            self.num_partitions = num_partitions
            # Hot tenants get dedicated partitions at fixed indices
            self.hot_tenant_map = {t: i for i, t in enumerate(sorted(hot_tenants))}
            self.num_hot = len(hot_tenants)
            self.remaining = num_partitions - self.num_hot
            assert self.remaining > 0, "num_partitions must exceed number of hot tenants"

        def numPartitions(self):
            return self.num_partitions

        def getPartition(self, key):
            tenant = key.split(":")[0]
            if tenant in self.hot_tenant_map:
                return self.hot_tenant_map[tenant]
            # Non-hot tenants hash into the remaining partitions
            return self.num_hot + (hash(tenant) % self.remaining)

        def __eq__(self, other):
            return (isinstance(other, TenantPartitioner)
                    and other.num_partitions == self.num_partitions
                    and other.hot_tenant_map == self.hot_tenant_map)

        def __hash__(self):
            return hash((self.num_partitions, tuple(sorted(self.hot_tenant_map.items()))))

    hot = ["tenantA", "tenantB"]
    partitioner = TenantPartitioner(20, hot)
    rdd1 = events.partitionBy(partitioner)
    rdd2 = profiles.partitionBy(partitioner)    # same instance → co-partitioned
    rdd1.join(rdd2)                             # narrow - no shuffle
```

- Scala -
```scala
    import org.apache.spark.Partitioner

    class TenantPartitioner(
        override val numPartitions: Int,
        hotTenants: Seq[String]
    ) extends Partitioner {

        // Sort hot tenants for deterministic index assignment
        private val hotMap: Map[String, Int] =
            hotTenants.sorted.zipWithIndex.toMap
        private val numHot = hotMap.size
        private val remaining = numPartitions - numHot
        require(remaining > 0, "numPartitions must exceed hot tenant count")

        override def getPartition(key: Any): Int = {
            val tenant = key.toString.split(":")(0)
            hotMap.get(tenant) match {
                case Some(idx) => idx
                case None =>
                    numHot + (Math.abs(tenant.hashCode) % remaining)
            }
        }

        override def equals(other: Any): Boolean = other match {
            case p: TenantPartitioner =>
                p.numPartitions == numPartitions && p.hotMap == hotMap
            case _ => false
        }

        override def hashCode: Int =
            31 * numPartitions + hotMap.hashCode
    }

    val partitioner = new TenantPartitioner(20, Seq("tenantA", "tenantB"))
    val rdd1 = events.partitionBy(partitioner).cache()
    val rdd2 = profiles.partitionBy(partitioner).cache()
    rdd1.join(rdd2)     // narrow - zero shuffle
```

### Locality-Based Partitioner Pattern

- Co-locate keys that will be joined together on the same partition
- Eg - partition by `(userId % n)` so all user-related data across multiple RDDs lands on the same executor

- Scala -
```scala
    // Partition all user data to the same executor
    class UserPartitioner(override val numPartitions: Int) extends Partitioner {
        override def getPartition(key: Any): Int =
            Math.abs(key.toString.toInt % numPartitions)

        override def equals(other: Any): Boolean = other match {
            case p: UserPartitioner => p.numPartitions == numPartitions
            case _ => false
        }

        override def hashCode: Int = numPartitions
    }

    val userPartitioner = new UserPartitioner(100)

    // Partition all datasets by userId once
    val events    = rawEvents.map(e => (e.userId, e)).partitionBy(userPartitioner).cache()
    val profiles  = rawProfiles.map(p => (p.userId, p)).partitionBy(userPartitioner).cache()
    val purchases = rawPurchases.map(p => (p.userId, p)).partitionBy(userPartitioner).cache()

    // All subsequent joins are narrow - no shuffle
    events.join(profiles).join(purchases)
```

### Co-partitioning with Custom Partitioner

- Two RDDs are co-partitioned only if `rdd1.partitioner.get == rdd2.partitioner.get` (uses `equals`)
- Using the exact same instance guarantees co-partitioning (reference equality implies value equality)
- Using two separately constructed instances with the same parameters works only if `equals` is correctly implemented
- A common mistake -
```python
    # Wrong - two different instances; equals not overridden → never co-partitioned
    rdd1 = data1.partitionBy(MyPartitioner(10))
    rdd2 = data2.partitionBy(MyPartitioner(10))
    rdd1.join(rdd2)     # shuffles both sides despite looking co-partitioned
```

### Partitioner Comparison

| Partitioner                | Key Requirement          | Distribution                | Globally Sorted | Construction Cost         | Co-partition Detection                    |
| -------------------------- | ------------------------ | --------------------------- | --------------- | ------------------------- | ----------------------------------------- |
| `HashPartitioner(n)`       | `hashCode` + `equals`    | Uniform if keys are uniform | No              | Zero                      | `numPartitions` equality                  |
| `RangePartitioner(n, rdd)` | `Ordered` / `Comparable` | Uniform by record count     | Yes             | Sampling job on RDD       | `numPartitions` + boundary array equality |
| `CustomPartitioner`        | Domain-specific          | Controlled                  | Optional        | As complex as you make it | `equals()` + `hashCode()` implementation  |
