# Catalyst Optimizer

## Catalyst Overview

- __Catalyst__ - Spark's extensible query optimization framework; transforms a user's SQL string or DataFrame DSL into an optimized physical execution plan of RDDs; emits Java bytecode via Janino
- Everything is a tree of immutable nodes - optimization = applying rule functions that rewrite the tree
- Four-phase pipeline -
    ```
    SQL / DataFrame DSL
    → Unresolved Logical Plan   (symbolic names, no types)
    → Analyzed Logical Plan     (every attribute resolved; exprId, dataType, nullable assigned)
    → Optimized Logical Plan    (same semantics, cheaper tree)
    → Physical Plan             (join algorithms, shuffle placement decided)
    → RDD DAG                   (actual execution via SparkPlan.execute())
    ```

### Why Catalyst is Designed This Way

- __Rule-based rewriting__ - adding an optimization = adding a rule function; no monolithic optimizer to modify
- __Immutable trees__ - every transformation returns a new tree; safe to run rules concurrently; easy to compare before/after
- __Extensible__ - user code can inject rules at every phase via `SparkSessionExtensions`
- __Unified__ - same framework handles SQL strings, DataFrame DSL, and Dataset typed operations; all converge to the same `LogicalPlan` tree

---

## TreeNode & QueryPlan

- __`TreeNode[T]`__ - the base class for every node in Catalyst; `org.apache.spark.sql.catalyst.trees.TreeNode[T <: TreeNode[T]]`
- Nodes are immutable - transformations return new trees; original unchanged
- Every `TreeNode` has `children: Seq[T]` - the subtree beneath it

### TreeNode Traversal

- __`transform(rule)`__ / __`transformUp(rule)`__ -
    - Post-order (bottom-up) traversal - rewrites children before parents
    - When rule fires on a node, all children are already in final form
    - No need to manually recurse inside a rule
    - Use for most optimization rules - fold constants upward, resolve attributes upward
- __`transformDown(rule)`__ -
    - Pre-order (top-down) traversal - visits parent before children
    - Use when a rule creates new nodes that the same rule should match again
    - Classic use - predicate pushdown creates a new `Filter` position further down; `transformDown` ensures the newly placed filter is also pushed as far as possible
- __`foreach(func)`__ - side-effectful traversal; used for validation and debugging
- __`collect`__ / __`collectLeaves`__ - collect nodes matching a predicate
- __`containsChild(node)`__ - structural containment check

### TreeNode Subclasses

- __`Expression`__ -
    - Produces a value when evaluated against a row
    - Has `dataType`, `nullable`, `foldable` (can be evaluated without row), `deterministic`
    - Examples - `Literal(42)`, `Add(left, right)`, `AttributeReference("name", StringType)`, `Cast(expr, DoubleType)`, `If(pred, trueExpr, falseExpr)`
    - `eval(row: InternalRow): Any` - interpreted evaluation
    - `genCode(ctx: CodegenContext): ExprCode` - code generation path
- __`LogicalPlan`__ -
    - Represents a relational algebra operation
    - Has `output: Seq[Attribute]` - the attributes produced by this plan
    - Has `resolved: Boolean` - true when all attributes and relations are resolved
    - Examples - `Filter(condition, child)`, `Project(exprs, child)`, `Join(left, right, joinType, condition)`, `Aggregate(grouping, agg, child)`

### QueryPlan

- __`QueryPlan[T <: QueryPlan[T]]`__ - extends `TreeNode`; base class for both `LogicalPlan` and `SparkPlan` (physical plan)
- Adds -
    - `output: Seq[Attribute]` - schema of rows produced by this plan node
    - `outputSet: AttributeSet` - set form of output for fast containment checks
    - `inputSet: AttributeSet` - all attributes consumed from children
    - `missingInput: AttributeSet` - attributes referenced but not produced by any child; non-empty = unresolved
    - `references: AttributeSet` - attributes referenced by this node's expressions
    - `schema: StructType` - output as `StructType`; what `df.schema` returns

### Attribute and AttributeReference

- __`Attribute`__ - abstract; represents a column reference
- __`AttributeReference(name, dataType, nullable, metadata)(exprId, qualifier)`__ - concrete attribute -
    - `exprId` - globally unique ID assigned during Analysis; tracks this column through all operators
    - `qualifier` - table/subquery alias; used to resolve ambiguous column names
    - Two `AttributeReference` nodes are the same column iff their `exprId` matches - name alone is insufficient

---

## Unresolved Logical Plan

- __Unresolved Logical Plan__ - the direct output of the parser or DataFrame DSL construction; contains symbolic placeholders with no type information
- Produced by -
    - `SparkSqlParser` (ANTLR4-based; grammar in `SqlBase.g4`) → `AstBuilder` walks parse tree → emits logical nodes
    - DataFrame DSL - `df.filter(col("x") > 1)` builds `Filter(UnresolvedAttribute("x") > Literal(1), child)`
- Parser checks syntax only - not semantics; a query referencing a non-existent table parses successfully

### Placeholder Nodes

- __`UnresolvedAttribute`__ - column reference with only a name; no type, no `exprId`
    - `col("user.name")` → `UnresolvedAttribute(Seq("user", "name"))`
- __`UnresolvedRelation`__ - table or view reference with only a name; no schema
    - `spark.table("events")` → `UnresolvedRelation(Seq("events"))`
- __`UnresolvedFunction`__ - function call with only a name; no return type
    - `sum(col("amount"))` → `UnresolvedFunction("sum", Seq(UnresolvedAttribute("amount")), isDistinct=false)`
- __`UnresolvedStar`__ - `SELECT *`; expands to actual columns during Analysis
- __`UnresolvedAlias`__ - unnamed expression in SELECT; gets a name during Analysis
- __`UnresolvedSubqueryColumnAliases`__ - column aliases on a subquery; resolved once subquery is analyzed

---

## Analyzer

- __`Analyzer`__ - extends `RuleExecutor[LogicalPlan]`; resolves all placeholder nodes against the Catalog; produces the Analyzed Logical Plan
- First phase where semantic correctness is checked - invalid table names, unknown columns, type mismatches all surface here

### Catalog Architecture

- __`SessionCatalog`__ - per-session catalog; manages tables, views, temporary views, functions, and partition metadata
- Delegates to __`ExternalCatalog`__ for persistent metadata -
    - `InMemoryCatalog` - local/testing; no persistence
    - `HiveExternalCatalog` - production; backed by Hive Metastore (or AWS Glue as a drop-in)
- __Catalog V2__ (`CatalogPlugin` interface) - pluggable catalog for modern data lakes -
    - Iceberg, Delta Lake, Hudi implement `CatalogPlugin`
    - Accessed via three-part identifiers - `catalog.database.table`
    - `CatalogManager` resolves catalog names during Analysis
    - Supports fine-grained operations - table versioning, schema evolution, ACID transactions at catalog level

### Key Analysis Rules

- __`ResolveRelations`__ -
    - `UnresolvedRelation("events")` → looks up in `SessionCatalog` → returns `LogicalRelation` with full schema
    - Handles views - expands view definition inline as a subquery
- __`ResolveReferences`__ -
    - `UnresolvedAttribute("name")` → `AttributeReference("name", StringType, nullable=true, exprId=#42)`
    - `exprId` uniquely tracks this attribute through every downstream operator
    - Resolves against the `output` of the child plan
    - Ambiguous references (same name in two join inputs without qualifier) → `AnalysisException`
- __`ResolveFunctions`__ -
    - `UnresolvedFunction("sum", ...)` → looks up in function registry → `Sum(child)` with correct return type
    - Handles built-in functions, registered UDFs, Hive UDFs
- __`ResolveSubquery`__ -
    - Correlates subquery references to outer plan attributes
    - Marks correlated subqueries for decorrelation during optimization
- __`ImplicitTypeCasts`__ / __`TypeCoercion`__ -
    - Inserts `Cast` nodes where types mismatch
    - `INT` compared to `BIGINT` → inserts `Cast(intExpr, LongType)` automatically
    - Follows type precedence rules; promotes to wider type
- __`ResolveAggregateFunctions`__ -
    - Validates aggregate expressions; ensures non-aggregate expressions in SELECT are in GROUP BY
- __`EliminateSubqueryAliases`__ -
    - Removes unnecessary `SubqueryAlias` wrappers after names are resolved

### Analyzer Execution Model

- `Analyzer` runs rule batches to `FixedPoint` (max $100$ iterations per batch) - keeps re-applying until tree stops changing
- Multiple batches - some rules must run after others (resolution before type coercion)
- `AnalysisException` thrown if after fixed-point the plan is still not fully resolved

---

## Resolved Logical Plan

- __Resolved (Analyzed) Logical Plan__ - output of `Analyzer`; every node is semantically complete
- Every `AttributeReference` has -
    - `exprId` - unique, stable identity
    - `dataType` - fully resolved type
    - `nullable` - null propagation flag
- Every `UnresolvedRelation` replaced with actual `LogicalRelation` or `HiveTableRelation`
- Every `UnresolvedFunction` replaced with concrete aggregate or scalar function
- `Cast` nodes inserted for implicit type conversions
- Accessible via `df.queryExecution.analyzed`

### Verification

- `Analyzer` calls `checkAnalysis(plan)` after resolution -
    - Walks resolved plan; checks for `UnresolvedAttribute` / `UnresolvedRelation` still present → `AnalysisException`
    - Validates data types for each operator
    - Checks aggregate expressions
    - This is the gate - if analysis fails, no further processing

---

## Optimizer Rule Batches

- __`Optimizer`__ - extends `RuleExecutor[LogicalPlan]`; applies rule batches to the Analyzed plan; produces Optimized Logical Plan
- Rules are grouped into batches; each batch specifies execution strategy - `Once` or `FixedPoint`
- `FixedPoint` batches run until tree stops changing or hit max iterations (default $100$)
- Order matters - some rules enable others; batches ordered so enabling rules run before dependent rules

### Major Optimizer Batches (in order)

- __Finish Analysis__ (`Once`) - cleanup from Analysis phase; eliminate unneeded nodes
- __Union__ (`Once`) - merge adjacent `Union` nodes
- __Pullup Correlated Expressions__ (`Once`) - prep for subquery decorrelation
- __Subquery__ (`Once`) - subquery decorrelation; scalar/EXISTS subqueries → joins
- __Replace Operators__ (`Once`) - `Distinct` → `Aggregate`; `Intersect` → `Join`
- __Aggregate__ (`FixedPoint`) - aggregate expression rewrites; partial aggregation enablement
- __Operator Optimizations__ (`FixedPoint`) - the main batch; predicate pushdown, column pruning, constant folding, etc.
- __Early Filter and Projection Push-Down__ (`Once`) - push filters/projections into scan before other rewrites
- __Join Reorder__ (`Once`, if CBO enabled) - DP join reordering
- __Remove Redundant Sorts__ (`Once`) - eliminate sorts whose output ordering is not consumed
- __Decimal Optimizations__ (`FixedPoint`) - promote `Decimal` to `Long` where precision allows
- __Object Expressions Optimization__ (`FixedPoint`) - Dataset typed operations; encoder simplification
- __Local Relation__ (`FixedPoint`) - constant-fold entire subtrees to `LocalRelation` (empty/constant tables)
- __Check Cartesian Products__ (`Once`) - warn on unintentional cross joins

---

## RuleExecutor

- __`RuleExecutor[TreeType]`__ - the engine behind `Analyzer` and `Optimizer`; applies ordered batches of rules to a tree

### Rule Definition

- A rule is `TreeType => TreeType` - pattern match on node types, return rewritten tree or return input unchanged
- Rules must be pure functions - no side effects; same input = same output

- Scala -
```scala
    object MyRule extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformDown {
            // Pattern match on specific node types
            case Filter(condition, child) if canPushDown(condition, child) =>
                pushFilterDown(condition, child)
            // Return plan unchanged if rule doesn't apply
        }
    }
```

### Batch Execution

- `RuleExecutor.execute(plan)` -
    1. For each batch in order -
        2. If batch strategy is `Once` - apply each rule once; move to next batch
        3. If batch strategy is `FixedPoint` -
            - Apply all rules in the batch; check if tree changed
            - If changed - repeat; if unchanged - stop (or stop at max iterations)
    4. Return final tree
- `FixedPoint(maxIterations)` - default $100$; prevents infinite loops for rules that may cycle

### Rule Metrics

- `RuleExecutor` tracks which rules fired and how many times - accessible via `RuleExecutor.dumpTimeSpent()`
- `spark.sql.optimizer.excludedRules` - comma-separated list of rule class names to disable
    - Use to debug by disabling specific rules and comparing plan output

---

## Predicate Pushdown

- __Predicate Pushdown__ - moves `Filter` nodes as close to the data source as possible; reduces data read and processed by downstream operators
- One of the highest-impact optimizations - a filter pushed into a Parquet scan eliminates entire row groups before any deserialization

### How It Works

- `PushDownPredicates` rule uses `transformDown` (pre-order) -
    - Visiting `Filter(condition, child)` - tries to push condition into `child`
    - If `child` is `Project` - push filter below project if condition only references project's input columns
    - If `child` is `Join` - split conjunctive predicates; push each clause to left or right child depending on which columns it references
    - If `child` is `Aggregate` - push predicates on non-aggregate columns below the aggregate
    - `transformDown` ensures newly positioned filters are visited again by the same rule for further pushing

### Blocking Conditions

- __UDFs__ - output is opaque; Catalyst cannot reason about nullability or selectivity; filter cannot be pushed past a UDF
- __Non-deterministic expressions__ - `rand()`, `uuid()`, `spark_partition_id()` - pushing changes semantics; blocked
- __Implicit casts__ that change nullability - pushing past a `Cast` that can introduce nulls changes filter semantics
- __Outer join filters__ - a filter on the nullable side of an outer join cannot be pushed below the join (would convert outer to inner)

### Data Source Pushdown

- `SupportsPushDownFilters` (DSv2) - physical planning passes filters to the source -
    - Source inspects each filter and returns the ones it could NOT handle
    - Unhandled filters remain as `FilterExec` in Spark plan
    - Handled filters evaluated inside the source (row group skipping in Parquet, partition pruning in Delta)
- Verify pushdown - `df.explain("extended")` shows which filters appear in `BatchScanExec` vs `FilterExec`

```python
# Check if filter was pushed to source
df.filter(col("event_date") == "2024-01-01").explain("extended")
# Look for filter appearing inside BatchScanExec (pushed) vs FilterExec above scan (not pushed)
```

---

## Column Pruning

- __Column Pruning__ - removes columns that no downstream operator needs from `Project` and scan nodes; critical for columnar formats (Parquet, ORC) where unneeded columns can be completely skipped

### How It Works

- `ColumnPruning` rule walks the plan top-down -
    - For each operator, computes which columns are actually referenced by the operator and all ancestors
    - Inserts `Project` node below operators that produce more columns than needed
    - The inserted `Project` propagates down to the scan; scan only reads required columns

### Impact

- Parquet/ORC are columnar - unneeded columns are not read from disk at all (not just filtered after read)
- Reduces deserialization cost (only decode needed columns)
- Reduces memory pressure (fewer columns in each `UnsafeRow`)
- Reduces shuffle data volume (fewer columns serialized)

### Interaction with Schema Evolution

- `mergeSchema=true` on Parquet - schema merged across files; extra columns in some files filled with null
- Column pruning still works - only requested columns fetched; missing columns produce null without reading

---

## Constant Folding

- __Constant Folding__ - evaluates constant expressions at planning time; replaces with `Literal` nodes; eliminates runtime evaluation cost

### What Gets Folded

- `ConstantFolding` rule calls `expr.eval(EmptyRow)` on any `foldable` expression -
    - `foldable` is true if the expression contains only `Literal`s and deterministic, side-effect-free operations
    - `Literal(3) + Literal(4)` → `Literal(7)` at planning time; never evaluated at runtime
    - `Cast(Literal("2024-01-01"), DateType)` → `Literal(date_days)` at planning time
    - `upper(Literal("hello"))` → `Literal("HELLO")`

### Related Rules

- __`BooleanSimplification`__ - `TRUE AND expr → expr`, `FALSE OR expr → expr`, `NOT NOT expr → expr`
- __`SimplifyConditionals`__ - `IF(TRUE, a, b) → a`, `IF(FALSE, a, b) → b`
- __`NullPropagation`__ - `null + expr → null`, `null == expr → null`; allows null-aware optimizations
- __`LikeSimplification`__ - `col LIKE "abc%"` → `col.startsWith("abc")` which has faster execution path
- __`SimplifyCasts`__ - `Cast(expr, expr.dataType)` → `expr` (no-op cast eliminated)
- __`OptimizeIn`__ - `col IN (1, 2, 3)` → convert to `InSet` (hash-based) when set is large enough for faster membership testing

---

## Boolean Simplification

- __Boolean Simplification__ - rewrites boolean expressions to simpler equivalent forms; reduces expression evaluation work

### Rules Applied

- `TRUE AND expr → expr`
- `FALSE AND expr → FALSE` (short-circuit; eliminates subtree)
- `TRUE OR expr → TRUE` (short-circuit; eliminates subtree)
- `FALSE OR expr → expr`
- `NOT (NOT expr) → expr` (double negation)
- `NOT (a AND b) → NOT a OR NOT b` (De Morgan's law; may enable further pushdowns)
- `NOT (a OR b) → NOT a AND NOT b` (De Morgan's law)
- `expr AND expr → expr` (idempotent AND elimination)
- `expr OR expr → expr` (idempotent OR elimination)

### Why It Matters

- Boolean simplification enables other rules - after simplification, predicate pushdown may find simpler conditions that push further
- Eliminates entire subtrees - `FALSE AND expensive_udf(x)` short-circuits; UDF never evaluated

---

## Subquery Decorrelation

- __Subquery Decorrelation__ - rewrites correlated subqueries into joins; prevents per-outer-row subquery execution

### The Problem

- Correlated subquery - a subquery that references columns from the outer query
```sql
    SELECT * FROM orders o
    WHERE o.amount > (SELECT avg(amount) FROM orders WHERE customer_id = o.customer_id)
```
- Naive execution - for each row in outer `orders`, execute the subquery with that row's `customer_id`
- $10M$ outer rows = $10M$ subquery executions = catastrophic performance

### How Decorrelation Works

- `PullupCorrelatedPredicates` rule lifts correlated predicates out of the subquery
- `RewriteCorrelatedScalarSubquery` / `RewritePredicateSubquery` rewrites to joins -
    - Correlated scalar subquery → left outer join + groupBy
    - `IN (SELECT ...)` subquery → semi join
    - `EXISTS (SELECT ...)` → existence join (semi join without returning values)
    - `NOT IN (SELECT ...)` → anti join
    - `NOT EXISTS (SELECT ...)` → anti join

### Decorrelated Plan

```sql
-- Original correlated subquery
SELECT * FROM orders o
WHERE o.amount > (SELECT avg(amount) FROM orders WHERE customer_id = o.customer_id)

-- After decorrelation
SELECT o.* FROM orders o
JOIN (SELECT customer_id, avg(amount) as avg_amt FROM orders GROUP BY customer_id) avg_orders
ON o.customer_id = avg_orders.customer_id
WHERE o.amount > avg_orders.avg_amt
```

### When Decorrelation Fails

- Some correlated subqueries cannot be decorrelated (complex correlated predicates, multiple levels of correlation)
- Fall back to interpreted `SubqueryExec` - executes as a broadcast nested loop; correct but slow
- Check `df.queryExecution.optimizedPlan` - if you see `ScalarSubquery` node, decorrelation did not fire

---

## Optimized Logical Plan

- __Optimized Logical Plan__ - output of `Optimizer`; semantically identical to Analyzed plan but structurally cheaper
- Accessible via `df.queryExecution.optimizedPlan`
- Key structural differences from Analyzed plan -
    - Filters pushed closer to or into scan nodes
    - Unnecessary `Project` nodes collapsed or removed
    - Constant expressions replaced with `Literal` nodes
    - Correlated subqueries replaced with join nodes
    - `Distinct` replaced with `Aggregate`
    - Empty table short-circuits replaced with `LocalRelation`

### Debugging the Optimized Plan

- Python -
```python
    df.queryExecution.optimizedPlan      # inspect optimized plan
    df.explain("extended")               # prints all stages including optimized

    # Check if predicate was pushed down
    # Look for Filter node position - closer to leaf = better
    # Look for Project node just above scan - column pruning applied
    # Look for absence of correlated ScalarSubquery - decorrelation succeeded
```

- `spark.sql.planChangeLog.level=TRACE` -
    - Logs every rule application and tree diff
    - Use when a rule isn't firing when expected
    - Very verbose; only enable for debugging specific queries

---

## SparkPlanner

- __`SparkPlanner`__ - extends `QueryPlanner[SparkPlan]`; converts Optimized Logical Plan to Physical Plan (`SparkPlan` tree)
- `QueryPlanner` applies strategies (partial functions) to logical nodes to produce physical operator candidates

### Planning Strategies

- `SparkPlanner` holds an ordered list of `Strategy` objects -
    - Each `Strategy` is `PartialFunction[LogicalPlan, Seq[SparkPlan]]`
    - Returns multiple physical plan candidates for a logical node (eg - different join algorithms)
- Strategies applied in order; first strategy that matches a node wins (unless cost model overrides)
- Built-in strategies -
    - `FileSourceStrategy` - `LogicalRelation` → `FileSourceScanExec`
    - `DataSourceV2Strategy` - DSv2 relations → `BatchScanExec`
    - `InMemoryScans` - cached RDDs → `InMemoryTableScanExec`
    - `BasicOperators` - `Filter` → `FilterExec`, `Project` → `ProjectExec`, `Sort` → `SortExec`, `Limit` → `LocalLimitExec`/`GlobalLimitExec`
    - `Aggregation` - `Aggregate` → `HashAggregateExec` or `SortAggregateExec`
    - `JoinSelection` - `Join` → `BroadcastHashJoinExec`, `ShuffledHashJoinExec`, `SortMergeJoinExec`, `BroadcastNestedLoopJoinExec`, `CartesianProductExec`
    - `SparkStrategies.SpecialLimits` - `GlobalLimit` with `Sort` → `TakeOrderedAndProjectExec`

### prepareForExecution

- After strategy application, `QueryExecution.prepareForExecution(plan)` runs preparation rules on the physical plan -
    - __`EnsureRequirements`__ - inserts `ShuffleExchangeExec` and `SortExec` where required distributions/orderings are not satisfied
    - __`CollapseCodegenStages`__ - wraps operators that support codegen in `WholeStageCodegenExec`
    - __`ReuseExchange`__ - deduplicates identical shuffle exchanges; same shuffle output reused by multiple consumers
    - __`ReuseSubquery`__ - deduplicates identical subquery computations

---

## Physical Plan

- __Physical Plan (`SparkPlan`)__ - extends `QueryPlan[SparkPlan]`; describes HOW to compute (physical operators, algorithm choices, shuffle placement)
- `SparkPlan.execute()` returns `RDD[InternalRow]` - this is the bridge from Catalyst to the RDD execution engine

### SparkPlan Properties

- `requiredChildDistribution: Seq[Distribution]` - what data distribution each child must provide -
    - `UnspecifiedDistribution` - no requirement
    - `BroadcastDistribution` - child must be broadcast
    - `HashPartitioning(exprs, n)` - child must be hash-partitioned on `exprs` into `n` partitions
    - `RangePartitioning(ordering, n)` - child must be range-partitioned and sorted
    - `SinglePartition` - all data must be in one partition
- `requiredChildOrdering: Seq[Seq[SortOrder]]` - sort ordering each child must satisfy
- `outputPartitioning: Partitioning` - distribution of this plan's output
- `outputOrdering: Seq[SortOrder]` - sort ordering of this plan's output

### EnsureRequirements

- Walks the physical plan bottom-up -
    - For each operator, checks if each child's `outputPartitioning` satisfies the operator's `requiredChildDistribution`
    - If not satisfied - inserts `ShuffleExchangeExec` between operator and child
    - Checks if child's `outputOrdering` satisfies `requiredChildOrdering`
    - If not satisfied - inserts `SortExec` between operator and child
- This is where ALL shuffles originate - not from the logical plan, not from the optimizer; `EnsureRequirements` decides exactly which shuffles are needed
- Debugging unexpected shuffles -
```python
    # Compare sparkPlan (before EnsureRequirements) vs executedPlan (after)
    df.queryExecution.sparkPlan     # no ShuffleExchangeExec yet
    df.queryExecution.executedPlan  # ShuffleExchangeExec injected by EnsureRequirements
```

### Key Physical Operators

- __`FilterExec(condition, child)`__ - evaluates condition per row; codegen-supported
- __`ProjectExec(projectList, child)`__ - evaluates projection expressions; codegen-supported
- __`HashAggregateExec(grouping, agg, child)`__ - hash-based aggregation; two-phase (partial + final)
- __`SortAggregateExec`__ - sort-based aggregation; used when `HashAggregateExec` not applicable (non-hashable types)
- __`SortExec(order, global, child)`__ - sort operator; uses `ExternalSorter`; supports spill
- __`ShuffleExchangeExec(partitioning, child)`__ - shuffle operator; triggers `ShuffledRDD`
- __`BroadcastExchangeExec(mode, child)`__ - broadcasts child data to all executors
- __`BroadcastHashJoinExec`__ - hash join with broadcast; no shuffle; fastest join
- __`ShuffledHashJoinExec`__ - hash join with shuffle; one side shuffled and hashed
- __`SortMergeJoinExec`__ - sort-merge join; both sides shuffled and sorted; default for large joins
- __`BroadcastNestedLoopJoinExec`__ - nested loop; O(n²); only for non-equi joins with small side
- __`FileSourceScanExec`__ - reads files; handles partition pruning, column pruning, predicate pushdown to source
- __`InMemoryTableScanExec`__ - reads cached `InMemoryRelation`; deserializes from Tungsten binary

---

## Physical Plan Selection

### Join Algorithm Selection

| Condition | Algorithm | Notes |
| --- | --- | --- |
| One side estimated < `spark.sql.autoBroadcastJoinThreshold` ($10 MB$ default) | `BroadcastHashJoinExec` | No shuffle; fastest; threshold vs estimated size not actual |
| One side buildable in memory, not broadcastable, equi-join | `ShuffledHashJoinExec` | One shuffle; build hash table from smaller side |
| Both sides large, equi-join | `SortMergeJoinExec` | Two shuffles; default for large equi-joins |
| Non-equi join, one side small | `BroadcastNestedLoopJoinExec` | O(n²); broadcast small side |
| No join condition | `CartesianProductExec` | Cross product; almost always a mistake |

### The Broadcast Trap

- Threshold compared against Catalyst's row count × average row size estimate - NOT actual data size
- Stale statistics → Catalyst underestimates → broadcasts large table → executor OOM
- Fix -
    - `ANALYZE TABLE t COMPUTE STATISTICS` to refresh stats
    - `broadcast(df)` hint to force broadcast when you know better than optimizer
    - `spark.sql.autoBroadcastJoinThreshold=-1` to disable automatic broadcast if causing OOM

- Python -
```python
    from pyspark.sql.functions import broadcast

    # Force broadcast join
    result = large_df.join(broadcast(small_df), "key")

    # Disable auto-broadcast globally
    spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
```

### Aggregation Strategy Selection

- `HashAggregateExec` preferred - uses hash map; O(n) average case
- `SortAggregateExec` fallback when -
    - Grouping key types not hashable (eg - `MapType`, `ArrayType`)
    - `HashAggregateExec` cannot support the aggregate function (very rare)
- Two-phase aggregation for `HashAggregateExec` -
    - Partial aggregate on map side (within partition) - reduces shuffle data
    - Final aggregate on reduce side - merges partial results
    - `EnsureRequirements` inserts shuffle between partial and final stages

### AQE Plan Changes

- AQE re-optimizes physical plan at runtime using actual statistics from completed shuffle stages
- `CoalesceShufflePartitions` - merges small post-shuffle partitions; avoids $200$ tiny tasks
    - `spark.sql.adaptive.advisoryPartitionSizeInBytes` (default $64 MB$) - target merged partition size
- `OptimizeSkewedJoin` - detects skewed partitions; splits hot partitions; replicates other side
    - Only for `SortMergeJoinExec` - `BroadcastHashJoinExec` skew is moot
    - `spark.sql.adaptive.skewJoin.skewedPartitionFactor` (default $5$) - partition is skewed if size > $5×$ median
    - `spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes` (default $256 MB$)
- `DynamicJoinSelection` - converts `SortMergeJoinExec` → `BroadcastHashJoinExec` at runtime if actual join side is small
- `EliminateJoinToEmptyRelation` - short-circuits join if one side is empty after execution

### Cost-Based Optimizer (CBO)

- Enable via `spark.sql.cbo.enabled=true` (default `false`)
- Requires statistics - `ANALYZE TABLE t COMPUTE STATISTICS FOR COLUMNS col1, col2`
- Statistics maintained -
    - Row count
    - Per-column - `min`, `max`, `ndv` (number of distinct values), `avgColLen`, `maxColLen`, `nullCount`
    - Histograms (equi-height) for skewed distributions
- CBO uses statistics for -
    - Selectivity estimation of filter predicates
    - Cardinality estimation through joins
    - Join reordering via DP algorithm (`spark.sql.cbo.joinReorder.enabled=true`)
    - Choosing between `BroadcastHashJoinExec` and `SortMergeJoinExec`
- Cardinality propagation -
    - `Filter` - `rowCount × selectivity`
    - Inner `Join` - `left.rowCount × right.rowCount × join_selectivity`
    - `Aggregate` - estimated from `ndv` of grouping keys
- Without stats - hardcoded defaults used; filter selectivity = $1/3$, join selectivity = $0.1$; often wrong

### Whole-Stage Code Generation

- `CollapseCodegenStages` wraps chains of codegen-supporting operators in `WholeStageCodegenExec`
- Within a WSCG stage, operators implement push-based model -
    - `doProduce` starts the loop (leaf operator generates the row source loop)
    - `doConsume` handles each row (parent operators add logic inline)
- Generated class compiled by Janino (in-process; no subprocess; fast)
- All operators in stage fused into single tight loop - no virtual dispatch per row; primitives stay unboxed in registers
- `spark.sql.codegen.wholeStage=true` (default)
- Fallback to interpreted mode -
    - `spark.sql.codegen.maxFields` (default $100$) - if schema has more fields than this, codegen disabled for that stage
    - Expression types not supporting codegen (legacy UDFs) split the WSCG stage
- __UDFs break codegen__ -
    - `ScalaUDF` does not implement `CodegenSupport`
    - Stage boundary splits at the UDF; everything outside UDF runs codegen; UDF itself runs interpreted row-by-row
    - Primary reason `functions.*` builtins outperform equivalent UDFs - builtins fuse into WSCG; UDFs do not
    - `pandas_udf` - operates on Arrow `ColumnarBatch`; not as fast as native codegen but orders of magnitude faster than row-level UDFs

```python
# Debug codegen
df.explain("codegen")                   # shows generated Java code
df.queryExecution.executedPlan          # inspect physical plan with codegen stages
```

---

## Catalyst Extension Points

- `SparkSessionExtensions` (Spark 2.2+) - the official API for injecting custom Catalyst logic
- Extensions registered at session creation time; cannot be added to existing `SparkSession`

- Scala -
```scala
    val spark = SparkSession.builder()
        .withExtensions { ext =>
            ext.injectResolutionRule(_ => MyResolutionRule)         // inject into Analysis
            ext.injectAnalyzerRule(_ => MyPostAnalysisRule)         // after Analysis
            ext.injectOptimizerRule(_ => MyOptimizerRule)           // inject into Logical Opt
            ext.injectPlannerStrategy(_ => MyPlanningStrategy)      // inject into Physical Planning
            ext.injectCheckRule(_ => MyCheckRule)                   // inject into checkAnalysis
            ext.injectPostHocResolutionRule(_ => MyPostHocRule)     // after resolution, before optimization
            ext.injectColumnar(_ => MyColumnarRule)                 // inject columnar processing rules
        }
        .getOrCreate()
```

### Extension Injection Points

- `injectResolutionRule` - runs during Analysis; use for resolving custom node types or functions
- `injectPostHocResolutionRule` - runs after Analysis completes; use for cross-node validation
- `injectOptimizerRule` - runs during Logical Optimization; use for custom rewrites
- `injectPlannerStrategy` - runs during Physical Planning; use for custom physical operators
- `injectCheckRule` - runs during `checkAnalysis`; use for semantic validation

---

## Custom Logical Plan Rules

- A custom optimizer rule is a Scala object extending `Rule[LogicalPlan]`
- Applied during the Optimizer phase via `injectOptimizerRule`

### Implementation Pattern

- Scala -
```scala
    import org.apache.spark.sql.catalyst.plans.logical.LogicalPlan
    import org.apache.spark.sql.catalyst.rules.Rule
    import org.apache.spark.sql.catalyst.expressions._

    // Example - rewrite expensive UDF calls to built-in equivalent
    object RewriteMyUdf extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformAllExpressions {
            // Replace calls to registered UDF "my_upper" with built-in Upper
            case ScalaUDF(_, _, Seq(child), _, _, Some("my_upper"), _, _) =>
                Upper(child)
        }
    }

    // Example - constant-fold custom function
    object FoldMyConstantFunction extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformAllExpressions {
            case MyCustomFunction(Literal(v, StringType)) =>
                Literal(myFunction(v.toString))
        }
    }

    // Register
    val spark = SparkSession.builder()
        .withExtensions(_.injectOptimizerRule(_ => RewriteMyUdf))
        .getOrCreate()
```

### Custom Logical Plan Node

- Define a custom `LogicalPlan` node for domain-specific semantics -

- Scala -
```scala
    import org.apache.spark.sql.catalyst.plans.logical.{LogicalPlan, UnaryNode}
    import org.apache.spark.sql.catalyst.expressions.Attribute

    // Custom logical node - eg a "SampleN" operator that samples exactly N rows
    case class SampleN(n: Long, child: LogicalPlan) extends UnaryNode {
        override def output: Seq[Attribute] = child.output
        override protected def withNewChildInternal(newChild: LogicalPlan): LogicalPlan =
            copy(child = newChild)
    }

    // Resolution rule - resolve SampleN (already resolved if only wrapping child)
    object ResolveSampleN extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformUp {
            case SampleN(n, child) if child.resolved => SampleN(n, child)
        }
    }

    // Optimizer rule - push SampleN below Filter if possible
    object PushSampleNBelowFilter extends Rule[LogicalPlan] {
        def apply(plan: LogicalPlan): LogicalPlan = plan transformDown {
            case Filter(cond, SampleN(n, child)) =>
                SampleN(n, Filter(cond, child))    // push filter below sample
        }
    }
```

### Rule Design Principles

- Rules must be idempotent - applying a rule twice should give the same result as applying once
- Rules should be conservative - if pattern doesn't match, return plan unchanged
- Use `transformAllExpressions` for expression-level rewrites; `transformDown`/`transformUp` for plan-level rewrites
- Test rules in isolation with `rule.apply(logicalPlan)` before registering

---

## Custom Physical Plan Strategies

- A custom planning strategy converts custom `LogicalPlan` nodes to physical `SparkPlan` nodes
- Registered via `injectPlannerStrategy`

### Implementation Pattern

- Scala -
```scala
    import org.apache.spark.sql.Strategy
    import org.apache.spark.sql.catalyst.plans.logical.LogicalPlan
    import org.apache.spark.sql.execution.SparkPlan

    // Custom physical operator
    case class SampleNExec(n: Long, child: SparkPlan) extends UnaryExecNode {
        override def output: Seq[Attribute] = child.output

        override protected def doExecute(): RDD[InternalRow] = {
            child.execute().mapPartitions { iter =>
                iter.take(n.toInt)      // sample n rows per partition (simplified)
            }
        }

        override protected def withNewChildInternal(newChild: SparkPlan): SparkPlan =
            copy(child = newChild)
    }

    // Planning strategy - converts SampleN logical → SampleNExec physical
    class SampleNStrategy(spark: SparkSession) extends Strategy {
        def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
            case SampleN(n, child) =>
                SampleNExec(n, planLater(child)) :: Nil     // planLater recurses into child
            case _ =>
                Nil     // don't handle; let other strategies try
        }
    }

    // Register
    val spark = SparkSession.builder()
        .withExtensions { ext =>
            ext.injectResolutionRule(_ => ResolveSampleN)
            ext.injectOptimizerRule(_ => PushSampleNBelowFilter)
            ext.injectPlannerStrategy(spark => new SampleNStrategy(spark))
        }
        .getOrCreate()
```

### `planLater(child)`

- `planLater(logicalPlan)` - tells `SparkPlanner` to recursively plan the child logical node
- Required for strategies that handle one logical node but let the planner handle children
- Without `planLater`, child logical nodes would not be converted to physical nodes

### Codegen Support for Custom Operators

- Custom physical operators can optionally implement `CodegenSupport` for WSCG integration -
```scala
    case class SampleNExec(n: Long, child: SparkPlan)
        extends UnaryExecNode with CodegenSupport {

        override def inputRDDs(): Seq[RDD[InternalRow]] = child.asInstanceOf[CodegenSupport].inputRDDs()

        override protected def doProduce(ctx: CodegenContext): String = {
            child.asInstanceOf[CodegenSupport].produce(ctx, this)
        }

        override def doConsume(ctx: CodegenContext, input: Seq[ExprCode], row: ExprCode): String = {
            val counter = ctx.addMutableState("long", "counter", v => s"$v = 0;")
            s"""
               |if ($counter < ${n}L) {
               |    $counter++;
               |    ${consume(ctx, input)}
               |}
             """.stripMargin
        }
    }
```

### Debugging Custom Extensions

- Verify extension is registered -
```python
    # Python - check if custom rule fired
    df.queryExecution.optimizedPlan     # should show custom node transformed/absent
    df.queryExecution.executedPlan      # should show custom physical operator
    df.explain("extended")              # check all stages
```
- `spark.sql.optimizer.excludedRules` - exclude conflicting built-in rules when testing custom rules
- `spark.sql.planChangeLog.level=TRACE` - see exactly when and how your rule fires
