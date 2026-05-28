# Resilient Distributed Dataset

- RDD = immutable, partitioned, distributed collection of records
- Core abstraction underneath Spark execution
- Transformations create new RDDs; actions trigger execution
- Low-level API - gives direct control over partitioning, serialization, locality etc
- Typed -
    - `RDD[T]` carries the element type at compile time (Scala/Java)
    - Python erases this at the JVM boundary

> [!NOTE]
> `DataFrame = Dataset[Row]` - a Dataset where the type parameter is an untyped `Row`
>
> Under the hood, a DataFrame/Dataset compiles down to an `RDD` of `InternalRow` for execution

- __When to use RDDs__ -
    - Custom partitioning
    - Custom binary/non-tabular formats
    - Fine-grained locality control
    - Legacy Hadoop `InputFormat`
    - Explicit control over caching and recomputation

- __Creating RDDs__ -
    | Source           | API                                 |
    | ---------------- | ----------------------------------- |
    | Local collection | `sc.parallelize`                    |
    | Files            | `sc.textFile`                       |
    | Hadoop formats   | `sc.hadoopFile`, `newAPIHadoopFile` |
    | Existing RDD     | Transformations                     |
    | DataFrame        | `df.rdd`                            |

> [!WARNING]
> `df.rdd` drops DataFrame-level optimizations including schema awareness, Catalyst and Tungsten/codegen benefits etc

## RDD Partitions

- Partition = atomic unit of parallelism
- One task processes one partition
- Partition count determines maximum task parallelism
- Each partition has -
    - Index ($0$-based)
    - Preferred locality
    - Dependency mapping to parent partitions

- Partition sizing -
    | Problem             | Symptom                                                  |
    | ------------------- | -------------------------------------------------------- |
    | Too few partitions  | OOM, poor parallelism, stragglers                        |
    | Too many partitions | Scheduler overhead, tiny files, driver metadata pressure |

> [!TIP]
> Good rule of thumb -
>   - Target roughly 128 MB - 256 MB per partition
>   - Use 2-4x total executor cores for parallelism headroom

### Partition Count Rules

- `sc.parallelize(seq, numSlices)` -
    - Defaults to `sc.defaultParallelism` (= total executor cores)
    - Override with `numSlices`
- `sc.textFile(path, minPartitions)` -
    - One partition per HDFS block (default $128 MB$)
    - `minPartitions` sets a floor
    - Spark may split further but never merges blocks
- __Shuffle output__ -
    - Controlled by `spark.sql.shuffle.partitions` (SQL/DataFrame) or `rdd.partitionBy(n)`
- `coalesce(n)` -
    - Reduces to `n` without shuffle - may still move data between executors, but it avoids a shuffle exchange
    - May produce skewed partition sizes if reduction is large
- `repartition(n)` -
    - Always full shuffle
    - Even distribution guaranteed

### Partition Locality

| Locality        | Meaning                   |
| --------------- | ------------------------- |
| `PROCESS_LOCAL` | Data in same executor JVM |
| `NODE_LOCAL`    | Data on same machine      |
| `RACK_LOCAL`    | Data on same rack         |
| `ANY`           | Anywhere                  |

- DAGScheduler computes preferred locality for each task
- TaskScheduler uses delay scheduling -
    - Waits briefly for better locality
    - Falls back if no local slot becomes free

## RDD Iterator Model & Compute Chain

- Each RDD implements `compute(partition, context): Iterator[T]`
- Transformations wrap parent iterators
- A task calls `iterator()` on the final RDD in the stage
- Execution is pull-based -
    - The terminal consumer (shuffle writer or action) pulls elements from the outermost iterator
    - Each `next()` call cascades down the chain through every transformation
- Records flow one-by-one through the pipeline -
    - Spark does not materialize intermediate collections
    - It builds one iterator chain and processes records in a streaming fashion
- Example -
    ```scala
    sc.textFile("hdfs://...")
        .filter(_.contains("ERROR"))
        .map(line => parse(line))
        .mapPartitions(iter => aggregate)
        .collect()

    // HadoopRDD
    // → FilteredRDD
    // → MappedRDD
    // → MapPartitionsRDD
    // → ShuffleWriter / Action
    ```

> [!NOTE]
> The iterator model is why Spark can process a $100 GB$ partition on an executor with $4 GB$ heap
>
> Memory pressure comes from aggregation (building hash tables for `groupBy`) and caching, not from the pipeline itself

## `map` vs `mapPartitions`

| API                | Granularity                 | Best Use                                   |
| ------------------ | --------------------------- | ------------------------------------------ |
| `map`              | Per record                  | Simple stateless transforms                |
| `mapPartitions`    | Per partition               | Expensive setup, batching, DB/client reuse |
| `foreach`          | Per record side effect      | Rarely preferred                           |
| `foreachPartition` | Partition-level side effect | External writes, batching                  |

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

## RDD Lineage & DAG

- Lineage = recipe to rebuild an RDD
- Stored as dependencies between RDDs
- Forms a DAG because RDDs are immutable
- Lineage enables fault tolerance without replication -
    - Lost partitions are recomputed from lineage
    - But long lineage can become expensive
    - Use -
        - `persist()` to avoid recomputation
        - `checkpoint()` to truncate lineage

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
    #  |    HadoopRDD[...] ...
    ```

## NarrowDependency vs ShuffleDependency

- __NarrowDependency__ -
    - Each child partition depends on a fixed, bounded set of parent partitions (usually exactly one)
    - No shuffle
    - Pipelines within a stage
    - Eg - `map`, `filter`, `flatMap`, `mapPartitions`, `union`
- __ShuffleDependency__ (wide dependency) -
    - Each output partition may depend on all input partitions
    - Forces shuffle
    - Creates stage boundary
    - Eg - `reduceByKey`, `groupByKey`, `join`, `distinct`, `repartition`

### Failure Recovery

| Dependency Type | Recovery Cost                                          |
| --------------- | ------------------------------------------------------ |
| Narrow          | Recompute only affected partition chain                |
| Shuffle         | May require re-running map stage if shuffle files lost |

> [!TIP]
> External Shuffle Service helps preserve shuffle files when executors die, avoiding unnecessary parent-stage recomputation

## Co-partitioning

- If two RDDs share the same `Partitioner` (same type, same `numPartitions`), a join between them is narrow - matching partitions are already co-located
- Python -
    ```python
    # Both use HashPartitioner(10) - join is narrow, no shuffle
    rdd1 = pairs.partitionBy(10)
    rdd2 = other_pairs.partitionBy(10)
    joined = rdd1.join(rdd2)                        # no shuffle - co-partitioned

    # other_pairs uses default partitioner - join triggers full shuffle
    joined = rdd1.join(other_pairs)                 # shuffle on other_pairs side
    ```

> [!NOTE]
> Cache the co-partitioned RDD after `partitionBy` - otherwise `partitionBy` itself shuffles on every action

## RDD Persistence

- Persistence materializes RDD partitions so future actions reuse them instead of replaying lineage
- `cache()` / `persist()` only mark the RDD -
    - Actual materialization happens on the first action that computes each partition
    - Subsequent reads hit the `BlockManager` instead of recomputing
- `cache()` vs `persist()` -
    - `cache()` - shorthand for `persist(StorageLevel.MEMORY_ONLY)`
    - `persist(storageLevel)` - explicit control over where and how partitions are stored
- `unpersist()` -
    - Instructs `BlockManagerMaster` to remove all cached blocks for this RDD from all executors
    - Asynchronous by default (`blocking=false`)
    - `unpersist(blocking=True)` - wait until eviction completes

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

### `BlockManager` & Eviction

- Cached RDD partitions are stored as blocks - 
    ```python
    RDDBlockId(rddId, partitionIndex)
    ```

- Executor-side -
    - `MemoryStore` - holds deserialized objects or serialized byte arrays in JVM heap (or off-heap)
    - `DiskStore` - writes serialized bytes to local disk under `spark.local.dir`
- Driver-side -
    - `BlockManagerMaster` tracks block locations

- Eviction is LRU-based -
    - If a cached block is dropped -
        - `MEMORY_ONLY` - recompute from lineage
        - `MEMORY_AND_DISK` - read from disk if spilled
        - Replicated level - fetch from another executor if available

> [!TIP]
> Kryo is 3-10× faster and produces 3-5× smaller output
>
> ```scala
> SparkSession.builder()
>   .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
>   .config("spark.kryo.registrationRequired", "false")
>   .getOrCreate()
> ```

### Checkpointing

- Checkpointing writes RDD data to reliable storage (like HDFS) and truncates lineage
    - The checkpointed RDD has no parents
    - Recomputation starts from the checkpoint file
    - Requires a checkpoint directory set via `sc.setCheckpointDir("hdfs://...")`    

> [!TIP]
> Always `persist()` before `checkpoint()` - otherwise Spark recomputes the RDD twice - once to checkpoint, once for the job.

## PairRDD

- PairRDD = `RDD[(K, V)]`
    - Unlocks key-based distributed operations through `PairRDDFunctions`
    - Scala implicit enrichment (`PairRDDFunctions`) activates it when the element type is a $2$-tuple

| Function         | Usage Example                               | Parameters                                                                                                                                                                                                                     | Special Notes                                                 |
| ---------------- | ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------- |
| `reduceByKey`    | `rdd.reduceByKey(_ + _)`                    | - `f` - merge function for values of same key<br>- `numPartitions` - output partition count                                                                                                                                    | - Uses map-side combine<br>- Preferred aggregation API        |
| `aggregateByKey` | `rdd.aggregateByKey((0,0))(seqOp, combOp)`  | - `zeroValue` - initial accumulator<br>- `seqOp` - merge value into accumulator<br>- `combOp` - merge accumulators across partitions<br>- `numPartitions` - output partitions                                                  | - Output type can differ from input type                      |
| `combineByKey`   | `rdd.combineByKey(create, mv, mc)`          | - `createCombiner` - create initial combiner<br>- `mergeValue` - merge value locally<br>- `mergeCombiners` - merge after shuffle<br>- `numPartitions` - output partitions<br>- `mapSideCombine` - enable local pre-aggregation | - Most general aggregation primitive                          |
| `groupByKey`     | `rdd.groupByKey()`                          | - `numPartitions` - output partitions                                                                                                                                                                                          | - Shuffles all values<br>- Expensive for large datasets       |
| `foldByKey`      | `rdd.foldByKey(0)(_ + _)`                   | - `zeroValue` - identity value<br>- `f` - aggregation function<br>- `numPartitions` - output partitions                                                                                                                        | - Similar to `reduceByKey` with identity                      |
| `countByKey`     | `rdd.countByKey()`                          | None                                                                                                                                                                                                                           | - Returns counts to Driver<br>- OOM risk on large cardinality |
| `join`           | `rdd1.join(rdd2)`                           | - `other` - RDD to join with<br>- `numPartitions` - output partitions                                                                                                                                                          | - Inner join<br>- Shuffle avoided if partitioners match       |
| `leftOuterJoin`  | `rdd1.leftOuterJoin(rdd2)`                  | - `other` - right-side RDD<br>- `numPartitions` - output partitions                                                                                                                                                            | - Keeps all left keys                                         |
| `rightOuterJoin` | `rdd1.rightOuterJoin(rdd2)`                 | - `other` - left-side RDD<br>- `numPartitions` - output partitions                                                                                                                                                             | - Keeps all right keys                                        |
| `fullOuterJoin`  | `rdd1.fullOuterJoin(rdd2)`                  | - `other` - RDD to join with<br>- `numPartitions` - output partitions                                                                                                                                                          | - Keeps all keys                                              |
| `cogroup`        | `rdd1.cogroup(rdd2)`                        | - `other` - one or more RDDs<br>- `numPartitions` - output partitions                                                                                                                                                          | - Basis of all RDD joins                                      |
| `subtractByKey`  | `rdd1.subtractByKey(rdd2)`                  | - `other` - RDD containing keys to remove<br>- `numPartitions` - output partitions                                                                                                                                             | - Removes matching keys                                       |
| `mapValues`      | `rdd.mapValues(_.toUpperCase)`              | - `f` - transform applied only to values                                                                                                                                                                                       | - Preserves partitioner                                       |
| `flatMapValues`  | `rdd.flatMapValues(_.split(","))`           | - `f` - returns multiple values per input                                                                                                                                                                                      | - Preserves partitioner                                       |
| `keys`           | `rdd.keys()`                                | None                                                                                                                                                                                                                           | - Returns only keys<br>- Loses partitioner                    |
| `values`         | `rdd.values()`                              | None                                                                                                                                                                                                                           | - Returns only values<br>- Loses partitioner                  |
| `sortByKey`      | `rdd.sortByKey()`                           | - `ascending` - sort order<br>- `numPartitions` - output partitions                                                                                                                                                            | - Uses `RangePartitioner`                                     |
| `partitionBy`    | `rdd.partitionBy(new HashPartitioner(100))` | - `partitioner` - key → partition strategy                                                                                                                                                                                     | - Full shuffle<br>- Enables co-partitioned joins              |
| `lookup`         | `rdd.lookup("user1")`                       | - `key` - key to search                                                                                                                                                                                                        | - Efficient only with partitioner                             |
| `sampleByKey`    | `rdd.sampleByKey(False, fractions)`         | - `withReplacement` - allow duplicates<br>- `fractions` - sample rate per key<br>- `seed` - random seed                                                                                                                        | - Stratified sampling                                         |
| `countByValue`   | `rdd.countByValue()`                        | None                                                                                                                                                                                                                           | - Counts distinct element frequencies                         |
| `collectAsMap`   | `rdd.collectAsMap()`                        | None                                                                                                                                                                                                                           | - Collects PairRDD into Driver map<br>- Small datasets only   |

- __Join performance__ -
    - A join avoids shuffle only when partitioners match
    - Otherwise -
        - One side may shuffle
        - Or both sides may shuffle
    - If one side is small - manual broadcast join may beat shuffle join

> [!TIP]
> RDD joins are built on top of `cogroup` + `flatMap`

> [!NOTE]
> `map` vs `mapValues` -
>   - `mapValues` is partitioner-preserving because the keys do not change
>   - `map` is not because Spark cannot prove key is unchanged

## CoGroupedRDD

- Internal RDD behind multi-RDD key operations - `join`, `cogroup`, `subtractByKey`, `intersection`
- `cogroup` groups values by key across RDDs -
    - Returns `RDD[(K, (Iterable[V1], Iterable[V2]))]`
    - Supports up to 4 RDDs

- __Execution Model__ -
    - For each parent RDD -
        - Matching partitioner - `NarrowDependency` (local read)
        - Different or missing partitioner - `ShuffleDependency`
    - Co-partitioned parents avoid shuffle entirely
    - Uses `ExternalAppendOnlyMap` internally -
        - Hashes keys
        - Buffers values per key
        - Spills to disk under memory pressure

- __Memory Characteristics__ -
    - Values for a key land in one reduce partition
    - Spark can spill, but hot keys still cause memory/disk pressure
    - `groupByKey` is expensive because it materializes full value lists
    - `reduceByKey` is safer because it incrementally merges values
    - Large joins can OOM due to value cross-product explosion

## ShuffledRDD

- Concrete RDD created by wide transformations that redistribute data across the network
- Has one parent RDD and one `ShuffleDependency`
- Created internally by operations like `reduceByKey`, `groupByKey`, `aggregateByKey`, `sortByKey`, `repartition`
- Key fields -
    - `prev` - parent RDD
    - `part` - output partitioner
    - `aggregator` - optional aggregation logic
    - `keyOrdering` - optional ordering for sorted shuffle output
    - `mapSideCombine` - whether pre-aggregation happens before shuffle write
- __Shuffle Write - Map Side__ -
    - Each map task -
        - Reads one input partition
        - Partition records by target reducer
        - Aggregate if possible
        - Spill if memory pressure
        - Write shuffle data + index file
- __Shuffle Read - Reduce Side__ -
    - Each reduce task -
        - Asks `MapOutputTracker` where map outputs are
        - Fetches its partition slice from every mapper
        - Uses `BlockManager` / Netty for remote fetches
        - Merges, sorts, or aggregates fetched records if needed
- __Shuffle Files__ -
    - `SortShuffleManager` usually writes -
        - One data file per mapper
        - One index file per mapper
    - Avoids `numMappers × numReducers` small-file explosion by older `HashShuffleManager`
    - With External Shuffle Service - these files are served by the node daemon, not the executor JVM
        - Executor can die and be relaunched without losing shuffle output

> [!TIP]
> Shuffle tuning -
>   - `spark.sql.shuffle.partitions` - output partition count (default $200$, almost always wrong; target $128 MB$ per partition)
>   - `spark.shuffle.compress` - compress shuffle files (default `true`; use `lz4` codec)
>   - `spark.shuffle.file.buffer` - map-side write buffer per output partition (default $32 KB$; increase to $1 MB$ for large shuffles)
>   - `spark.shuffle.sort.bypassMergeThreshold` - bypass writer threshold (default $200$; lower if file descriptor limit is hit)
>   - `spark.shuffle.io.maxRetries` - fetch retry count (default $3$; increase for flaky networks)
>   - `spark.reducer.maxSizeInFlight` - reduce-side fetch buffer (default $48 MB$; increase for fast networks)
>   - `spark.reducer.maxReqsInFlight` - max simultaneous fetch requests per reduce task (default `Int.MaxValue`)
>   - `spark.reducer.maxBlocksInFlightPerAddress` - max blocks fetched from one executor at a time (default `Int.MaxValue`)
>   - `spark.shuffle.io.retryWait` - wait between retries (default $5s$)

## UnionRDD

- Combines multiple parent RDDs into one logical RDD
- No shuffle, no data movement
- Uses narrow dependencies
- Partition count = sum of parent partition counts
- `sc.union(rdds)` or `rdd1.union(rdd2)` -
    - `sc.union` more efficient for many RDDs - creates one `UnionRDD` with `N` parents
    - `rdd1.union(rdd2).union(rdd3)` chains binary `UnionRDD`s - creates a deep lineage tree
- Example -
    ```python
    rdd1 = sc.parallelize([("a", 1)], 2)
    rdd2 = sc.parallelize([("b", 2)], 3)

    union = sc.union([rdd1, rdd2])

    union.getNumPartitions()   # 5
    union.partitioner          # None
    ```

- Characteristics -
    - Does not deduplicate
    - Does not rebalance
    - Can create too many partitions
    - Loses partitioner metadata

> [!NOTE]
> The partitioner loss is the most dangerous `UnionRDD` gotcha in production -
>   ```python
>   rdd1 = large_rdd.partitionBy(100)   # HashPartitioner(100)
>   rdd2 = other_rdd.partitionBy(100)   # HashPartitioner(100)
>
>   union = rdd1.union(rdd2)            
>
>   union.partitioner                   # partitioner = None
>   union.join(third_rdd)               # may trigger full shuffle
>   ```
>
> Fix -
>   ```
>   union = rdd1.union(rdd2).partitionBy(100).persist()
>   ```

## HadoopRDD

- Leaf RDD used to read data through Hadoop `InputFormat`
- Created by  `sc.textFile`, `sc.sequenceFile`, `sc.hadoopFile`, `sc.newAPIHadoopFile`
-  __Partitioning Model__ -
    - Each partition (`HadoopPartition`) wraps one Hadoop `InputSplit` -
        - `TextInputFormat` - one `InputSplit` per HDFS block (default $128 MB$)
        - `CombineTextInputFormat` - multiple blocks combined into one split
    - Preferred locations come from block replica locations - enables data-local scheduling
- __Execution Model__ -
    - Each task -
        - Opens one `RecordReader`
        - Streams records lazily
        - Closes reader on task completion/failure
        - Does not load the full split into memory
- Example -
    ```python
    rdd = sc.textFile("hdfs://path/data.txt", minPartitions=10)
    ```

    - `minPartitions` is a hint, not a hard guarantee
    - Spark usually cannot create fewer partitions than underlying HDFS blocks
    - To reduce partitions after read, use `coalesce`

### Small Files Problem

```
10,000 tiny files
  -> 10,000 input splits
  -> 10,000 partitions
  -> 10,000 tasks
  -> high scheduler overhead
```

- Solutions -
    - Compact files upstream before ingestion (preferred)
    - Use `CombineTextInputFormat` -
        - Combine multiple small files into one split
        - Set via `hadoopConfiguration`
    - `sc.wholeTextFiles` - 
        - Reads each file as one record `(filename, content)`
        - Reduces partition count but loads entire file into memory

### Reading files

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
