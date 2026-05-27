# Partitioning & Data Layout

## Partitioning Overview

- __Partition__ - the atomic unit of parallelism in Spark; a contiguous slice of data processed by exactly one task on one executor
- Partitioning decisions affect - parallelism, data locality, shuffle volume, task skew, output file layout, and query pruning at read time
- Three distinct partitioning contexts -
    - __Input partitioning__ - how source data is split into partitions at read time
    - __Shuffle partitioning__ - how data is redistributed across partitions between stages
    - __Output partitioning__ - how data is physically laid out on disk at write time

### Partitioning Taxonomy

| Context | Controlled By | Key Config |
| --- | --- | --- |
| Input | Source splits, `minPartitions` | `spark.sql.files.maxPartitionBytes` |
| Shuffle | `HashPartitioner`, `RangePartitioner` | `spark.sql.shuffle.partitions` |
| Output file layout | `partitionBy`, `bucketBy` | Writer options |
| In-memory repartition | `repartition`, `coalesce` | Explicit in code |

---

## Input Partitioning

- __Input partitioning__ - how Spark splits source data into partitions when reading; determines initial parallelism and data locality

### File-Based Sources (Parquet, ORC, CSV, JSON)

- Spark uses `FilePartition` - groups of `PartitionedFile` objects; each file partition processed by one task
- __`spark.sql.files.maxPartitionBytes`__ (default $128 MB$) - maximum bytes per input partition
    - Files larger than this are split at block boundaries
    - Multiple small files combined into one partition if total < threshold
- __`spark.sql.files.openCostInBytes`__ (default $4 MB$) - estimated cost to open a file; added to file size when computing partition assignments
    - Prevents many tiny files from being over-combined; treats each file as at least $4 MB$ for packing purposes
- __`spark.sql.files.minPartitionNum`__ (default `spark.default.parallelism`) - minimum partition count for file scans
- File splitting rules -
    - Splittable formats (Parquet, ORC, uncompressed CSV) - split at block boundaries
    - Non-splittable formats (gzip-compressed CSV, JSON) - one partition per file regardless of size; gzip cannot be split mid-stream
    - Use `bzip2` or `zstd` for splittable compression on text formats

### HDFS Block Locality

- HDFS stores each block on $3$ replica nodes
- `FileInputFormat.getSplits` returns block locations per split
- `DAGScheduler` uses these as preferred locations → tasks scheduled on nodes holding the data → `NODE_LOCAL` or `PROCESS_LOCAL` execution

### JDBC Partitioning

- Single-partition by default (all data through one connection) - never acceptable for large tables
- Parallel read requires specifying partition column and bounds -
```python
    df = spark.read.jdbc(
        url=url,
        table="orders",
        column="order_id",          # partition column (numeric, date, or timestamp)
        lowerBound=1,               # inclusive lower bound
        upperBound=10000000,        # inclusive upper bound
        numPartitions=100,          # creates 100 equally-sized ranges
        properties=props
    )
    # Creates 100 queries: WHERE order_id >= 1 AND order_id < 100001
    #                       WHERE order_id >= 100001 AND order_id < 200001 ...
```
- Skew risk - equal numeric ranges ≠ equal row counts; use `predicates` for custom splits -
```python
    predicates = [
        "order_date < '2023-01-01'",
        "order_date >= '2023-01-01' AND order_date < '2024-01-01'",
        "order_date >= '2024-01-01'"
    ]
    df = spark.read.jdbc(url=url, table="orders", predicates=predicates, properties=props)
```

### Kafka / Streaming Sources

- Kafka partition count = Spark partition count (default); one task per Kafka partition
- `maxOffsetsPerTrigger` / `minPartitions` can override to split Kafka partitions further
- Ordering within a Kafka partition preserved within a Spark partition

---

## Shuffle Partitioning

- __Shuffle partitioning__ - how data is redistributed across partitions between stages; governed by `HashPartitioner` or `RangePartitioner`
- Every wide transformation produces a `ShuffledRDD` with a fixed output partition count

### Partition Count Control

- __DataFrame/SQL__ - `spark.sql.shuffle.partitions` (default $200$)
    - Applies to ALL shuffle operations in the query unless AQE coalesces them
    - Default $200$ is almost always wrong - tune per workload
- __RDD API__ - pass `numPartitions` argument to wide operations -
```python
    rdd.reduceByKey(lambda a, b: a + b, numPartitions=400)
    rdd.partitionBy(400)
```

### Sizing Shuffle Partitions

- Target $\sim 128 MB$ per partition after shuffle
- Formula - `numPartitions = ceil(totalShuffleOutputBytes / targetPartitionSize)`
- Too few partitions - tasks OOM; long GC pauses; stragglers; no parallelism
- Too many partitions - scheduler overhead dominates; tiny tasks; excessive small files; high Driver metadata load
- With AQE enabled - set `spark.sql.shuffle.partitions` high (eg - $1000$); AQE coalesces small partitions automatically

### HashPartitioner Mechanics

- `partition = nonNegativeMod(key.hashCode(), numPartitions)`
- Uniform distribution IF keys have uniform hash distribution
- Skewed key cardinality → skewed partitions; see Salting section

### RangePartitioner Mechanics

- Samples RDD to determine range boundaries; assigns keys to partitions by sorted range
- Used by `sortByKey`, `repartitionByRange`
- Sampling triggers a job - constructing `RangePartitioner` is not free

---

## Output Partitioning

- __Output partitioning__ - how data is physically organized on disk at write time; affects both write performance and downstream read efficiency

### Default Output Files

- `df.write.parquet(path)` - one output file per task (= one per shuffle partition or one per input partition)
- Number of output files = number of partitions in the DataFrame at write time
- Too many small files - HDFS NameNode metadata pressure; slow listing; slow reads due to file open overhead
- Too few large files - no parallelism on read; single task bottleneck

### Controlling Output File Count

- Python -
```python
    # Reduce files via coalesce (no shuffle - may produce uneven files)
    df.coalesce(10).write.parquet(path)

    # Reduce files via repartition (full shuffle - even files)
    df.repartition(10).write.parquet(path)

    # Limit records per file
    df.write.option("maxRecordsPerFile", 1000000).parquet(path)

    # AQE can coalesce partitions before write automatically
    # if spark.sql.adaptive.enabled=true
```

- `spark.sql.files.maxRecordsPerFile` (default $0$ = unlimited) - splits output files when record count threshold reached

---

## Coalesce vs Repartition Internals

### coalesce(n)

- __`coalesce(n)`__ - reduces partition count to `n` WITHOUT a full shuffle; merges existing partitions locally
- Implemented as `CoalesceExec` / `CoalescedRDD` - assigns multiple existing partitions to each new partition
- Each new partition reads from multiple old partitions on (potentially) multiple nodes - may require cross-node reads but NO shuffle exchange
- __Narrow dependency__ - `RangeDependency`; no shuffle write/read cycle

### When coalesce is Safe

- Reducing partition count - `df.repartition(1000).coalesce(100)` - merge $1000$ into $100$; safe
- Writing final output - `df.coalesce(1)` - collect to one file; DANGEROUS for large DataFrames (Driver/single executor OOM)
- After aggressive filter - much less data; reduce partitions without shuffle cost

### When coalesce Produces Skew

- If input partitions are already skewed, `coalesce` combines skewed partitions with normal ones
- Result - some coalesced partitions much larger than others; uneven task duration
- Fix - use `repartition` when even distribution is required

### repartition(n) / repartition(n, col)

- __`repartition(n)`__ - full shuffle to exactly `n` partitions; even distribution guaranteed via `HashPartitioner(n)`
- __`repartition(n, col)`__ - shuffle partitioned on specific column(s); all rows with same column value in same partition
- __Wide dependency__ - `ShuffleDependency`; full shuffle write + read

### repartition(n, col) vs partitionBy (Write)

- `df.repartition(n, "date")` - in-memory repartition; $n$ partitions; all rows with same `date` in same partition; multiple dates can share a partition
- `df.write.partitionBy("date")` - output directory partitioning; one directory per distinct date value; rows for different dates NEVER in same file; partition count per date = tasks writing that date

### coalesce vs repartition Decision

| Scenario | Use |
| --- | --- |
| Reduce partition count, data already balanced | `coalesce(n)` |
| Reduce partition count, even output required | `repartition(n)` |
| Change partitioning column | `repartition(n, col)` |
| Write single file | `coalesce(1)` (small data only) |
| Fix skew before join | `repartition(n, joinKey)` |

---

## Repartition by Range

- __`repartitionByRange(n, col)`__ - repartitions data using `RangePartitioner`; output partitions are globally sorted ranges of `col`
- Produces globally ordered data across partitions when read sequentially
- More expensive than `repartition` - requires sampling job to compute boundaries

### Use Cases

- Pre-sorting data for downstream sort-merge join (eliminates sort step in SMJ)
- Writing range-partitioned output for efficient range scans
- Creating globally sorted output for ranking operations

- Python -
```python
    # All rows with id in [0, 1000) in partition 0, [1000, 2000) in partition 1, etc.
    df.repartitionByRange(100, col("id")).write.parquet(path)

    # Multiple sort keys
    df.repartitionByRange(100, col("date").desc(), col("id")).write.parquet(path)
```

### Sampling Cost

- `RangePartitioner` construction samples the DataFrame to determine boundaries - triggers a Spark job
- Sample size - `min(20 × numPartitions, 1e6)` rows
- For $100$ partitions - up to $2000$ sample rows collected to Driver; fast
- For $10000$ partitions - up to $200000$ sample rows collected to Driver; noticeable overhead

---

## partitionBy at Write

- __`partitionBy`__ - organizes output files into a directory hierarchy based on column values; each distinct combination of partition column values gets its own directory
- Standard Hive-style partitioning; supported by all major query engines

### Directory Structure

```
utput/
├── year=2023/
│   ├── month=01/
│   │   ├── part-00000-uuid.parquet
│   │   └── part-00001-uuid.parquet
│   └── month=02/
│       └── part-00000-uuid.parquet
└── year=2024/
└── month=01/
└── part-00000-uuid.parquet
```

### Partition Discovery

- Spark automatically detects Hive-style partitioned directories on read
- Partition column values inferred from directory names - NOT stored in the files
- `spark.sql.sources.partitionColumnTypeInference.enabled=true` (default) - infers types from directory names

### Write Behavior

- `df.write.partitionBy("year", "month").parquet(path)` -
    - Each task writes to multiple directories (one per distinct partition value in its data)
    - Number of output files per partition directory = number of tasks that wrote to it
    - Tasks not aware of other tasks' partition values - multiple files per directory is normal

### Partition Overwrite Modes

- `spark.sql.sources.partitionOverwriteMode` -
    - `static` (default) - `overwrite` mode deletes ENTIRE output path before writing; all existing partitions removed
    - `dynamic` - `overwrite` mode only deletes partitions present in the new data; other partitions untouched
    - Always use `dynamic` for incremental writes to avoid accidentally deleting unchanged partitions

- Python -
```python
    spark.conf.set("spark.sql.sources.partitionOverwriteMode", "dynamic")

    # Only overwrites year=2024/month=01; other partitions untouched
    df_jan_2024.write.mode("overwrite").partitionBy("year", "month").parquet(path)
```

### Partition Pruning at Read

- `spark.read.parquet(path).filter(col("year") == 2024)` -
    - Spark lists only `year=2024/` directory; skips all other year directories
    - `FileSourceScanExec` applies partition filters before listing files
    - `spark.sql.hive.metastorePartitionPruning=true` - uses metastore partition metadata instead of directory listing (faster for tables with thousands of partitions)

### Choosing Partition Columns

- High cardinality columns (user_id, UUID) → millions of directories → NameNode metadata explosion; NEVER use
- Low-to-medium cardinality (date, country, status) → manageable directory count; good for pruning
- Cardinality guideline - partition column should have $< 10000$ distinct values total
- File size guideline - aim for $\geq 128 MB$ per partition directory to avoid small files

---

## bucketBy & sortBy

- __Bucketing__ - pre-partitions data into a fixed number of buckets by hashing a column; bucket assignment stored in file metadata; enables shuffle-free joins and sort-free sorts for subsequent queries
- Available only for `saveAsTable` (catalog-registered tables); not for path-based writes

### Bucketing Syntax

- Python -
```python
    df.write \
        .bucketBy(numBuckets=64, col="user_id") \
        .sortBy("user_id") \
        .mode("overwrite") \
        .saveAsTable("bucketed_events")

    # Multiple bucket columns
    df.write \
        .bucketBy(64, "user_id", "event_type") \
        .sortBy("user_id", "event_type") \
        .saveAsTable("bucketed_events")
```

- Scala -
```scala
    df.write
        .bucketBy(64, "user_id")
        .sortBy("user_id")
        .mode("overwrite")
        .saveAsTable("bucketed_events")
```

### File Naming Convention

- Bucket files named `part-XXXXX-{uuid}_00000.snappy.parquet` where `XXXXX` is the bucket ID
- $64$ buckets → $64$ files per write task group; task 0 writes bucket 0, task 1 writes bucket 1, etc.
- Actually - each write task writes to ALL buckets it has data for; bucket assignment via `hash(key) % numBuckets`

### sortBy with bucketBy

- `sortBy` within `bucketBy` - data WITHIN each bucket file is sorted by the specified columns
- Enables sort-free sort-merge join - both sides already sorted within each bucket
- `sortBy` without `bucketBy` - not supported; `sortBy` only meaningful with `bucketBy`

---

## Bucketing Internals & Shuffle Elimination

### How Bucketing Eliminates Shuffles

- At read time, Spark recognizes the bucketed partitioning of a table -
    - `FileSourceScanExec` sets `outputPartitioning = HashPartitioning(bucketColumns, numBuckets)`
    - `EnsureRequirements` checks if join children's `outputPartitioning` satisfies join requirements
    - If both sides bucketed on the same column with same bucket count → no `ShuffleExchangeExec` inserted

### Requirements for Shuffle-Free Bucketed Join

- Both tables bucketed on the SAME column(s)
- Same number of buckets
- Both tables registered in catalog (metastore)
- `spark.sql.sources.bucketing.enabled=true` (default `true`)
- `spark.sql.autoBroadcastJoinThreshold=-1` to prevent Spark from using BHJ instead of bucketed SMJ (otherwise Spark may broadcast a table rather than use bucketing)

- Python -
```python
    # Write both tables with same bucketing
    orders.write.bucketBy(64, "user_id").sortBy("user_id").saveAsTable("orders_bucketed")
    users.write.bucketBy(64, "user_id").sortBy("user_id").saveAsTable("users_bucketed")

    # Disable auto-broadcast to force bucketed join
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)

    # Join is shuffle-free AND sort-free
    result = spark.table("orders_bucketed").join(spark.table("users_bucketed"), "user_id")
    result.explain()
    # SortMergeJoin WITHOUT ShuffleExchangeExec - both sides already co-partitioned and sorted
```

### Bucketing and Sort Elimination

- With `bucketBy + sortBy` - data within each bucket is sorted
- SMJ on bucketed+sorted tables eliminates BOTH shuffles AND sorts
- `EnsureRequirements` sees both `outputPartitioning` and `outputOrdering` satisfied → no shuffle, no sort

### Bucketing Limitations

- Only works for tables registered in Hive metastore (or Spark catalog) - not path-based reads
- Bucketing metadata stored in metastore; if table recreated without bucketing, metadata lost
- Schema evolution - adding/removing columns doesn't invalidate bucketing; but changing bucket columns requires full rewrite
- Writes are slower - data must be sorted and split by bucket; more expensive than simple write
- `spark.sql.sources.bucketing.autoBucketedScanEnabled=true` (default `true` in Spark 3.1+) - automatically disables bucketed scan when it would not help (eg - aggregation without join)

---

## Z-Ordering

- __Z-ordering__ (also called Z-order clustering or space-filling curve ordering) - a data layout technique that co-locates related data across multiple dimensions within the same files; enables multi-column data skipping
- NOT a built-in Spark open-source feature - native to Delta Lake (`OPTIMIZE ... ZORDER BY`) and Databricks
- Understanding Z-ordering is principal-level knowledge for interviews about data layout optimization

### Problem Z-Ordering Solves

- Parquet/ORC row group statistics (`min`, `max` per column) enable single-column pruning -
    - File with `date` between `2024-01-01` and `2024-01-31` → skip if query filters `date = '2024-03-01'`
    - But filtering on `(date, user_id)` jointly - files sorted by `date` have `user_id` scattered throughout → no pruning on `user_id`
- Z-ordering interleaves values from multiple columns so that rows with similar values on ALL columns are physically co-located

### Z-Order Curve

- Maps multi-dimensional keys to a single 1D ordering by interleaving bits -
    - `date` bits and `user_id` bits interleaved → single Z-order key
    - Rows are sorted by Z-order key → rows close in multi-dimensional space are close in file order
- Result - each file covers a compact region in `(date, user_id)` space
- Row group statistics on both `date` AND `user_id` become useful for data skipping

### Z-Ordering via Delta Lake

- SQL (Delta Lake) -
```sql
    OPTIMIZE events ZORDER BY (event_date, user_id)
```
- Rewrites table files; each file covers a compact Z-order region; row group stats updated
- Subsequent queries with filters on `event_date` AND/OR `user_id` skip files efficiently

### Z-Ordering vs partitionBy

| Aspect | partitionBy | Z-Ordering |
| --- | --- | --- |
| Column cardinality | Low (< 10K distinct values) | High (millions of values) |
| Directory structure | One dir per value | No directory partitioning |
| Pruning mechanism | Directory listing | Row group min/max skipping |
| Multi-column | Hierarchical (nested dirs) | Joint multi-column |
| File count | One+ file per partition value | Controlled by OPTIMIZE |
| Engine support | All engines | Delta Lake / Databricks |

---

## Data Skew — Detection & Root Causes

- __Data skew__ - unequal distribution of data across partitions; some partitions have orders of magnitude more data than others; causes - one task runs for hours while others finish in seconds; stage stalls; SLA breaches

### Root Causes

- __Hot keys in shuffle__ - a few values of the join/groupBy key have vastly more rows than others
    - User_id with millions of events; product_id for a viral product; `null` keys all hash to partition 0
- __Null key concentration__ - `null` values all map to the same partition via `HashPartitioner` (null hashCode = 0 → partition 0 always)
- __Non-uniform hash distribution__ - custom key types with poor `hashCode` implementations; many keys collide to same bucket
- __Imbalanced input files__ - one HDFS file $10 GB$, others $100 MB$; cannot split non-splittable formats (gzip)
- __Range partitioning misconfiguration__ - `RangePartitioner` boundaries based on sample; if sample is not representative, actual data distribution creates uneven partitions
- __Explode on nested data__ - `explode(array_col)` where one row has an array of millions of elements → one partition gets all exploded rows from that one row

### Detection Methods

- __Spark UI__ - Stages tab → sort tasks by Duration or Input Size → dramatic outliers indicate skew
- __Task metrics__ -
```python
    # Check partition size distribution
    from pyspark.sql.functions import spark_partition_id, count

    df.groupBy(spark_partition_id().alias("partition")) \
      .count() \
      .orderBy(col("count").desc()) \
      .show(20)
```
- __Key frequency analysis__ -
```python
    # Find hot keys before a join
    df.groupBy("join_key") \
      .count() \
      .orderBy(col("count").desc()) \
      .show(20)

    # Statistical summary
    df.groupBy("join_key").count().describe("count").show()
```
- __AQE skew detection__ - `spark.sql.adaptive.skewJoin.enabled=true`; AQE logs detected skewed partitions in Spark UI

### Skew Symptoms in Spark UI

- Stage progress bar shows $199/200$ tasks complete for minutes/hours while $1$ task runs
- Task timeline shows one task bar extending far beyond all others
- "Shuffle Read Size" metric shows one task reading $10 GB$ while others read $100 MB$
- GC time high for one task (large partition causes GC pressure)

---

## Salting for Skew

- __Salting__ - artificially distributing hot keys across multiple partitions by appending a random integer suffix; eliminates hot key concentration at the cost of replicating the other side of a join

### Basic Salting Pattern

- Python -
```python
    from pyspark.sql.functions import col, concat, lit, rand, floor, explode, array

    n_salt = 10     # number of salt buckets; tune based on skew factor

    # Salt the skewed (large) side - random assignment
    left_salted = left.withColumn(
        "salted_key",
        concat(
            col("join_key").cast("string"),
            lit("_"),
            (rand() * n_salt).cast("int").cast("string")
        )
    )

    # Replicate the small side - all salt values for each key
    right_replicated = right.withColumn(
        "salt_arr",
        array([lit(i) for i in range(n_salt)])
    ).withColumn(
        "salt", explode(col("salt_arr"))
    ).withColumn(
        "salted_key",
        concat(
            col("join_key").cast("string"),
            lit("_"),
            col("salt").cast("string")
        )
    ).drop("salt_arr", "salt")

    # Join on salted key
    result = left_salted.join(right_replicated, "salted_key").drop("salted_key")
```

### Selective Salting (Hot Keys Only)

- Salting ALL keys is wasteful - replicates the entire right side `n_salt×`
- Only salt known hot keys; other keys join normally -

- Python -
```python
    hot_keys = {"user_123", "user_456"}   # known hot keys from analysis
    n_salt = 20

    from pyspark.sql.functions import when, broadcast

    # Left side - salt hot keys; leave others unchanged
    left_salted = left.withColumn(
        "salted_key",
        when(
            col("join_key").isin(*hot_keys),
            concat(col("join_key").cast("string"), lit("_"),
                   (rand() * n_salt).cast("int").cast("string"))
        ).otherwise(col("join_key").cast("string"))
    )

    # Right side - replicate hot keys; leave others unchanged
    right_normal = right.filter(~col("join_key").isin(*hot_keys)) \
                        .withColumn("salted_key", col("join_key").cast("string"))

    right_hot = right.filter(col("join_key").isin(*hot_keys)) \
                     .withColumn("salt_arr", array([lit(i) for i in range(n_salt)])) \
                     .withColumn("salt", explode(col("salt_arr"))) \
                     .withColumn("salted_key",
                         concat(col("join_key").cast("string"), lit("_"),
                                col("salt").cast("string"))) \
                     .drop("salt_arr", "salt")

    right_replicated = right_normal.union(right_hot)

    result = left_salted.join(right_replicated, "salted_key").drop("salted_key")
```

### Salting for GroupBy/Aggregation

- Two-phase aggregation with salting -

- Python -
```python
    n_salt = 10

    # Phase 1 - partial aggregate with salted key
    partial = df.withColumn(
        "salt", (rand() * n_salt).cast("int")
    ).groupBy("group_key", "salt") \
     .agg(sum("amount").alias("partial_sum"),
          count("*").alias("partial_count"))

    # Phase 2 - final aggregate without salt
    result = partial.groupBy("group_key") \
                    .agg(sum("partial_sum").alias("total_amount"),
                         sum("partial_count").alias("total_count"))
```

### Null Key Handling

- Null keys always hash to partition 0 → all nulls in one partition
- If nulls are valid join keys that should match → separate null handling -
```python
    # Filter nulls out before join; handle separately
    left_non_null = left.filter(col("join_key").isNotNull())
    left_null = left.filter(col("join_key").isNull())

    # Join non-null keys normally
    result_non_null = left_non_null.join(right, "join_key")

    # Handle nulls - cross join or fill with default
    result = result_non_null.union(left_null.withColumn("right_col", lit(None)))
```

### Salting Trade-offs

| Factor | Impact |
| --- | --- |
| `n_salt` value | Higher = better skew reduction; more right-side replication |
| Right side size | Small right side → replication cheap; large right side → expensive |
| Hot key fraction | Few hot keys → selective salting much cheaper than full salting |
| AQE availability | AQE `OptimizeSkewedJoin` may handle automatically; check first |

### When to Use Each Approach

- __AQE `OptimizeSkewedJoin`__ - first choice; automatic; no code changes; handles most production skew
- __Broadcast join__ - if non-skewed side is small; eliminates shuffle entirely; no salting needed
- __Full salting__ - when skew affects most keys; AQE insufficient; both sides large
- __Selective salting__ - when a few known hot keys cause skew; minimizes replication cost
- __Two-phase aggregation__ - for skewed `groupBy`; no join involved

> [!NOTE]
> Always check AQE skew join logs before implementing manual salting. `spark.sql.adaptive.skewJoin.enabled=true` with properly tuned `skewedPartitionFactor` and `skewedPartitionThresholdInBytes` handles the majority of real-world skew scenarios without any code changes. Manual salting adds complexity that must be maintained; prefer AQE when possible.

> [!TIP]
> Skew root cause matters for the fix -
> - Hot keys → salting or broadcast
> - Null concentration → null filter + separate handling
> - Imbalanced files → `repartition` at read or use splittable compression
> - Explode skew → pre-aggregate arrays before explode; limit array size
> - `RangePartitioner` miscalibration → increase sample size or use `HashPartitioner`