# Catalyst Optimizer

- Transforms a user's SQL string or DataFrame DSL expression into an optimized physical execution plan of RDDs
- Emits Java bytecode via Janino
- Everything is a tree of immutable nodes
- Optimisation = applying rule functions that rewrite the tree

## Pipeline

- SQL / DataFrame
    → Unresolved Logical Plan   (symbolic names, no types)
    → Analyzed Logical Plan     (every attribute has `exprId`, `dataType`, `nullable`)
    → Optimized Logical Plan    (same semantics, cheaper tree)
    → Physical Plan             (join algorithms, shuffle placement decided)
    → RDD DAG                   (actual execution)

## TreeNode

- Everything in Catalyst is a `TreeNode` -
    - `TreeNode[T <: TreeNode[T]]` in `org.apache.spark.sql.catalyst.trees`
    - Nodes are immutable - transformations return new trees
    - `transform` / `transformUp` 
        - Does post-order traversal - rewrites children before parents
        - When your rule fires on a node, children are already in final form
        - No need to manually recurse inside a rule.
    - `transformDown` -
        - Does pre-order traversal - use when your rule creates new nodes the same rule should match again 
        - Eg - predicate pushdown creates a new filter position that should be pushed further
- Two subclasses -
    - `Expression` -
        - Produces a value when evaluated against a row
        - Eg - `Literal(42)`, `Add(left, right)`, `AttributeReference("name", StringType)`, `Cast(expr, DoubleType)`
    - `LogicalPlan` - 
        - Represents a relational algebra operation
        - Eg - `Filter(condition, child)`, `Project(exprs, child)`, `Join(left, right, joinType, condition)`

## RuleExecutor

- The engine behind every phase - both `Analyzer` and `Optimizer` extend it
- You define batches of rules -
    - A rule is `LogicalPlan => LogicalPlan` - pattern match, return rewritten tree or return input unchanged
    - Each batch runs either `Once` or to `FixedPoint` (keeps re-applying until the tree stops changing, max $100 iterations$)
    - This design is why Catalyst is extensible — everything is a rule, adding an optimisation = adding a rule

### Phase 0 - Parsing

- ANTLR4-based SQL parser (`SparkSqlParser`) - grammar lives in `SqlBase.g4`
- Parser produces an ANTLR `ParseTree`
- `AstBuilder` walks `ParseTree` (visitor pattern) to produce `Expression` / `LogicalPlan` nodes
- Parser only checks syntax, not semantics
- Output contains only placeholder nodes -
    - `UnresolvedAttribute` — `col("name")`, no type yet
    - `UnresolvedRelation` — `spark.table("users")`, no schema yet
    - `UnresolvedFunction` — `sum(...)`, no return type yet

### Phase 1 - Analysis

- `Analyzer` extends `RuleExecutor[LogicalPlan]`
- `Analyzer` resolves all placeholder nodes against the Catalog -
    - `SessionCatalog` manages tables, views, functions, and partition metadata
    - Delegates to `ExternalCatalog` - `InMemoryCatalog` (local) or `HiveExternalCatalog` (production, backed by Hive Metastore or Glue)
    - Catalog V2 - `CatalogPlugin` interface -
        - Iceberg, Delta, Hudi implement this
        - Accessed via three-part identifiers - `catalog.database.table`
        - Catalyst's analysis phase uses `CatalogManager` to resolve these
- Fixed-point per batch until tree stops changing
 - Key resolution rules -
    - `ResolveRelations` - `UnresolvedRelation("users")` → actual table schema
    - `ResolveReferences` - 
        - `col("name") → AttributeReference("name", StringType, nullable=true, exprId=#42)`
        - The `exprId` uniquely tracks this attribute through every downstream operator
    - `ResolveFunctions` - `sum(...) → Sum(child)` with correct return type
    - `ResolveSubquery` - correlates subqueries to their outer plan
    - `ImplicitTypeCasts` / `TypeCoercion` - inserts `Cast` nodes where types mismatch (e.g. comparing `INT` to `BIGINT`)
- After analysis, every node is resolved - every attribute has a known `exprId`, `dataType`, and `nullable` flag

### Phase 2 - Logical Optimization

- `Optimizer` extends `RuleExecutor[LogicalPlan]`
- Applies rule batches in order
    - Each batch specifies whether rules run `Once` or to `FixedPoint`
- Important logical optimization rules -
    - __Predicate Pushdown__ -
        - Moves `Filter` nodes toward the data source
        - Blocked by - UDFs, implicit casts, non-deterministic expressions (`rand()`, `uuid()`)
        - Uses `transformDown` internally - pushing a filter down creates new positions for the same rule
    - __Column Pruning__ -
        - Removes columns no downstream operator needs
        - Critical for columnar formats - only deserialise required columns
    - __Subquery Decorrelation / Scalar Subquery Elimination__ -
        - Rewrites correlated subqueries into joins
        - A correlated subquery executes once per outer row i.e. $10M outer rows = 10M subquery executions$
        - If you see a slow query with `WHERE x IN (SELECT ...)` - check whether decorrelation fired in `qe.optimizedPlan`
    - __Join Reordering__ -
        - Only with `spark.sql.cbo.joinReorder.enabled=true`
        - Uses a DP algorithm (selinger-style) to find the optimal join order
        - Considers table statistics, selectivity estimates, and join type
        - Without this, Catalyst joins left-to-right as written - `A JOIN B JOIN C` always does `A×B` first regardless of sizes

- Few other optimization rules -
    - Constant Folding - `Literal(3) + Literal(4) → Literal(7)`
    - Boolean Simplification - `TRUE AND expr → expr`
    - Limit Pushdown - Moves `Limit` nodes below `Union` / `Sort` where semantics allow
    - Collapse Projections - Merges two adjacent `Project` nodes into one

### Phase 3 - Physical Planning

- `SparkPlanner` extends `QueryPlanner[SparkPlan]`
- It holds a list of strategies - each strategy is a `PartialFunction[LogicalPlan, Seq[SparkPlan]]`
- The planner -
    - Applies list of strategies to the logical plan in order
    - Each strategy is a `PartialFunction[LogicalPlan, Seq[SparkPlan]]`
    - Collects all resulting `SparkPlan` candidates
    - Runs `prepareForExecution`
    - Picks the best plan using the cost model (when CBO is enabled) or defaults to the first strategy's output

- __Join Selection__ -
    | Condition                             | Algorithm                  | Notes                                     |
    | ------------------------------------- | -------------------------- | ----------------------------------------- |
    | One side `< 10 MB` (estimated)        | Broadcast Hash Join        | No shuffle, fastest by far                |
    | One side buildable, not broadcastable | Shuffle Hash Join          | One shuffle                               |
    | Both sides large, equi-join           | Sort Merge Join            | Two shuffles, default for large joins     |
    | Non-equi join, small side             | Broadcast Nested Loop Join | O(n²) — only viable if small side is tiny |
    | No join condition                     | Cartesian Product          | Almost always a mistake                   |

> [!WARNING]
> The broadcast trap - threshold is compared against Catalyst's estimates, not the actual size.
>
> In case of stale stats, Catalyst can broadcast huge tables, leading to executor OOM.
>
> Fix - `ANALYZE TABLE` regularly, or use `broadcast(df)` hint explicitly when you know better than the optimizer.

- __`EnsureRequirements`__ -
    - All shuffles come from it
    - Walks the physical plan bottom-up -
        - Inserts `ShuffleExchangeExec` wherever a parent's required distribution isn't satisfied
        - Inserts `SortExec` where ordering isn't satisfied
    - Eg - 
        - `SortMergeJoin` requires both children hash-partitioned on the join key and sorted
        - If upstream doesn't satisfy that, `EnsureRequirements` inserts exchange + sort automatically

> [!TIP]
> if you see unexpected extra shuffles in the Spark UI, `qe.sparkPlan` vs `qe.executedPlan` shows you exactly what `EnsureRequirements` added and why.

### Phase 4 - Whole-Stage CodeGen (WSCG)

- A physical plan tree is partitioned into "stages"
- Within a stage, all operators are fused into a single generated Java class
- Primitives stay unboxed in registers
- Operators implement `CodegenSupport` with a push-based model -
    - `doProduce` starts the loop
    - `doConsume` handles each row
- Compiled at runtime by _Janino_ (not javac - in-process, no subprocess, fast)
- If generated class would exceed JVM limits, it falls back to interpreted mode for that stage (`spark.sql.codegen.maxFields`)
- `spark.sql.codegen.wholeStage=true` by default

> [!TIP]
> Set `"codegen"` mode in `df.explain()` to see the generated Java - useful when debugging why a stage is slow.

- __UDFs break codegen__ -
    - `ScalaUDF` doesn't implement `CodegenSupport`
    - The stage boundary splits at the UDF, everything runs interpreted row-by-row
    - This is the primary reason `functions.*` builtins outperform equivalent UDFs - builtins codegen through, UDFs don't.
- __`pandas_udf`__ -
    - Operates on Arrow `ColumnarBatch` not individual rows
    - Still not as fast as native codegen but orders of magnitude faster than row-level UDFs
    - Use this when you can't replace a UDF with a builtin.

## Adaptive Query Execution (AQE)

- AQE re-optimizes the physical plan at runtime using actual shuffle statistics
- AQE inserts `QueryStageExec` boundaries at every shuffle -
    - After each stage completes, real partition statistics are collected and Catalyst re-plans the remainder
    - This chains - stage 1 completes → re-plan → stage 2 → re-plan again
- Key AQE rules -
    - `CoalesceShufflePartitions` - 
        - Merges tiny post-shuffle partitions
        - If a shuffle produces $200$ partitions but most are tiny, AQE coalesces them into fewer partitions
        - Controlled by 
            - `spark.sql.adaptive.advisoryPartitionSizeInBytes` (default $64 MB$)
            - `spark.sql.adaptive.coalescePartitions.minPartitionSize`
    - `OptimizeSkewedJoin` - 
        - Detects hot-key partitions, splits them, replicates the other join side
        - Replaces manual salting
        - Only works for `SortMergeJoin` - 
            - `BroadcastHashJoin` skew is moot because the broadcast side is already on every executor
        - Controlled by - 
            - `spark.sql.adaptive.skewJoin.skewedPartitionFactor` 
            - `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`
    - `DynamicJoinSelection` - 
        - Converts `SortMergeJoin` to `BroadcastHashJoin` at runtime if actual join side turns out small
    - `EliminateJoinToEmptyRelation` - 
        - Short-circuits a join if one side is empty after execution

## Cost-Based Optimizer (CBO)

- Enable with `spark.sql.cbo.enabled=true` (by default,`false`)
- CBO collects and uses table/column statistics (`ANALYZE TABLE ... COMPUTE STATISTICS FOR COLUMNS ...`).
- Statistics maintained -
    - Row count
    - Per-column - `min`, `max`, `ndv` (number of distinct values), `avgColLen`, `maxColLen`, `null count`
    - Histograms (equi-height, equi-width) for skewed distributions
- CBO uses these for -
    - Selectivity estimation of filter predicates
    - Cardinality estimation through joins
    - Join reordering (DP algorithm)
    - Choosing between `BroadcastHashJoin` and `SortMergeJoin`
- Cardinality propagation rules - each logical operator has a `Statistics` method that propagates estimates -
    - `Filter` - `rowCount * selectivity`
    - `Join` (inner) - `left.rowCount * right.rowCount * join_selectivity`
    - `Aggregate` - estimate based on `ndv` of grouping keys
- Without stats, Catalyst uses hardcoded defaults - filter selectivity = 1/3, join selectivity = 0.1, etc

## Extensibility

- `SparkSessionExtensions` (Spark 2.2+) lets you inject custom logic
    ```
    SparkSession.builder()
        .withExtensions { ext =>
        ext.withResolutionRule(_ => MyAnalysisRule)     // inject into Analysis
        ext.withOptimizerRule(_ => MyOptimizerRule)     // inject into Logical Opt
        ext.withStrategy(_ => MyPlanningStrategy)       // inject into Physical Planning
    }
    ```

- UDFs registered via `spark.udf.register` -
    - Wrapped in `ScalaUDF` 
    - Black box, no optimizations pass through
- `Encoder[T]` - 
    - Serializes between JVM objects and `InternalRow`
    - Case classes - derived at compile time
    - Schema mismatch between encoder and Parquet causes runtime error
    - Use `spark.read.schema(schema)` to pin explicitly

## InternalRow - The In-Memory Data Format

- `Row` (public API) boxes every value as `Any` - heap allocation per field, GC pressure
- Catalyst uses `InternalRow` internally to avoid this
- `UnsafeRow` - the primary `InternalRow` implementation in production code paths -
    - Binary format, fixed-width fields (int, long, double) stored inline at known byte offsets
    - Variable-length fields (strings, arrays) stored as (offset, length) pointers into a backing buffer
    - Null bitmap at the start (1 bit per field)
    - Zero-copy to shuffle - the binary layout is already the wire format, no serialization step
    - Byte-comparable on fixed-width fields - sort a `Long` column by comparing $8 bytes$, no deserialization
    - In-place mutable - aggregation hash maps update values without allocation, no GC
    - `UTF8String` - Catalyst's string type, byte-by-byte comparison, no `java.lang.String` object creation
    - `Decimal` - Long-backed for $precision ≤ 18$ (fast), `BigDecimal` for higher precision (slow); Catalyst promotes to `Long` wherever possible

## Data Source V2 (DSV2)

- DSV2 lets sources participate in optimization - not just serve data
- Key DSV2 interfaces -
    - `SupportsPushDownFilters` - 
        - Catalyst passes all filters to the source
        - Source returns the ones it could not handle - those stay as `FilterExec` in the Spark plan
        - The rest are evaluated inside the source (row-group skipping, partition pruning in Parquet/Delta)
    - `SupportsPushDownRequiredColumns` - 
        - Catalyst sends pruned `StructType`; source only reads required columns
        - This is how column pruning reaches into the file format itself
    - `SupportsPartitionPushDown` -
        - Partition predicates pushed all the way to the source before any `InputPartition` is created
- The plan node for DSV2 scans is `DataSourceV2ScanRelation` (logical) → `BatchScanExec` (physical)
- When a DSV2 source produces `ColumnarBatch` (Arrow layout) and downstream operators support columnar execution -
    - WSCG fuses through `BatchScanExec` without ever converting to rows
    - This is the fast path for Parquet reads in Spark 3+.

> [!TIP]
> When debugging why a filter isn't pruning, check whether the source returned it as "could not push down". 
>
> `df.explain("extended")` shows which filters made it into the source vs stayed as `FilterExec`.

## Debugging Catalyst Plans

- `df.explain(mode)` - mode options -
    - `"simple"` - physical plan only
    - `"extended"` - all four plan stages
    - `"codegen"` - shows generated Java code
    - `"cost"` - includes CBO stats per node
    - `"formatted"` - prettified columnar output

- `spark.sql.planChangeLog.level=TRACE` - 
    - Logs every rule application and the tree diff
    - Use when a rule isn't firing when you expect it to
- `QueryExecution` - the object holding all plan stages -
    ```
    val qe = df.queryExecution
    qe.logical                      // unresolved
    qe.analyzed                     // after analysis - types resolved; check for unexpected Cast nodes
    qe.optimizedPlan                // after logical optimization - check filters pushed down, subqueries decorrelated
    qe.sparkPlan                    // before EnsureRequirements - no shuffles yet
    qe.executedPlan                 // final physical plan - shuffles and sorts injected
    ```

- `Spark UI > SQL tab` - shows the executed plan as a DAG with per-node metrics - rows, bytes, time
