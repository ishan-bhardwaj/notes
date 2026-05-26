# Adaptive Query Execution

## AQE Overview & Architecture

- __Adaptive Query Execution (AQE)__ - Spark's runtime query re-optimization framework; re-plans parts of a query using actual statistics collected from completed shuffle and broadcast stages
- Enabled by default since Spark 3.2 - `spark.sql.adaptive.enabled=true`
- Core insight - statistics estimated at planning time are approximate (stale stats, independence assumptions, NDV errors); actual shuffle output sizes are exact; re-planning with exact statistics produces better plans

### What AQE Can and Cannot Do

- __Can__ -
    - Coalesce small post-shuffle partitions into fewer, larger partitions
    - Convert `SortMergeJoin` → `BroadcastHashJoin` at runtime when actual join side is small
    - Split skewed shuffle partitions into smaller sub-partitions and replicate the other side
    - Eliminate empty relations (skip join/aggregation when a stage produces zero rows)
- __Cannot__ -
    - Reorder joins - join ordering fixed at planning time
    - Change scan strategies - file read plans fixed before any computation
    - Re-plan after result stage - only re-plans between shuffle stages

### AQE vs Static Planning

| Aspect | Static Planning | AQE |
| --- | --- | --- |
| Statistics source | CBO estimates or defaults | Actual shuffle output |
| Join algorithm | Fixed at plan time | Can change at runtime |
| Partition count | Fixed at plan time | Adjusted at runtime |
| Skew handling | Manual salting or CBO hints | Automatic detection and splitting |
| Plan reuse | Full plan compiled once | Plan fragments recompiled after each stage |

### High-Level Architecture

```
AdaptiveSparkPlanExec
├── initialPlan: SparkPlan          ← static plan; fallback if AQE disabled
├── currentPhysicalPlan: SparkPlan  ← live plan; updated after each stage
└── queryStageOptimizerRules        ← rules applied after each stage completes
├── CoalesceShufflePartitions
├── OptimizeSkewedJoin
└── OptimizeLocalShuffleReader
```

---

## QueryStageExec

- __`QueryStageExec`__ - the fundamental unit of AQE execution; represents a portion of the query plan that can be executed independently and whose output statistics can be collected
- AQE inserts `QueryStageExec` boundaries at every shuffle and broadcast exchange
- Each `QueryStageExec` is a self-contained stage - executed as a complete Spark job before downstream stages are re-planned

### Purpose

- Materializes intermediate results (shuffle files or broadcast data)
- After materialization, collects exact statistics (partition sizes, row counts)
- These statistics feed the AQE re-optimizer for downstream plan fragments

### QueryStageExec Lifecycle

1. Created during initial planning - `InsertAdaptiveSparkPlan` rule wraps each exchange in a `QueryStageExec`
2. `AdaptiveSparkPlanExec` executes stages in dependency order
3. Stage completes - collects `MapOutputStatistics` (shuffle) or broadcast data size
4. `AdaptiveSparkPlanExec` re-optimizes the remaining plan using actual statistics
5. Creates new `QueryStageExec`s for next wave of stages with updated plan
6. Repeat until all stages complete

### Stage Reuse

- If two different parts of the query plan share an identical `QueryStageExec` subtree -
    - AQE detects the structural equivalence
    - Executes the stage only once
    - Both consumers reuse the same `MapOutputStatistics`
    - `ReuseExchange` and `ReuseAdaptiveSubquery` rules handle this

---

## ShuffleQueryStageExec

- __`ShuffleQueryStageExec`__ - a `QueryStageExec` wrapping a `ShuffleExchangeExec`; materializes shuffle output and collects partition statistics

### Statistics Collection

- After the map stage completes, `MapOutputTracker` holds `MapStatus` for each map task
- `ShuffleQueryStageExec.mapStats` - lazily computes `MapOutputStatistics` from `MapStatus` array -
    - `bytesByPartitionId: Array[Long]` - actual byte size of each shuffle output partition
    - `recordsByPartitionId: Array[Long]` - actual record count per partition (if available)
- These exact sizes are what AQE rules use for coalescing and skew detection

### Materialization

- `ShuffleQueryStageExec.doMaterialize()` - calls `ShuffleExchangeExec.submitShuffleJob()` -
    - Launches map tasks as a Spark job
    - Returns a `Future` that completes when all map tasks finish
    - `AdaptiveSparkPlanExec` awaits all stage futures before re-planning

### Result Reuse

- `ShuffleQueryStageExec.isMaterialized` - true after map stage completes
- Reused stages skip `doMaterialize()` - shuffle files already on disk; `MapOutputStatistics` already available

---

## BroadcastQueryStageExec

- __`BroadcastQueryStageExec`__ - a `QueryStageExec` wrapping a `BroadcastExchangeExec`; materializes broadcast data and collects size statistics

### Statistics Collection

- After broadcast completes, collects -
    - Total serialized byte size of the broadcast relation
    - Row count of broadcast data
- Used by `DynamicJoinSelection` to decide if a `SortMergeJoin` side can be converted to broadcast

### Materialization

- `BroadcastQueryStageExec.doMaterialize()` - calls `BroadcastExchangeExec.submitBroadcastJob()` -
    - Collects broadcast side data to Driver
    - Builds `HashedRelation` or `IdentityBroadcastMode` relation
    - Broadcasts to all executors via `TorrentBroadcast`
    - Returns `Future[Broadcast[Any]]`

### Interaction with DynamicJoinSelection

- When a `ShuffleQueryStageExec` completes and actual output size < broadcast threshold -
    - `DynamicJoinSelection` may decide to replace the downstream `SortMergeJoin` with `BroadcastHashJoin`
    - This requires creating a NEW `BroadcastQueryStageExec` to broadcast the small side
    - The original `ShuffleQueryStageExec` data (already on disk) is broadcast from the Driver by reading shuffle files

---

## AdaptiveSparkPlanExec

- __`AdaptiveSparkPlanExec`__ - the top-level physical plan node for AQE-enabled queries; orchestrates the entire adaptive execution lifecycle
- Wraps the entire static physical plan; returned as the `executedPlan` when AQE is enabled
- One `AdaptiveSparkPlanExec` per query (or per subquery in complex queries)

### Internal State

- `initialPlan: SparkPlan` - the original static plan; never modified
- `currentPhysicalPlan: SparkPlan` - live plan; updated after each stage completes; what actually executes
- `queryStageOptimizerRules: Seq[Rule[SparkPlan]]` - rules applied after each stage to re-optimize
- `allChildStages: Seq[QueryStageExec]` - all materialized and pending stages
- `stageMaterializationFutures: Map[QueryStageExec, Future[_]]` - async tracking of stage completion

### Execution Loop

1. `AdaptiveSparkPlanExec.doExecute()` -
2. Find all "ready" `QueryStageExec`s (all parents materialized)
3. Submit ready stages for execution (async)
4. Wait for any stage to complete
5. Collect statistics from completed stage
6. Apply `queryStageOptimizerRules` to remaining plan using new statistics
7. Create new `QueryStageExec`s for newly exposed exchanges in updated plan
8. Go to step 2 until final stage (result stage) is ready
9. Execute final stage; return `RDD[InternalRow]`

### Plan Reoptimization

- After each stage completes, `AdaptiveSparkPlanExec.reOptimize(logicalPlan)` -
    - Uses updated statistics (injected as `Statistics` overrides on `LogicalPlan` nodes)
    - Re-runs physical planning (`SparkPlanner`) on the updated logical plan
    - Applies `queryStageOptimizerRules` to the new physical plan
    - Compares new plan to current plan; if different, updates `currentPhysicalPlan`
- Logical plan preserved throughout - only physical plan changes

### FinalStage

- Once all shuffle/broadcast stages complete, the remaining plan fragment is the "final stage"
- Final stage is a `ResultQueryStageExec` (result task) or another shuffle (if multi-level)
- Executed with the fully re-optimized physical plan

---

## AQE Runtime Statistics

- AQE collects two types of runtime statistics -

### MapOutputStatistics

- Collected from `MapOutputTracker` after shuffle map stage completes
- `bytesByPartitionId: Array[Long]` - exact byte size of each reduce partition
- Used by -
    - `CoalesceShufflePartitions` - identifies small partitions to merge
    - `OptimizeSkewedJoin` - identifies large partitions to split
    - `DynamicJoinSelection` - checks if total size < broadcast threshold

### Partition Statistics Accuracy

- `MapStatus` stores compressed partition sizes (log-scale 8-bit encoding)
- AQE decompresses to get approximate sizes - not exact bytes but very close
- `HighlyCompressedMapStatus` (used for high partition count) stores average + bitmap - less accurate
- `spark.sql.adaptive.fetchShuffleBlocksInBatch` - batch fetch optimization affects how stats are used

### Statistics Injection into Logical Plan

- After stage completes, AQE injects actual statistics back into the logical plan -
    - `LogicalRDD` nodes corresponding to completed stages get `Statistics(sizeInBytes, rowCount)` overridden with actual values
    - These overridden statistics feed the CBO-style cardinality estimation for downstream re-planning
    - `spark.sql.adaptive.forceOptimizeSkewedJoin` - force skew optimization even when stats seem fine

---

## Dynamic Partition Coalescing

- __`CoalesceShufflePartitions`__ - AQE rule that merges small post-shuffle partitions into fewer, larger partitions; reduces task overhead for stages following a shuffle that produces many tiny partitions

### The Problem

- `spark.sql.shuffle.partitions=200` (default) creates $200$ post-shuffle partitions
- If the shuffled dataset is small (after aggressive filtering), $200$ partitions may each have only kilobytes of data
- $200$ tiny tasks → scheduler overhead dominates; each task spawned, serialized, scheduled, started, completed with negligible actual work

### How CoalesceShufflePartitions Works

- After `ShuffleQueryStageExec` materializes, `CoalesceShufflePartitions` examines `bytesByPartitionId` -
    1. Greedily groups consecutive partitions (must be consecutive - data locality)
    2. Keeps merging until combined size reaches `spark.sql.adaptive.advisoryPartitionSizeInBytes` (default $64 MB$)
    3. Creates `CoalescedShuffleSpec` mapping groups of original partition IDs to single new partitions
    4. Injects `AQEShuffleReadExec` nodes above the shuffle that read multiple original partitions as one

### AQEShuffleReadExec

- Physical operator that reads multiple shuffle partitions as a single logical partition
- No data movement - reads directly from existing shuffle files
- Reduce tasks for the coalesced plan read from multiple original partition IDs in one task

### Configuration

- `spark.sql.adaptive.advisoryPartitionSizeInBytes` (default $64 MB$) - target size per coalesced partition
- `spark.sql.adaptive.coalescePartitions.minPartitionSize` (default $1 MB$) - minimum partition size; partitions smaller than this always coalesced
- `spark.sql.adaptive.coalescePartitions.minPartitionNum` (default `2 × spark.default.parallelism`) - minimum number of partitions after coalescing; prevents over-coalescing
- `spark.sql.adaptive.coalescePartitions.enabled` (default `true`) - enable/disable this specific rule
- `spark.sql.adaptive.coalescePartitions.parallelismFirst` (default `true`) - prioritize parallelism over partition size; set `false` to strictly target `advisoryPartitionSizeInBytes`

### Impact

- Before AQE - must manually tune `spark.sql.shuffle.partitions` per query
- With AQE - set a high `spark.sql.shuffle.partitions` (eg - $1000$); AQE coalesces down to appropriate count automatically
- Particularly impactful for queries with highly selective filters - query may filter $99\%$ of data; post-shuffle data tiny; AQE collapses $200$ tasks to $2-3$

---

## Dynamic Join Strategy Switching

- __`DynamicJoinSelection`__ - AQE rule that converts `SortMergeJoin` → `BroadcastHashJoin` at runtime when actual shuffle output size is below broadcast threshold

### Trigger Conditions

- After `ShuffleQueryStageExec` for one join side completes -
    - Actual total size of that side's shuffle output < `spark.sql.autoBroadcastJoinThreshold` ($10 MB$ default)
    - OR actual total size < `spark.sql.adaptive.maxShuffleHashJoinLocalMapThreshold` → convert to `ShuffledHashJoin`
    - Join type supports broadcast (inner, left outer with right broadcast, etc.)

### Conversion Process

1. `DynamicJoinSelection` detects small shuffle output from `MapOutputStatistics`
2. Marks the `SortMergeJoin` for conversion
3. AQE inserts new `BroadcastExchangeExec` for the small side
4. Creates `BroadcastQueryStageExec` wrapping the new broadcast exchange
5. Replaces `SortMergeJoinExec` with `BroadcastHashJoinExec` in updated physical plan
6. The large side's `ShuffleQueryStageExec` (already complete) feeds directly into `BHJ` probe side
7. Large side shuffle NOT re-executed - data already on disk; just read differently

### Why This Is Valuable

- Static planning uses estimates - if estimate is wrong (stale stats, high filter selectivity), SMJ chosen when BHJ would work
- AQE catches this at runtime - SMJ with two shuffles + sorts replaced by BHJ with zero additional shuffles
- Most impactful when queries have selective filters that weren't modeled at plan time

### SMJ → SHJ Conversion

- `spark.sql.adaptive.maxShuffleHashJoinLocalMapThreshold` (default $0$) -
    - If small side < this threshold, converts SMJ → `ShuffledHashJoin` instead of BHJ
    - SHJ avoids sort but still uses shuffle; useful when broadcast not possible but sort is bottleneck
    - Default $0$ = disabled; set to positive value to enable

---

## Skew Join Detection & Splitting

- __`OptimizeSkewedJoin`__ - AQE rule that detects and handles skewed partitions in `SortMergeJoin`; splits hot partitions into sub-partitions and replicates the corresponding other-side partition

### Skew Detection

- After both `ShuffleQueryStageExec`s complete, `OptimizeSkewedJoin` examines partition sizes -
    - Computes median partition size across all partitions of each side
    - A partition is __skewed__ if ALL of the following -
        - `partitionSize > spark.sql.adaptive.skewJoin.skewedPartitionFactor × medianSize` (default $5×$)
        - `partitionSize > spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` (default $256 MB$)
    - Both conditions required - prevents false positives when median itself is small

### Skew Splitting

- For each skewed partition on side A -
    1. Split into `ceil(partitionSize / targetSize)` sub-partitions
    2. `targetSize = spark.sql.adaptive.advisoryPartitionSizeInBytes` (default $64 MB$)
    3. Sub-partitions read from the original shuffle file using byte range offsets
    4. Corresponding partition from side B replicated once per sub-partition of side A
- Each (sub-partition of A, replicated partition of B) pair processed by one task
- No data movement - just changes read ranges; existing shuffle files reused

### Example

- Partition 42 on left side - $2 GB$ (skewed); median - $100 MB$; `skewedPartitionFactor=5` → threshold $500 MB$ → $2 GB > 500 MB$ → skewed
- `advisoryPartitionSizeInBytes=256 MB` → split into $\lceil 2 GB / 256 MB \rceil = 8$ sub-partitions
- Partition 42 on right side - $80 MB$ (not skewed) → replicated $8$ times
- Result - $8$ balanced tasks instead of $1$ massive task + $7$ tiny tasks

### Limitation - Only SortMergeJoin

- `OptimizeSkewedJoin` only applies to `SortMergeJoinExec`
- `BroadcastHashJoin` - broadcast side is replicated to ALL executors; skew on stream side handled naturally (each executor processes its local partition independently)
- If join was already converted to BHJ by `DynamicJoinSelection`, `OptimizeSkewedJoin` does not apply

### Multi-sided Skew

- AQE handles skew on BOTH sides simultaneously -
    - Skewed partitions on left split independently of skewed partitions on right
    - Non-skewed partitions on each side may be replicated to match the other side's splits
    - Results in a complex but correct task assignment matrix

---

## AQE Configs & Tuning

### Core AQE Switches

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.sql.adaptive.enabled` | `true` | Master AQE switch |
| `spark.sql.adaptive.forceApply` | `false` | Force AQE even for single-stage queries |
| `spark.sql.adaptive.logLevel` | `DEBUG` | Log level for AQE plan changes |

### Partition Coalescing

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.sql.adaptive.advisoryPartitionSizeInBytes` | `64MB` | Target size per coalesced partition |
| `spark.sql.adaptive.coalescePartitions.enabled` | `true` | Enable partition coalescing |
| `spark.sql.adaptive.coalescePartitions.minPartitionSize` | `1MB` | Min size; smaller always coalesced |
| `spark.sql.adaptive.coalescePartitions.minPartitionNum` | `2 × defaultParallelism` | Min partitions after coalescing |
| `spark.sql.adaptive.coalescePartitions.parallelismFirst` | `true` | Prioritize parallelism over target size |

### Join Strategy Switching

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.sql.autoBroadcastJoinThreshold` | `10MB` | Size threshold for SMJ→BHJ conversion |
| `spark.sql.adaptive.maxShuffleHashJoinLocalMapThreshold` | `0` | Size threshold for SMJ→SHJ conversion |

### Skew Join

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.sql.adaptive.skewJoin.enabled` | `true` | Enable skew join optimization |
| `spark.sql.adaptive.skewJoin.skewedPartitionFactor` | `5` | Skew threshold multiplier vs median |
| `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` | `256MB` | Absolute skew threshold |

### Tuning Guidelines

- __Increase `advisoryPartitionSizeInBytes`__ -
    - When post-coalesce partitions still too small (tasks finish in < $1s$)
    - Try $128 MB$ or $256 MB$ for large shuffles
- __Decrease `advisoryPartitionSizeInBytes`__ -
    - When post-coalesce tasks OOM (partitions too large after merge)
- __Increase `autoBroadcastJoinThreshold`__ -
    - When you have reliable stats and know one join side fits in executor memory
    - Risk - stale stats may broadcast unexpectedly large tables
- __Lower `skewedPartitionFactor`__ -
    - When skew is moderate ($2-3×$ median) and causing stragglers
    - May produce false positives on non-skewed data; test carefully
- __Disable specific rules__ -
    - `spark.sql.adaptive.coalescePartitions.enabled=false` - when fixed partition count needed (downstream systems expect specific partition count)
    - `spark.sql.adaptive.skewJoin.enabled=false` - when replication overhead exceeds skew cost (uniform data with occasional outliers)

---

## AQE Limitations

### No Join Reordering

- AQE cannot reorder multi-way joins at runtime
- Join ordering fixed at initial planning time - CBO or left-to-right default
- Even if a completed stage reveals that a different join order would be better, AQE cannot restructure the join tree
- Mitigation - use CBO (`spark.sql.cbo.enabled=true`, `spark.sql.cbo.joinReorder.enabled=true`) for join ordering before AQE takes over for runtime adaptation

### No Scan Re-planning

- File scan strategies (partitioning, predicate pushdown, column pruning) fixed before any execution
- AQE cannot change which files are read or how they are read based on runtime information
- Partition pruning at scan level based on static analysis only

### Single-Stage Query Bypass

- Queries with no shuffle or broadcast stages (single-stage) do not benefit from AQE
- `spark.sql.adaptive.forceApply=true` forces AQE overhead even for these queries; not recommended

### Subquery Handling

- Correlated subqueries and scalar subqueries create separate `AdaptiveSparkPlanExec` instances
- AQE optimizes each subquery independently but cannot coordinate optimization across the main query and subqueries
- Subquery results not used to re-plan the outer query

### Statistics Approximation

- `MapOutputStatistics` uses compressed `MapStatus` sizes - close but not exact
- `HighlyCompressedMapStatus` (high partition count) stores average + bitmap - less accurate for individual partition decisions
- Skew detection may miss or falsely detect skew when partition sizes are estimated less accurately

### Plan Recompilation Overhead

- Each AQE re-planning cycle recompiles physical plan fragments via Janino
- For queries with many shuffle stages, re-planning overhead accumulates
- Queries with $10+$ shuffle stages may see measurable overhead from repeated plan recompilation
- Usually negligible vs shuffle I/O time; only relevant for very fast, low-latency queries

### Non-Deterministic Behavior

- AQE plan changes depend on actual data characteristics at runtime
- Same query on different data (or same data with different distribution) may produce different physical plans
- Makes debugging harder - plan shown in `df.explain()` may not match what actually executed
- To see the final executed plan -
```python
    # After action completes - shows final adapted plan
    df.queryExecution.executedPlan.toString

    # AQE wraps everything - unwrap to see inner plan
    from pyspark.sql.execution import AdaptiveSparkPlanExec
    inner = df.queryExecution.executedPlan
    if hasattr(inner, 'executedPlan'):
        print(inner.executedPlan)   # the final adapted physical plan
```

### Interaction with Caching

- `df.cache()` materializes with the AQE-adapted plan
- Cached data reflects the adapted partition structure (coalesced partitions)
- Subsequent queries reading from cache use the coalesced partition count - may differ from `spark.sql.shuffle.partitions`
- `df.unpersist()` and re-cache if partition count mismatch causes downstream issues

> [!NOTE]
> AQE is not a substitute for good query design. It cannot fix -
> - Cartesian products (no equi-condition)
> - Missing indexes or partition pruning
> - Inefficient UDFs that break codegen
> - Wrong join ordering on many-table queries (CBO needed)
> AQE handles the runtime manifestations of imperfect planning estimates; it cannot compensate for fundamentally wrong query structure.

> [!TIP]
> AQE debugging workflow -
> ```python
> # 1. Check if AQE is active
> spark.conf.get("spark.sql.adaptive.enabled")   # should be "true"
>
> # 2. See what AQE changed
> df.explain("formatted")
> # Look for AQEShuffleRead, BroadcastHashJoin (converted from SMJ), SkewJoin markers
>
> # 3. Check actual partition counts after AQE coalescing
> result.rdd.getNumPartitions()   # after action; reflects coalesced count
>
> # 4. Check Spark UI - SQL tab
> # AdaptiveSparkPlanExec shows plan evolution; each re-planning shown as subgraph
> ```

