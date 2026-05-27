# Performance Tuning

## Parallelism Tuning

- __Parallelism__ - the number of tasks running concurrently; directly determines how well Spark utilizes cluster resources; wrong parallelism is the root cause of most Spark performance problems

### Input Parallelism

- `spark.sql.files.maxPartitionBytes` (default $128 MB$) - maximum bytes per input file partition
    - Decrease to increase parallelism for CPU-bound scans (eg - $32 MB$ for complex transformations)
    - Increase to reduce task count for I/O-bound scans with simple transformations
- `spark.sql.files.openCostInBytes` (default $4 MB$) - virtual cost to open a file; prevents over-splitting small files into many 1-record partitions
- `spark.default.parallelism` - default partition count for RDD operations; should equal $2-3×$ total executor cores
- Target - enough tasks to keep ALL executor cores busy; typically `totalCores × 2` to `totalCores × 4`

### Shuffle Parallelism

- `spark.sql.shuffle.partitions` (default $200$) - output partition count for ALL shuffle operations
    - $200$ is almost always wrong - calibrate per workload
    - Formula - `ceil(shuffleOutputBytes / targetPartitionSize)` where `targetPartitionSize = 128MB`
    - With AQE enabled - set high (eg - $1000$); AQE coalesces small partitions automatically
    - Without AQE - tune manually; too high = tiny tasks + scheduler overhead; too low = OOM + stragglers

### Partition Sizing Guidelines

- Target $128 MB$ per partition for most workloads
- Reduce to $32-64 MB$ for memory-intensive operations (many complex expressions, wide schemas)
- Increase to $256-512 MB$ for simple filter-and-write pipelines where task overhead dominates
- Never below $1 MB$ - scheduler overhead dominates; nothing computed
- Never above executor memory / (`spark.executor.cores` × $2$) - OOM risk per task

### Parallelism Antipatterns

- __Too many tiny partitions__ - $10000$ tasks processing $1 KB$ each; scheduler overhead dominates; $99\%$ of task time is startup/teardown
- __Too few large partitions__ - $4$ tasks on $200$ cores; $196$ cores idle; one skewed task stalls stage
- __Partition count not multiple of total cores__ - last wave of tasks uses partial core set; $201$ tasks on $200$ cores means one extra wave for $1$ task
- Fix - always target partition count = multiple of total executor cores; last wave fully utilized

---

## Memory Tuning

- Memory is the most critical Spark resource; wrong configuration causes OOM, excessive GC, and spill

### Memory Regions

```
JVM Heap (spark.executor.memory = 4g example)
├── Reserved: 300 MB (fixed; Spark internal objects)
├── User memory: (4096 - 300) × (1 - 0.6) = 1518 MB
│   └── UDFs, user data structures, Python overhead
└── Unified pool: (4096 - 300) × 0.6 = 2278 MB
├── Storage memory: 2278 × 0.5 = 1139 MB (initial; borrowable)
└── Execution memory: 2278 × 0.5 = 1139 MB (initial; borrowable)
```

### Memory Tuning Levers

- `spark.executor.memory` - total JVM heap per executor
- `spark.memory.fraction` (default $0.6$) - fraction of usable heap for unified pool
    - Increase to $0.7-0.8$ for shuffle/aggregation heavy jobs (more execution memory)
    - Decrease to $0.5$ for UDF-heavy jobs with large user data structures
- `spark.memory.storageFraction` (default $0.5$) - fraction of unified pool reserved for storage (protected from execution eviction)
    - Increase for cache-heavy iterative algorithms
    - Decrease for shuffle-heavy jobs with no caching
- `spark.executor.memoryOverhead` (default `max(executor_memory × 0.1, 384MB)`) - off-heap memory for JVM overhead (native libraries, Python workers, Netty buffers)
    - Increase for Python-heavy workloads (`spark.executor.memoryOverhead = executor_memory × 0.2`)
    - Increase for jobs with many Netty connections (large shuffle, many executors)

### Memory Pressure Symptoms

- __Execution spill__ - `Spill (memory)` metric in Spark UI > 0; increase `spark.executor.memory` or reduce partition size
- __Storage eviction__ - cached RDDs evicted; `BlockManager` logs show eviction; increase `spark.memory.storageFraction`
- __Driver OOM__ - `collect()` on large dataset; result exceeds `spark.driver.maxResultSize`; use `write` instead
- __Container OOM__ - YARN/K8s kills executor container; total process memory exceeds container limit; increase `spark.executor.memoryOverhead`

### Heap Sizing Formula

Total executor container memory needed:
= spark.executor.memory

spark.executor.memoryOverhead
spark.memory.offHeap.size (if enabled)

Container limit must be > total
YARN: spark.yarn.executor.memoryOverhead auto-added
K8s:  spark.executor.memoryOverhead explicitly set

---

## GC Tuning (G1GC, ZGC)

- GC pauses are the most common cause of Spark job stalls; long GC pauses cause task timeouts, executor heartbeat failures, and cascade failures

### Why Spark Causes GC Pressure

- Short-lived objects - task execution creates millions of temporary objects (iterators, row wrappers, expression results)
- Long-lived objects - cached RDD partitions, broadcast variables, large hash maps accumulate in old gen
- Mixed lifetimes - execution objects promoted to old gen before becoming garbage → old gen fills → full GC

### G1GC (Default since Java 9)

- __G1GC (Garbage First)__ - region-based collector; splits heap into equal-sized regions; collects regions with most garbage first
- Suitable for Spark when heap $\leq 32 GB$; pause targets via `-XX:MaxGCPauseMillis`

### G1GC Tuning for Spark

```bash
--conf spark.executor.extraJavaOptions="\
  -XX:+UseG1GC \
  -XX:G1HeapRegionSize=16m \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:MaxGCPauseMillis=500 \
  -XX:G1NewSizePercent=10 \
  -XX:G1MaxNewSizePercent=40 \
  -XX:+ParallelRefProcEnabled \
  -verbose:gc \
  -XX:+PrintGCDetails \
  -XX:+PrintGCDateStamps"
```

- `G1HeapRegionSize=16m` - larger regions reduce region count; fewer metadata operations; important for large heaps
- `InitiatingHeapOccupancyPercent=35` - start concurrent GC earlier (default $45$); reduces full GC frequency
- `MaxGCPauseMillis=500` - G1 targets this pause; may not achieve it; sets scheduling intent
- `G1NewSizePercent=10` - minimum young gen size; prevents over-sizing young gen starving old gen
- `ParallelRefProcEnabled` - parallelize reference processing; reduces GC pause for jobs with many weak references

### ZGC (Java 11+)

- __ZGC (Z Garbage Collector)__ - concurrent, low-latency collector; pause times $< 10 ms$ regardless of heap size
- All GC work done concurrently with application; stop-the-world only for initial/final safepoints
- Ideal for Spark executors with large heaps ($> 32 GB$) or latency-sensitive streaming

```bash
--conf spark.executor.extraJavaOptions="\
  -XX:+UseZGC \
  -XX:ZAllocationSpikeTolerance=5 \
  -XX:ConcGCThreads=4"
```

- `ZAllocationSpikeTolerance=5` - allows ZGC to handle allocation spikes (Spark tasks allocate rapidly during shuffle)
- `ConcGCThreads` - concurrent GC thread count; $2-4$ for $16$-core executors

### Shenandoah GC

- __Shenandoah__ - another ultra-low pause GC (OpenJDK); concurrent evacuation; $< 10 ms$ pauses
- Similar benefits to ZGC; available on OpenJDK builds

```bash
-XX:+UseShenandoahGC -XX:ShenandoahGCMode=iu
```

### GC Tuning Diagnosis

- Enable GC logging and parse with GCEasy or GCViewer
- High GC time in Spark UI task metrics → GC pressure
- `Full GC` entries in logs → old gen pressure; likely execution memory objects promoted prematurely
- Fix strategies -
    - Reduce per-task data size (more partitions)
    - Use `MEMORY_ONLY_SER` storage level (serialized blocks = fewer GC-visible objects)
    - Increase `spark.memory.fraction` (more memory for Spark; less for GC overhead space)
    - Switch to off-heap execution (`spark.memory.offHeap.enabled=true`)

---

## Executor Sizing Strategies (Few Large vs Many Small)

- __Executor sizing__ - choosing how many cores and how much memory per executor; one of the most impactful configuration decisions

### Few Large Executors

- Eg - $5$ executors × $16$ cores × $32 GB$ memory on a $10$-node cluster
- __Pros__ -
    - More cores per executor → better data locality (more partitions processed locally)
    - Less serialization for broadcast (fewer copies needed per host)
    - `mapPartitions` connections (DB, ML model) amortized over more tasks
- __Cons__ -
    - Large JVM heap → longer GC pauses (G1GC degrades above $32 GB$; use ZGC)
    - More tasks compete for same executor memory → more spill if one task uses more than expected
    - Single executor failure loses more work

### Many Small Executors

- Eg - $40$ executors × $2$ cores × $4 GB$ memory on same $10$-node cluster
- __Pros__ -
    - Smaller JVM heap → shorter GC pauses
    - Fault isolation - one executor failure loses less work
    - Better for jobs with variable task memory requirements
- __Cons__ -
    - More executor overhead (JVM startup, heartbeat traffic)
    - More broadcast copies (one per executor)
    - Worse for data locality (fewer co-located tasks)

### Recommended Approach

- Sweet spot - $4-5$ cores per executor with $4-8 GB$ per core
- Rule of thumb -
    - `spark.executor.cores = 4` or `5`
    - `spark.executor.memory = spark.executor.cores × memoryPerCore` where `memoryPerCore = 4-6 GB`
    - Leave $1$ core per node for OS + HDFS DataNode + YARN NodeManager overhead
    - Leave $1-2 GB$ per node for OS overhead

### Example Sizing Calculation

Cluster: 10 nodes × 16 cores × 64 GB RAM
Per node:

Reserve 1 core for OS/HDFS/YARN: 15 usable cores
Reserve 1 GB for OS: 63 GB usable
Executor cores: 5
Executors per node: 15 / 5 = 3
Memory per executor: 63 / 3 = 21 GB
spark.executor.memory = 19 GB (leave 2 GB for memoryOverhead)
spark.executor.memoryOverhead = 2 GB

Total cluster:

Executors: 3 × 10 = 30 executors
Total cores: 30 × 5 = 150 cores
spark.sql.shuffle.partitions = 300-450 (2-3× total cores)

---

## Kryo Tuning

- Kryo misconfiguration causes `KryoException: Buffer overflow` errors and suboptimal serialization performance

### Kryo Buffer Sizing

- `spark.kryoserializer.buffer` (default $64 KB$) - initial Kryo output buffer per serializer instance
    - If objects being serialized are typically smaller than this - leave at default
    - If objects are larger - Kryo auto-grows up to `buffer.max`; frequent resizing adds overhead
    - Set to $P75$ of your typical serialized object size

- `spark.kryoserializer.buffer.max` (default $64 MB$) - maximum Kryo buffer per serializer instance
    - `KryoException: Buffer overflow` → increase this
    - Must be large enough to serialize your largest single object
    - For large broadcast objects or wide rows: set to $256 MB$ - $1 GB$

```python
spark = SparkSession.builder \
    .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
    .config("spark.kryoserializer.buffer", "512k") \
    .config("spark.kryoserializer.buffer.max", "256m") \
    .config("spark.kryo.registrationRequired", "false") \
    .getOrCreate()
```

### Kryo Registration for Maximum Performance

```scala
class ProductionKryoRegistrator extends KryoRegistrator {
    override def registerClasses(kryo: Kryo): Unit = {
        // Register all domain classes - alphabetical for documentation
        kryo.register(classOf[EventRecord], 100)
        kryo.register(classOf[UserProfile], 101)
        kryo.register(classOf[Array[EventRecord]], 102)
        kryo.register(classOf[Array[UserProfile]], 103)

        // Register common Scala collections used in RDDs
        kryo.register(classOf[scala.collection.mutable.HashMap[_, _]], 200)
        kryo.register(classOf[scala.collection.immutable.Map[_, _]], 201)

        // Use custom serializer for complex types
        kryo.register(classOf[BloomFilter], new BloomFilterSerializer, 300)
    }
}
```

### Kryo Performance Debugging

```python
# Check if Kryo is actually being used
print(spark.conf.get("spark.serializer"))
# Should print: org.apache.spark.serializer.KryoSerializer

# Check for unregistered class warnings in logs
# Look for: "Class is not registered: com.example.MyClass"
# Set registrationRequired=true to force registration discipline
spark.conf.set("spark.kryo.registrationRequired", "true")
```

---

## Broadcast Threshold Tuning

- Broadcast join threshold controls when Spark automatically broadcasts a table; wrong threshold = OOM or missed optimization

### Threshold Configuration

- `spark.sql.autoBroadcastJoinThreshold` (default $10 MB$) - tables estimated smaller than this are broadcast
    - Increase to $50-200 MB$ when -
        - Executors have large memory and can hold bigger broadcast tables
        - Dimension tables frequently joined are between $10-100 MB$
        - SMJ overhead (two shuffles) exceeds broadcast cost
    - Set to $-1$ to disable automatic broadcast completely when -
        - Stats are stale and auto-broadcast causes OOM
        - Explicitly controlling all join strategies

```python
# Tune per-query (session-level)
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 50 * 1024 * 1024)  # 50 MB

# Force broadcast for known small table
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "key")

# Disable auto-broadcast globally for safety
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
```

### Broadcast OOM Diagnosis

- Symptom - executor OOM with `BroadcastHashJoinExec` or `BroadcastExchangeExec` in stack trace
- Cause - table estimated small (stale stats) but actually large; broadcast oversizes executor memory
- Fix -
    1. `ANALYZE TABLE` to refresh statistics
    2. Lower `autoBroadcastJoinThreshold`
    3. Use `MERGE` hint to force SMJ for specific queries
    4. Increase executor memory if broadcast genuinely needed

### AQE and Broadcast

- AQE `DynamicJoinSelection` converts SMJ → BHJ at runtime when actual shuffle output < threshold
- Works even when static planning chose SMJ due to stale/missing stats
- `spark.sql.adaptive.autoBroadcastJoinThreshold` (default = `spark.sql.autoBroadcastJoinThreshold`) - AQE-specific threshold; can differ from static threshold

---

## Small File Problem & Compaction

- __Small file problem__ - thousands of tiny files in HDFS/S3 cause -
    - HDFS NameNode metadata pressure ($1$ object per file in NameNode heap)
    - Slow directory listings ($O(n)$ per partition predicate)
    - Excessive task overhead (one task per file regardless of size)
    - Poor compression efficiency (each file compressed independently; less data to compress)

### Root Causes

- One output file per task × many partitions × many micro-batch writes = exponential small file growth
- `bucketBy` with high bucket count without enough data per bucket
- Streaming writes with frequent triggers producing small batches
- `partitionBy` with high-cardinality columns (one file per value per task)

### Compaction Strategies

- __`coalesce` before write__ - reduces output files without shuffle; may produce uneven files -
```python
    df.coalesce(10).write.mode("append").parquet(path)
```

- __`repartition` before write__ - full shuffle; even file sizes -
```python
    df.repartition(10).write.mode("append").parquet(path)
```

- __`maxRecordsPerFile`__ - limits records per output file; Spark creates new files when limit reached -
```python
    df.write.option("maxRecordsPerFile", 1_000_000).parquet(path)
```

- __Scheduled compaction job__ - separate job reads small files, repartitions, overwrites -
```python
    # Compaction job - run daily or after N micro-batches
    spark.read.parquet("hdfs://path/date=2024-01-01/") \
         .repartition(20) \
         .write.mode("overwrite").parquet("hdfs://path/date=2024-01-01/")
```

- __Delta Lake `OPTIMIZE`__ - automated compaction with bin-packing into target file sizes -
```sql
    OPTIMIZE events WHERE date = '2024-01-01'
```

### Small File Detection

```python
# Check file size distribution for a path
import subprocess
result = subprocess.run(
    ["hdfs", "dfs", "-ls", "hdfs://path/"],
    capture_output=True, text=True
)
# Parse file sizes; count files below 64 MB threshold

# Or use Spark
df = spark.read.parquet("hdfs://path/")
print(f"Partitions: {df.rdd.getNumPartitions()}")
df.groupBy(spark_partition_id()).count().describe("count").show()
```

---

## Predicate & Partition Pruning Tuning

- Pruning effectiveness directly determines how much data Spark reads; wrong pruning = full scan on petabyte tables

### Ensuring Predicate Pushdown

```python
# Verify predicate pushed to source
df.filter(col("date") == "2024-01-01").explain("extended")
# Look for filter inside BatchScanExec/FileSourceScanExec (pushed)
# vs FilterExec above scan (not pushed)

# Common reasons pushdown fails:
# 1. UDF in filter condition
# 2. Non-deterministic expression (rand(), uuid())
# 3. Complex expression source cannot handle

# Fix: rewrite filter to simple form
# BAD: .filter(my_udf(col("date")) == "2024-01-01")
# GOOD: .filter(col("date") == "2024-01-01")  # then apply UDF after filter
```

### Partition Pruning Optimization

- Hive-style partitioned tables - ensure filter uses exact partition column names and compatible types
- Type mismatches prevent pruning -
```python
    # BAD: partition column is IntegerType; filtering with string
    df.filter(col("year") == "2024")    # may not prune (implicit cast)

    # GOOD: matching types
    df.filter(col("year") == 2024)      # IntegerType literal; pruning works
```

- Metastore partition pruning vs directory listing -
```python
    # Enable metastore-based pruning (faster for many partitions)
    spark.conf.set("spark.sql.hive.metastorePartitionPruning", "true")
```

### Dynamic Partition Pruning

```python
# Enable DPP (default true in Spark 3+)
spark.conf.set("spark.sql.optimizer.dynamicPartitionPruning.enabled", "true")

# DPP requires:
# 1. Fact table partitioned on the join column
# 2. Dimension table has selective filter
# 3. Join condition on partition column
fact.join(dim.filter(col("year") == 2024), fact.date_id == dim.id)
# Fact table scans only partitions matching 2024 dates
```

---

## Join Strategy Tuning

### Strategy Selection Logic

```python
# Check which join strategy was chosen
df1.join(df2, "key").explain()
# Look for: BroadcastHashJoin, SortMergeJoin, ShuffledHashJoin

# Force specific strategy
df1.hint("broadcast").join(df2, "key")           # BroadcastHashJoin
df1.hint("merge").join(df2, "key")               # SortMergeJoin
df1.hint("shuffle_hash").join(df2, "key")        # ShuffledHashJoin
df1.hint("shuffle_replicate_nl").join(df2, "key") # BroadcastNestedLoopJoin
```

### SMJ Tuning

- SMJ requires two shuffles + two sorts; dominant cost for large-large joins
- Pre-partitioning eliminates shuffles -
```python
    # Pre-partition both tables on join key; cache
    df1_p = df1.repartition(200, "user_id").cache()
    df2_p = df2.repartition(200, "user_id").cache()
    df1_p.count()   # force cache
    df2_p.count()
    # Subsequent join is shuffle-free
    result = df1_p.join(df2_p, "user_id")
```

- Pre-sorting eliminates sort step too -
```python
    df1_ps = df1.repartition(200, "user_id") \
                .sortWithinPartitions("user_id").cache()
    df2_ps = df2.repartition(200, "user_id") \
                .sortWithinPartitions("user_id").cache()
```

### BHJ Sizing

- Broadcast table held in executor memory as `HashedRelation`
- Memory required per executor - `broadcastSize × 2-3` (raw data + hash map overhead + deserialized objects)
- $50 MB$ broadcast table → $150-200 MB$ per executor memory consumption
- With $200$ executors - $50 MB$ broadcast = $50 MB × 1 copy per executor × 200 = 10 GB$ total cluster memory for broadcast
- Tune `spark.sql.autoBroadcastJoinThreshold` relative to `spark.executor.memory`

---

## Skew Tuning

### AQE Skew Parameters

```python
# Enable AQE skew handling (default true)
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Tune skew detection sensitivity
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")       # default 5
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes",
               str(256 * 1024 * 1024))  # default 256 MB

# Lower factor to catch moderate skew
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "3")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes",
               str(64 * 1024 * 1024))   # 64 MB threshold
```

### Manual Salting (When AQE Insufficient)

```python
from pyspark.sql.functions import col, concat, lit, rand, explode, array

n_salt = 20

# Salt large (skewed) side
left_salted = skewed_df.withColumn(
    "salt_key",
    concat(col("join_key").cast("string"), lit("_"),
           (rand() * n_salt).cast("int").cast("string"))
)

# Replicate small side
right_replicated = small_df \
    .withColumn("salt_arr", array([lit(i) for i in range(n_salt)])) \
    .withColumn("salt", explode(col("salt_arr"))) \
    .withColumn("salt_key",
        concat(col("join_key").cast("string"), lit("_"),
               col("salt").cast("string"))) \
    .drop("salt_arr", "salt")

result = left_salted.join(right_replicated, "salt_key").drop("salt_key")
```

### Null Key Skew

```python
# All nulls go to partition 0 via HashPartitioner
# Handle nulls separately
non_null = df.filter(col("key").isNotNull())
null_rows = df.filter(col("key").isNull())

result = non_null.join(other, "key") \
                 .union(null_rows.withColumn("other_col", lit(None)))
```

---

## AQE Tuning

### Core AQE Configuration

```python
# Enable AQE (default true in Spark 3.2+)
spark.conf.set("spark.sql.adaptive.enabled", "true")

# Partition coalescing
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "128m")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionSize", "1m")
spark.conf.set("spark.sql.adaptive.coalescePartitions.minPartitionNum",
               str(spark.sparkContext.defaultParallelism))

# Dynamic join switching
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", str(50 * 1024 * 1024))  # 50 MB

# Skew handling
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes",
               str(256 * 1024 * 1024))
```

### AQE + High Shuffle Partitions

```python
# Set high shuffle partitions; AQE coalesces down
spark.conf.set("spark.sql.shuffle.partitions", "2000")
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.advisoryPartitionSizeInBytes", "128m")
# AQE will merge partitions to ~128 MB each
# Effective partition count = total_data_bytes / 128 MB
```

### Diagnosing AQE Behavior

```python
# See AQE plan changes
df.explain("formatted")
# Look for: AQEShuffleRead (coalescing), BroadcastHashJoin (converted from SMJ)

# Check actual partition count after AQE
result = df.groupBy("key").count()
result.write.parquet("/tmp/output")
# Check Spark UI - SQL tab - AQEShuffleRead shows merged partition groups
```

---

## Shuffle Tuning

- Shuffle is the most expensive operation; tuning it has outsized impact on job performance

### Partition Count Tuning

```python
# Calculate optimal shuffle partitions
# totalShuffleBytes estimated from Spark UI → Stage → shuffle write size
total_shuffle_bytes = 500 * 1024 * 1024 * 1024   # 500 GB example
target_partition_bytes = 128 * 1024 * 1024         # 128 MB

optimal_partitions = total_shuffle_bytes // target_partition_bytes  # ~4000
spark.conf.set("spark.sql.shuffle.partitions", str(optimal_partitions))
```

### Shuffle Read/Write Buffers

```python
# Map-side write buffer per output partition
spark.conf.set("spark.shuffle.file.buffer", "1m")       # default 32 KB
# Increase to reduce flush frequency for large shuffles; uses more heap

# Reduce-side fetch buffer
spark.conf.set("spark.reducer.maxSizeInFlight", "96m")   # default 48 MB
# Increase for faster networks (10+ Gbps); more data in-flight per reducer

# Max blocks per fetch request per executor
spark.conf.set("spark.reducer.maxBlocksInFlightPerAddress", "64")  # default unlimited
# Limit to prevent overwhelming a single executor serving many shuffle blocks
```

### Shuffle Compression

```python
# Shuffle file compression
spark.conf.set("spark.shuffle.compress", "true")        # default true
spark.conf.set("spark.io.compression.codec", "lz4")    # lz4 (fast) vs zstd (better ratio)
spark.conf.set("spark.shuffle.spill.compress", "true")  # default true

# For already-compressed data (Parquet sources) - disabling saves CPU
spark.conf.set("spark.shuffle.compress", "false")       # when shuffling compressed data
```

### SortShuffleManager Tuning

```python
# Bypass merge threshold - use simpler writer for small partition counts
spark.conf.set("spark.shuffle.sort.bypassMergeThreshold", "400")  # default 200
# Increase when num shuffle partitions < this AND no map-side aggregation
# BypassMergeSortShuffleWriter (no sort, multiple files then merge) used instead

# Shuffle I/O retry
spark.conf.set("spark.shuffle.io.maxRetries", "10")        # default 3; increase for flaky networks
spark.conf.set("spark.shuffle.io.retryWait", "10s")        # default 5s; backoff between retries
spark.conf.set("spark.shuffle.io.connectionTimeout", "120s")  # default; increase for slow networks
```

### External Shuffle Service

```python
# Enable for safe dynamic allocation
spark.conf.set("spark.shuffle.service.enabled", "true")
spark.conf.set("spark.dynamicAllocation.enabled", "true")
spark.conf.set("spark.shuffle.service.port", "7337")        # default port

# On YARN - also enable in NodeManager auxiliary service config
# spark.shuffle.service.enabled must be true in nodemanager
```

---

## Off-Heap Tuning

- Off-heap memory eliminates GC pressure for execution memory at the cost of explicit management

### Off-Heap Configuration

```python
spark = SparkSession.builder \
    .config("spark.memory.offHeap.enabled", "true") \
    .config("spark.memory.offHeap.size", str(4 * 1024 * 1024 * 1024)) \  # 4 GB per executor
    .getOrCreate()
```

### Off-Heap Sizing

Total executor memory:
= spark.executor.memory (JVM heap)

spark.executor.memoryOverhead (JVM native overhead)
spark.memory.offHeap.size (Tungsten off-heap)

Container must accommodate ALL three
Example:
spark.executor.memory = 8g
spark.executor.memoryOverhead = 1g
spark.memory.offHeap.size = 4g
Container limit = 8 + 1 + 4 = 13 GB minimum

### When Off-Heap Helps Most

- Large executor heaps ($> 16 GB$) where GC overhead becomes significant
- Shuffle-heavy jobs where execution buffers are large and long-lived
- Jobs with many concurrent tasks per executor (GC from multiple tasks compounds)
- Wide DataFrames with complex aggregations (large sort/hash buffers)

### Off-Heap Diagnosis

```python
# Check if off-heap being used
df.explain("extended")
# Physical operators using off-heap will show Tungsten off-heap mode

# Monitor in Spark UI
# Executor tab → Storage Memory used → high off-heap usage visible
```

---

## Network Tuning

- Network is often the bottleneck for shuffle-heavy jobs; tuning connection management and transfer parameters matters

### Timeout Configuration

```python
# Master network timeout - overrides all component-specific timeouts if not set
spark.conf.set("spark.network.timeout", "300s")         # default 120s
# Must be > longest expected GC pause; ZGC/Shenandoah: 120s sufficient; G1GC on large heap: 300s

# RPC-specific timeouts
spark.conf.set("spark.rpc.askTimeout", "300s")           # default 120s
spark.conf.set("spark.rpc.lookupTimeout", "120s")

# Executor heartbeat interval
spark.conf.set("spark.executor.heartbeatInterval", "20s")  # default 10s
# Must be < spark.network.timeout / 4
```

### Shuffle Network Parameters

```python
# Connection establishment timeout
spark.conf.set("spark.shuffle.io.connectionTimeout", "120s")

# Number of parallel connections per shuffle client
spark.conf.set("spark.shuffle.io.numConnectionsPerPeer", "1")   # default 1
# Increase to 2-3 for high-latency networks with high bandwidth

# Max chunk size for shuffle fetch
spark.conf.set("spark.shuffle.io.maxRetries", "10")
spark.conf.set("spark.shuffle.io.retryWait", "30s")

# Netty thread count for shuffle server
spark.conf.set("spark.shuffle.io.serverThreads", "8")    # default numCores/4
spark.conf.set("spark.shuffle.io.clientThreads", "8")
```

### Driver-Executor Network

```python
# Block manager port (shuffle serving)
spark.conf.set("spark.blockManager.port", "7078")

# Driver bind address (important for multi-homed nodes)
spark.conf.set("spark.driver.bindAddress", "0.0.0.0")
spark.conf.set("spark.driver.host", "driver-hostname")

# Max RPC message size (task serialization, MapStatus)
spark.conf.set("spark.rpc.message.maxSize", "256")   # default 128 MB; in MB not bytes
```

### S3/Object Store Tuning

```python
# For jobs reading from S3 - connection and thread tuning
spark.conf.set("spark.hadoop.fs.s3a.connection.maximum", "200")   # default 200
spark.conf.set("spark.hadoop.fs.s3a.threads.max", "64")
spark.conf.set("spark.hadoop.fs.s3a.connection.timeout", "200000")  # ms
spark.conf.set("spark.hadoop.fs.s3a.attempts.maximum", "10")

# Committer for S3 (consistent writes)
spark.conf.set("spark.hadoop.mapreduce.outputcommitter.factory.scheme.s3a",
               "org.apache.hadoop.fs.s3a.commit.S3ACommitterFactory")
spark.conf.set("spark.hadoop.fs.s3a.committer.name", "magic")  # or "staging"
```

---

## Resource Profiles (Spark 3.1+)

- __Resource Profiles__ - allow different stages within the same application to use different executor resource configurations; enables right-sizing per stage without running separate jobs

### Use Cases

- ETL pipeline with heterogeneous stages -
    - Stage 1 (file scan + filter) - CPU-bound; needs many cores; little memory
    - Stage 2 (complex ML feature engineering) - memory-bound; needs large memory per core
    - Stage 3 (model inference) - GPU-bound; needs GPU resources

### Resource Profile API

- Python -
```python
    from pyspark.resource import ResourceProfileBuilder, TaskResourceRequests, ExecutorResourceRequests

    # Default profile (used by most RDDs)
    # spark.executor.cores=4, spark.executor.memory=8g

    # Custom profile for memory-intensive stage
    rp_builder = ResourceProfileBuilder()

    # Executor-level resources
    exec_req = ExecutorResourceRequests() \
        .cores(2) \
        .memory("16g") \
        .memoryOverhead("2g") \
        .offHeapMemory("4g")

    # Task-level resources
    task_req = TaskResourceRequests().cpus(1)

    rp = rp_builder.require(exec_req).require(task_req).build

    # Apply profile to specific RDD
    rdd_with_profile = base_rdd.withResources(rp)

    # Or via DataFrame - requires converting to RDD with profile
```

- Scala -
```scala
    import org.apache.spark.resource._

    val rp = new ResourceProfileBuilder()
        .require(new ExecutorResourceRequests()
            .cores(2)
            .memory("16g")
            .memoryOverhead("2g"))
        .require(new TaskResourceRequests().cpus(1))
        .build

    val rddWithProfile = baseRDD.withResources(rp)
```

### Resource Profile Limitations

- Only supported on YARN and Kubernetes (not Standalone)
- Dynamic allocation respects resource profiles - different executor pools for different profiles
- Not yet fully supported for DataFrame/SQL stages; primarily RDD-level

---

## Task-Level Resource Assignments (GPUs)

- __Task resources__ - custom resources (GPUs, FPGAs, custom accelerators) assigned to individual tasks via Spark's resource management framework

### GPU Configuration

```python
# Configure executor GPU resources
spark = SparkSession.builder \
    .config("spark.executor.resource.gpu.amount", "1") \           # GPUs per executor
    .config("spark.executor.resource.gpu.discoveryScript",
            "/path/to/getGpusResources.sh") \                     # script to discover GPUs
    .config("spark.task.resource.gpu.amount", "0.5") \            # GPU fraction per task
    .getOrCreate()                                                  # 0.5 = 2 tasks share 1 GPU
```

### GPU Discovery Script

```bash
#!/bin/bash
# getGpusResources.sh - returns JSON of available GPU addresses
# Required format: {"name": "gpu", "addresses": ["0", "1"]}
nvidia-smi --query-gpu=index --format=csv,noheader | \
    jq -R . | jq -s '{"name": "gpu", "addresses": .}'
```

### Using GPU in Tasks

- Scala -
```scala
    import org.apache.spark.TaskContext

    rdd.mapPartitions { iter =>
        // Get GPU assigned to this task
        val gpuAddresses = TaskContext.get().resources()("gpu").addresses
        val gpuId = gpuAddresses(0)   // eg "0" or "1"

        // Initialize GPU context for this task
        val model = loadModelOnGpu(gpuId.toInt)
        iter.map(record => model.predict(record))
    }
```

### GPU Fraction Sharing

- `spark.task.resource.gpu.amount = 0.5` - two tasks share one GPU
- Spark assigns GPU addresses to tasks; tasks run concurrently sharing the GPU
- Works for inference workloads where GPU is not saturated by one task
- For training - set `spark.task.resource.gpu.amount = 1.0` (exclusive GPU per task)

### GPU + Resource Profiles

```scala
val gpuRp = new ResourceProfileBuilder()
    .require(new ExecutorResourceRequests()
        .cores(4)
        .memory("16g")
        .resource("gpu", 2, "/path/to/discovery.sh"))   // 2 GPUs per executor
    .require(new TaskResourceRequests()
        .cpus(1)
        .resource("gpu", 1))                             // 1 GPU per task
    .build

val gpuRdd = featureRdd.withResources(gpuRp)
```

> [!NOTE]
> The single highest-impact performance tuning action for most Spark jobs:
> 1. Enable AQE (`spark.sql.adaptive.enabled=true`) - handles partition coalescing, skew, join selection automatically
> 2. Configure Kryo (`spark.serializer=org.apache.spark.serializer.KryoSerializer`) - $3-10×$ faster RDD serialization
> 3. Size `spark.sql.shuffle.partitions` correctly (or let AQE handle it with high base value)
> 4. Set `spark.executor.memory` and `spark.executor.cores` using the $4-5$ cores per executor guideline
>
> Everything else is secondary optimization. Start with these four; measure; then tune further.

> [!TIP]
> Performance tuning workflow -
> ```
> 1. Profile first - identify actual bottleneck via Spark UI
>    - High GC time → GC tuning, reduce per-task data, serialized storage levels
>    - High shuffle read/write → shuffle partition tuning, co-partitioning
>    - Stage stall (1 task remaining) → skew tuning, AQE skew join
>    - Low CPU utilization → parallelism tuning (too few partitions)
>    - Spill metrics > 0 → memory tuning (increase executor memory or reduce partition size)
>
> 2. Change one thing at a time - Spark configs interact; changing multiple simultaneously
>    makes it impossible to attribute improvement
>
> 3. Use representative data - tuning on 1% sample then running on full data fails
>    because partition count, skew, and spill behavior all change with data volume
> ```

