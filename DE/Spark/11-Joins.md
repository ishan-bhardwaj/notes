# Joins

## Join Overview & Strategy Selection

- __Join__ - combines rows from two relations based on a condition; the most expensive and most optimization-sensitive operation in Spark SQL
- Join strategy selection happens during Physical Planning (`JoinSelection` strategy in `SparkPlanner`)
- Five physical join algorithms available; selection based on -
    - Table size estimates (from CBO stats or file metadata)
    - Join type (inner, outer, semi, anti, cross)
    - Join condition type (equi-join vs non-equi-join)
    - Explicit hints

### Join Strategy Decision Tree

```
Does join have an equi-condition?
├── No  → BroadcastNestedLoopJoin (if one side broadcastable)
│         → CartesianProduct (if no condition at all)
└── Yes → Can one side be broadcast? (size < autoBroadcastJoinThreshold = 10 MB)
├── Yes → BroadcastHashJoin (fastest; no shuffle)
└── No  → Can one side fit in memory as hash table?
├── Yes → ShuffledHashJoin (one shuffle)
└── No  → SortMergeJoin (default; two shuffles)
```

### Join Types

- __Inner join__ - only rows with matching keys on both sides
- __Left outer join__ - all left rows; `null` for missing right values
- __Right outer join__ - all right rows; `null` for missing left values
- __Full outer join__ - all rows from both sides; `null` for missing values on either side
- __Left semi join__ - left rows where key exists in right; right values NOT included in output
- __Left anti join__ - left rows where key does NOT exist in right
- __Cross join__ - Cartesian product; every left row × every right row

### Key Configurations

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.sql.autoBroadcastJoinThreshold` | `10MB` | Size threshold for auto-broadcast |
| `spark.sql.join.preferSortMergeJoin` | `true` | Prefer SMJ over SHJ even when SHJ applicable |
| `spark.sql.shuffle.partitions` | `200` | Shuffle partition count for join exchanges |
| `spark.sql.adaptive.enabled` | `true` | Enable AQE join re-planning |

---

## Broadcast Hash Join (BHJ)

- __`BroadcastHashJoinExec`__ - broadcasts the smaller relation to all executors; builds a hash table from it in memory; each executor probes the hash table with its local partition of the larger relation
- Fastest join algorithm - zero shuffle for the larger side; no sorting required
- Only requires one side to be small enough to broadcast

### Execution Flow

1. `BroadcastExchangeExec` collects the small side to Driver; serializes to `HashedRelation`
2. Driver broadcasts `HashedRelation` to all executors via `TorrentBroadcast`
3. Each executor holds the full small-side hash table in memory
4. Large side partitions processed locally - each row probed against local hash table
5. Matching rows emitted; no cross-executor communication after broadcast

### HashedRelation

- __`HashedRelation`__ - the in-memory hash map built from the broadcast side
- `UnsafeHashedRelation` - Tungsten-backed; keys and values stored as binary `UnsafeRow` in a hash map
- `LongToUnsafeRowMap` - optimized implementation when join key is a single `Long` (most numeric joins)
    - Uses open-addressing hash map with `long[]` backing; fastest possible lookup
- `HashedRelation` built on Driver; serialized and sent via broadcast mechanism
- Each executor deserializes once; reused across all tasks on that executor

### When BHJ is Selected

- One side estimated size < `spark.sql.autoBroadcastJoinThreshold` ($10 MB$ default)
- OR explicit `broadcast()` hint applied
- For outer joins - only applicable when the broadcast side is the appropriate side -
    - Left outer join - can broadcast right side (build hash table from nullable side)
    - Right outer join - can broadcast left side
    - Full outer join - BHJ NOT applicable (cannot broadcast either side for full outer)

### AQE and BHJ

- `DynamicJoinSelection` AQE rule - converts `SortMergeJoin` → `BHJ` at runtime if actual shuffle output size is below threshold
- Runtime conversion path -
    - Map stage of SMJ completes → actual partition sizes known
    - If smaller side total size < threshold → AQE inserts `BroadcastExchange` + replans to `BHJ`
    - Avoids the sort and shuffle of the larger side entirely

### BHJ Memory Risk

- `HashedRelation` must fit in executor memory - specifically in execution memory pool
- If broadcast table is larger than expected (stale stats) → `HashedRelation` build OOM
- `spark.sql.autoBroadcastJoinThreshold=-1` to disable auto-broadcast; use explicit `broadcast()` hint only when certain
- Signs of BHJ OOM - `java.lang.OutOfMemoryError` in `HashedRelation` or `BroadcastExchangeExec`

---

## Sort Merge Join (SMJ)

- __`SortMergeJoinExec`__ - shuffles both sides on the join key; sorts both sides; merges sorted streams
- Default join algorithm for large equi-joins when neither side can be broadcast
- Scales to arbitrarily large tables - spill-tolerant; does not require either side to fit in memory

### Execution Flow

1. __Shuffle both sides__ - `ShuffleExchangeExec` added by `EnsureRequirements` for each side
    - Both sides hash-partitioned on join key with same `HashPartitioner(n)`
    - Rows with same join key guaranteed to land in same partition on both sides
2. __Sort each partition__ - `SortExec` added for each side within each partition
    - Sort by join key; `ExternalSorter` used; spill-tolerant
3. __Merge sorted streams__ - `SortMergeJoinExec.doExecute()` processes matching partitions pairwise
    - Two-pointer merge scan; O(n + m) per partition pair
    - For each key, cross-products matching rows from left and right buffers
    - `StreamedPlan` (left) and `BufferedPlan` (right) terminology - one side streamed, other buffered per key group

### Memory Requirements

- SMJ is the most memory-efficient join algorithm - only needs to buffer rows with the same key simultaneously
- If one key has millions of values on both sides - the cross-product of matching rows must fit in memory
- For skewed keys - AQE `OptimizeSkewedJoin` splits the hot partition

### SMJ vs BHJ Performance

- BHJ always faster than SMJ when broadcast is possible - no shuffle for large side, no sort
- SMJ required when both sides are large - accepts higher latency for correctness at scale
- SMJ shuffle cost = two `ShuffleExchangeExec` nodes; verify in `executedPlan`

### SMJ Codegen

- `SortMergeJoinExec` implements `CodegenSupport` - the merge loop is code-generated
- Inner join path fully fused into WSCG; outer join paths partially fused
- Generated merge loop eliminates virtual dispatch per row comparison

---

## Shuffled Hash Join (SHJ)

- __`ShuffledHashJoinExec`__ - shuffles both sides on the join key; builds a hash table from the smaller side per partition; probes with the larger side per partition
- Middle ground between BHJ (no shuffle) and SMJ (sort required) - one shuffle, no sort

### Execution Flow

1. __Shuffle both sides__ - same as SMJ; both sides hash-partitioned on join key
2. __Build phase__ - for each partition pair, build hash table from the smaller side in memory
    - `HashedRelation` built per partition (not broadcast globally)
    - Memory required = size of smaller side's partition
3. __Probe phase__ - stream larger side through hash table; emit matches

### When SHJ is Selected

- Equi-join where one side not broadcastable globally BUT one side fits in memory per partition
- `spark.sql.join.preferSortMergeJoin=true` (default) - even when SHJ applicable, SMJ preferred
    - Set to `false` to allow SHJ when conditions met
- Explicit `SHUFFLE_HASH` hint forces SHJ regardless of preference setting

### SHJ vs SMJ

| Aspect | SHJ | SMJ |
| --- | --- | --- |
| Shuffle | Both sides | Both sides |
| Sort | No | Yes (both sides) |
| Memory | Must fit smaller partition in memory | Only buffers matching rows per key |
| Spill | `HashedRelation` cannot spill (build side OOM risk) | `ExternalSorter` spills |
| Skew handling | OOM on skewed build side partition | AQE `OptimizeSkewedJoin` handles |
| Speed | Faster than SMJ (no sort) | Slower; spill-safe |

### SHJ OOM Risk

- Build side partition must fit in executor memory - no spill path for `HashedRelation`
- Skewed partition on build side → that partition's `HashedRelation` OOM
- SMJ generally safer for unknown data distributions; SHJ for controlled, uniform data

---

## Broadcast Nested Loop Join (BNLJ)

- __`BroadcastNestedLoopJoinExec`__ - broadcasts one side; iterates all broadcast rows for every row of the other side; O(n × m)
- Used for non-equi joins where no equality condition exists
- Only viable when broadcast side is very small (tens of thousands of rows maximum)

### Execution Flow

1. Broadcast smaller side to all executors (same mechanism as BHJ)
2. For each row of the larger side (streamed partition by partition) -
    - Iterate ALL rows of the broadcast side
    - Apply join condition to each pair
    - Emit matching pairs
3. O(n × m) per executor - catastrophic for large broadcast sides

### When BNLJ is Selected

- Non-equi join condition (range condition, `<`, `>`, `!=`, complex expressions) AND one side broadcastable
- Example - `a.start_date < b.date AND b.date < a.end_date` (range overlap join)
- Also used for cross joins with a filter condition applied after

### BNLJ Performance

- $10K$ broadcast rows × $10M$ stream rows = $10^{11}$ comparisons per executor
- Acceptable only for broadcast side < $\sim 10K$ rows
- If BNLJ appears in plan with large broadcast side → rewrite query to use equi-join or range join optimization

---

## Cross Join

- __`CartesianProductExec`__ - Cartesian product of two relations; every left row paired with every right row
- No join condition; output size = `|left| × |right|`
- Almost always a mistake at scale; Spark warns but does not prevent

### Execution

- Left side partitioned (or kept as-is)
- Right side replicated to every left partition
- Each left partition paired with each right partition; full cross product computed

### Safety Gate

- `spark.sql.crossJoin.enabled=false` (default `false` in older Spark; `true` in Spark 3+)
- In older Spark - cross join throws `AnalysisException` unless enabled explicitly
- In Spark 3+ - allowed but logs warning

### Legitimate Cross Join Uses

- Small dimension table × large fact table to generate all combinations
- Generating test data combinations
- Date spine joins (all dates × all entities) - usually better handled with `sequence` + `explode`

---

## Co-partitioned Join (Shuffle-free)

- __Co-partitioned join__ - when both sides of a join are already partitioned with the same `Partitioner` on the join key; no shuffle required; narrowly the most efficient join after BHJ

### How Co-partitioning is Detected

- `EnsureRequirements` checks `outputPartitioning` of each join child -
    - If both children have `HashPartitioning(joinKeys, n)` with the same `n` and same keys → join is already co-partitioned
    - No `ShuffleExchangeExec` inserted
- `DAGScheduler` can then use `ZippedPartitionsRDD` - each partition pair processed together without shuffle

### Achieving Co-partitioning

- DataFrame API -
```python
    # Pre-partition both DataFrames on join key with same partition count
    n = 200     # must match spark.sql.shuffle.partitions or be explicit

    df1_partitioned = df1.repartition(n, "user_id").cache()
    df2_partitioned = df2.repartition(n, "user_id").cache()

    # This join is shuffle-free - both already co-partitioned
    result = df1_partitioned.join(df2_partitioned, "user_id")
    result.explain()
    # SortMergeJoin WITHOUT ShuffleExchangeExec above either child - co-partitioned
```

- RDD API -
```python
    rdd1 = pairs1.partitionBy(200)      # HashPartitioner(200)
    rdd2 = pairs2.partitionBy(200)      # HashPartitioner(200)
    joined = rdd1.join(rdd2)            # narrow - no shuffle
```

### When Co-partitioning Pays Off

- Iterative algorithms that repeatedly join the same two datasets
- Star schema queries where fact table always joined to same dimension on same key
- Pay the shuffle cost once (during `repartition`); all subsequent joins are free
- Must `cache()` co-partitioned DataFrames - otherwise `repartition` re-shuffles on each action

### Co-partitioning with SMJ

- SMJ on co-partitioned data still requires sort within each partition
- To also eliminate the sort - pre-sort within partitions -
```python
    df1_ready = df1.repartition(200, "user_id").sortWithinPartitions("user_id").cache()
    df2_ready = df2.repartition(200, "user_id").sortWithinPartitions("user_id").cache()
    # Join now requires no shuffle AND no sort
```

---

## Join Hints

- __Join hints__ - explicit instructions to the query planner to use a specific join algorithm; override CBO and heuristic selection
- Applied per query; do not affect global configuration

### Hint Syntax

- SQL -
```sql
    -- Broadcast hint
    SELECT /*+ BROADCAST(t1) */ * FROM t1 JOIN t2 ON t1.id = t2.id

    -- Shuffle Hash hint
    SELECT /*+ SHUFFLE_HASH(t1) */ * FROM t1 JOIN t2 ON t1.id = t2.id

    -- Sort Merge hint
    SELECT /*+ MERGE(t1) */ * FROM t1 JOIN t2 ON t1.id = t2.id

    -- Shuffle Replicate NL hint (broadcast nested loop)
    SELECT /*+ SHUFFLE_REPLICATE_NL(t1) */ * FROM t1 JOIN t2 ON t1.id = t2.id
```

- Python -
```python
    from pyspark.sql.functions import broadcast

    # Force BHJ
    result = large_df.join(broadcast(small_df), "user_id")

    # Force SMJ via hint
    result = df1.hint("MERGE").join(df2, "user_id")

    # Force SHJ via hint
    result = df1.hint("SHUFFLE_HASH").join(df2, "user_id")

    # Force BNLJ via hint
    result = df1.hint("SHUFFLE_REPLICATE_NL").join(df2, "user_id")
```

### Hint Priority

- Explicit hints always override CBO and heuristics
- If conflicting hints on both sides - left side hint takes priority
- If hint requests an algorithm that cannot work (eg - broadcast a non-broadcastable table) - Spark falls back to next best algorithm and logs warning

### When to Use Hints

- Auto-broadcast threshold wrong (stale stats) - use `broadcast()` hint when you know the table is small
- SMJ chosen when SHJ would be faster (uniform data, controlled skew) - use `SHUFFLE_HASH` hint
- BHJ chosen but causing OOM (stale stats overestimate smallness) - set `autoBroadcastJoinThreshold=-1` or use `MERGE` hint to force SMJ

---

## Join Reordering

- Covered in detail in CBO section; summary here for join-specific context
- __Join reordering__ - reorders multi-way joins to minimize intermediate result sizes
- Enabled via `spark.sql.cbo.joinReorder.enabled=true` (requires `spark.sql.cbo.enabled=true`)

### Impact on Physical Join Selection

- Join ordering changes which tables are joined first → changes intermediate sizes → changes join algorithm selection
- Example - reordering `A JOIN B JOIN C` to `A JOIN C JOIN B` might make `A JOIN C` small enough for BHJ
- Without CBO - planner uses heuristics; larger tables may end up on the "build" side of a hash join

### Interaction with AQE

- Join ordering fixed at planning time - AQE cannot reorder joins at runtime
- AQE can change algorithm (`DynamicJoinSelection`) but not order
- CBO join reordering + AQE algorithm selection = complementary; use both

---

## Multi-way Joins

- Spark SQL processes multi-way joins as a left-deep tree of binary joins by default
- `A JOIN B JOIN C JOIN D` → `((A JOIN B) JOIN C) JOIN D`
- CBO join reordering can change this tree shape (bushy trees possible with DP algorithm)

### Left-Deep vs Bushy Trees

- __Left-deep tree__ - each join's left child is another join; right child is a base table
    - Predictable execution; each stage's output feeds next stage sequentially
    - `((A JOIN B) JOIN C) JOIN D`
- __Bushy tree__ - joins can have join results as both children
    - Can parallelize independent sub-joins; better for certain query shapes
    - `(A JOIN B) JOIN (C JOIN D)` - `A JOIN B` and `C JOIN D` can run in parallel
    - Spark CBO DP algorithm can produce bushy trees

### Optimization Strategies

- Enable CBO + join reorder for queries with $3+$ table joins
- Pre-filter tables before joining - reduce input sizes; smaller inputs → cheaper joins regardless of order
- Consider intermediate `cache()` for heavily reused join results

---

## Range Join Optimization

- __Range join__ - join condition involves range predicates (`a.start <= b.value AND b.value <= a.end`); no equality condition available
- Standard SMJ cannot exploit sort order for range conditions; standard planner uses BNLJ (very slow) or SMJ with post-filter

### Challenge

- Range joins have no equi-condition → cannot be hash-partitioned efficiently
- Naive approach - Cartesian product then filter; O(n × m)
- With BNLJ and broadcast - O(n × m) per executor; only viable for very small broadcast side

### Range Join Optimization in Spark

- Spark does NOT have a native range join optimizer in the open-source version
- Third-party libraries (eg - Databricks, Sedona for spatial joins) implement optimized range join via -
    - __Binning__ - discretize the range into fixed-size bins; each row replicated to bins it overlaps; bin ID used as equi-join key; then filter to eliminate false positives
    - Turns range join into equi-join + filter; standard SMJ or BHJ applicable

### Manual Range Join Optimization

- Python -
```python
    from pyspark.sql.functions import col, floor, sequence, explode

    bin_size = 100  # tune based on range overlap characteristics

    # Replicate each row to all bins it overlaps
    intervals = intervals.withColumn(
        "bins",
        sequence(
            floor(col("start") / bin_size).cast("long"),
            floor(col("end") / bin_size).cast("long")
        )
    ).withColumn("bin", explode(col("bins"))).drop("bins")

    points = points.withColumn("bin", floor(col("value") / bin_size).cast("long"))

    # Now equi-join on bin + filter for actual range condition
    result = intervals.join(points, "bin") \
        .filter((col("value") >= col("start")) & (col("value") <= col("end"))) \
        .drop("bin")
```

### Performance Characteristics

- Replication factor = average number of bins per interval = `avg_interval_length / bin_size`
- Small `bin_size` → more bins → more replication → larger shuffle → more false positives filtered
- Large `bin_size` → fewer bins → less replication → less precise pruning → more rows in final filter
- Optimal `bin_size` ≈ $\sqrt{avg\_interval\_length \times avg\_query\_selectivity}$

---

## Skewed Join Handling

- __Join skew__ - when one join key has disproportionately many rows; that partition's task runs far longer than others; holds up the entire stage
- Most common cause of SLA-breaking join slowness in production; classic symptom - $99\%$ tasks done, one task running for hours

### Diagnosing Skew

- Spark UI → Stages → find the join stage → sort tasks by duration → one or few tasks much slower than median
- Stages tab → "Input Size / Records" column → one partition has orders of magnitude more records
- Python -
```python
    # Check partition size distribution
    df.groupBy(spark_partition_id()).count().orderBy(col("count").desc()).show(20)

    # Find hot keys
    df.groupBy("join_key").count().orderBy(col("count").desc()).show(20)
```

### AQE Skew Join Optimization

- `spark.sql.adaptive.skewJoin.enabled=true` (default `true` when AQE enabled)
- AQE detects skewed partitions AFTER shuffle completes (actual sizes known) -
    - A partition is skewed if `size > spark.sql.adaptive.skewJoin.skewedPartitionFactor × median_size` (default $5×$) AND `size > spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` (default $256 MB$)
- For each skewed partition -
    - Splits it into multiple smaller sub-partitions
    - Replicates the corresponding partition from the other side to match each sub-partition
    - Each sub-partition processed by a separate task
- Only works for `SortMergeJoin` - `BroadcastHashJoin` skew handled naturally (broadcast side replicated to all)

### Manual Salting

- When AQE is disabled or skew is extreme -
- Python -
```python
    import random
    from pyspark.sql.functions import col, concat, lit, floor, rand, explode, array

    n_salt = 10

    # Salt the left (skewed) side - append random int to hot key
    left_salted = left.withColumn(
        "salted_key",
        concat(col("join_key"), lit("_"), (rand() * n_salt).cast("int").cast("string"))
    )

    # Replicate the right side - generate all salt values for each key
    right_replicated = right.withColumn(
        "salt_arr",
        array([lit(i) for i in range(n_salt)])
    ).withColumn("salt", explode(col("salt_arr"))) \
     .withColumn(
        "salted_key",
        concat(col("join_key"), lit("_"), col("salt").cast("string"))
    ).drop("salt_arr", "salt")

    # Join on salted key
    result = left_salted.join(right_replicated, "salted_key") \
                        .drop("salted_key")
```

### Salting Trade-offs

- __Pro__ - distributes hot key across `n_salt` partitions; each partition $n\_salt×$ smaller
- __Con__ - right side replicated `n_salt×` → `n_salt×` more shuffle data for right side
- Tune `n_salt` based on skew factor - if one key has $100×$ median records, `n_salt=10` reduces to $10×$ median (still slightly skewed but manageable)
- Only salt the hot keys - use `CASE WHEN` to apply salting only to known hot keys; other keys join normally

### Two-Phase Aggregation for Skewed GroupBy

- Skew affects `groupByKey`/`reduceByKey` as well as joins
- Pattern for skewed aggregation -
```python
    # Phase 1 - partial aggregate with salted key
    partial = df.withColumn("salt", (rand() * n_salt).cast("int")) \
                .groupBy("join_key", "salt") \
                .agg(sum("amount").alias("partial_sum"))

    # Phase 2 - final aggregate without salt
    result = partial.groupBy("join_key") \
                    .agg(sum("partial_sum").alias("total_sum"))
```

### Broadcast as Skew Solution

- If the OTHER side (non-skewed side) is small enough - broadcast it
- Eliminates shuffle entirely; skew on large side doesn't matter (each executor processes its local partitions independently with the broadcast hash table)
- Most effective skew solution when applicable

> [!NOTE]
> AQE `OptimizeSkewedJoin` handles skew automatically for `SortMergeJoin` when enabled. Always enable AQE (`spark.sql.adaptive.enabled=true`) before implementing manual salting - AQE handles most real-world skew without code changes. Manual salting is a last resort for extreme skew or when AQE is unavailable.

> [!TIP]
> Join optimization priority order -
> 1. Can one side be broadcast? → Use BHJ (fastest; no shuffle)
> 2. Are both sides pre-partitioned on join key? → Co-partitioned join (no shuffle)
> 3. Is AQE enabled? → Let AQE handle skew and algorithm selection at runtime
> 4. Are stats accurate and CBO enabled? → Let CBO choose join order
> 5. Known skew + AQE insufficient? → Manual salting
> 6. Non-equi join with small side? → BNLJ with broadcast
