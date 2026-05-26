# Cost-Based Optimizer

## CBO Overview

- __Cost-Based Optimizer (CBO)__ - an optional Catalyst optimization layer that uses actual table and column statistics to make better physical planning decisions; sits on top of the rule-based optimizer
- Enabled via `spark.sql.cbo.enabled=true` (default `false`)
- Rule-based optimizer (always active) applies transformations based on plan structure alone - no data knowledge
- CBO augments rule-based optimization with quantitative estimates - row counts, column cardinalities, data sizes - to choose better join algorithms and join orderings

### What CBO Improves

- __Join algorithm selection__ - chooses `BroadcastHashJoin` vs `SortMergeJoin` based on estimated table sizes rather than hardcoded thresholds
- __Join ordering__ - reorders multi-way joins to minimize intermediate result sizes; enabled separately via `spark.sql.cbo.joinReorder.enabled=true`
- __Filter selectivity__ - uses column statistics (NDV, histograms) to estimate how many rows survive a filter; propagates accurate cardinality through the plan tree
- __Aggregate cardinality__ - estimates output row count of `GROUP BY` using NDV of grouping columns

### CBO vs Rule-Based Optimizer

| Aspect | Rule-Based | Cost-Based |
| --- | --- | --- |
| Input | Plan structure only | Plan structure + statistics |
| Join algorithm | Threshold-based (estimated size vs `autoBroadcastJoinThreshold`) | Statistics-driven |
| Join ordering | Left-to-right as written | DP algorithm over statistics |
| Filter selectivity | Hardcoded defaults ($1/3$) | NDV and histogram-based |
| Cost | Always active; zero overhead | Requires `ANALYZE TABLE`; stale stats = wrong decisions |

---

## Table & Column Statistics

- __Table statistics__ - row count and total byte size of a table; used for join algorithm selection and broadcast decisions
- __Column statistics__ - per-column statistical summary; used for selectivity estimation and join ordering

### Table-Level Statistics

- `rowCount: Option[BigInt]` - estimated number of rows
- `sizeInBytes: BigInt` - estimated total uncompressed size in bytes; used to decide broadcast eligibility
- Without `ANALYZE TABLE` - Spark falls back to file size estimates from HDFS/S3 metadata; unreliable for compressed formats (Parquet, ORC actual row count unknown from file size alone)

### Column-Level Statistics

- Collected per column via `ANALYZE TABLE ... FOR COLUMNS`
- `min` - minimum value in column
- `max` - maximum value in column
- `ndv` (Number of Distinct Values) - estimated cardinality of column; key for selectivity and join cardinality estimation
- `avgColLen` - average byte length of a value; used for size estimation of variable-length types (strings, arrays)
- `maxColLen` - maximum byte length
- `nullCount` - number of null values; used for null-aware selectivity estimates
- `histogram` - optional; equi-height histogram for skewed distributions

### Where Statistics Are Stored

- Persistent tables - statistics stored in Hive Metastore (or Glue) as table/column properties
- Temporary views - statistics held in `SessionCatalog` in-memory; lost on session end
- DataFrames created from files - statistics estimated from file metadata; not as accurate as `ANALYZE TABLE`

---

## Histogram Types

- __Histogram__ - a compact representation of column value distribution; enables more accurate selectivity estimates than min/max/NDV alone; especially important for skewed data

### Equi-Width Histogram

- Divides the value range `[min, max]` into `N` buckets of equal width
- Each bucket records the count of values falling in its range
- Problem - for skewed data, most values cluster in a few buckets; empty buckets waste space; busy buckets are imprecise
- Not used by Spark CBO

### Equi-Height Histogram

- __Spark uses equi-height histograms__ (also called equi-depth or quantile histograms)
- Divides the value domain into `N` buckets where each bucket contains approximately the same number of rows
- Each bucket records -
    - `ndv` - number of distinct values in this bucket
    - `hi` - upper bound of this bucket (inclusive)
    - `prevDist` - distinct count up to the previous bucket (for range queries)
- For skewed data - narrow buckets where values are dense; wide buckets where values are sparse; much more accurate selectivity
- Default bucket count - `spark.sql.statistics.histogram.numBins` (default $254$)

### Histogram Example

- Column `age` with values concentrated between $25-35$ and sparse elsewhere -
    - Equi-width: $10$ buckets of width $10$ each; bucket $[20,30]$ has $70%$ of data, others nearly empty
    - Equi-height: $10$ buckets each with $10%$ of data; many narrow buckets in $[25,35]$ range; captures skew accurately

### Height-Balanced Histogram Construction

- Built during `ANALYZE TABLE ... FOR COLUMNS` -
    1. Reads column values and sorts them
    2. Divides sorted values into `numBins` quantile ranges
    3. Records upper bound and NDV per bucket
- Stored as serialized array in Hive Metastore
- Accessed during Logical Optimization - `EstimateStatistics` rules read histogram to compute filter selectivity

---

## ANALYZE TABLE / ANALYZE COLUMN

- __`ANALYZE TABLE`__ - Spark SQL command that computes and persists statistics for a table or partition
- Must be run manually - Spark does NOT automatically update statistics on write (unlike some RDBMS)
- Statistics become stale after DML operations (`INSERT`, `DELETE`, `UPDATE`, partition add/drop)

### Syntax

- SQL -
```sql
    -- Table-level statistics only (row count + size)
    ANALYZE TABLE database.table_name COMPUTE STATISTICS;

    -- Column-level statistics for specific columns
    ANALYZE TABLE database.table_name COMPUTE STATISTICS FOR COLUMNS col1, col2, col3;

    -- All columns
    ANALYZE TABLE database.table_name COMPUTE STATISTICS FOR ALL COLUMNS;

    -- Specific partition
    ANALYZE TABLE database.table_name PARTITION (year=2024, month=01) COMPUTE STATISTICS;

    -- All partitions
    ANALYZE TABLE database.table_name PARTITION (year, month) COMPUTE STATISTICS;
```

- Python -
```python
    spark.sql("ANALYZE TABLE events COMPUTE STATISTICS FOR COLUMNS user_id, event_date, amount")

    # Verify statistics collected
    spark.sql("DESCRIBE EXTENDED events").show(truncate=False)
    spark.sql("DESCRIBE EXTENDED events user_id").show(truncate=False)   # column stats
```

### What ANALYZE Computes

- Table-level scan - reads all data; computes row count and total size; stores in metastore
- Per-column scan - for each requested column, computes min, max, NDV (via HyperLogLog), avgColLen, maxColLen, nullCount
- If `spark.sql.statistics.histogram.enabled=true` (default `false`) - also computes equi-height histogram per column
    - Histograms require a sort per column; significantly more expensive than basic stats

### HyperLogLog for NDV

- Exact NDV computation requires storing all distinct values - infeasible for large tables
- Spark uses __HyperLogLog__ (HLL) algorithm for approximate NDV estimation -
    - Error rate - $\pm 5%$ by default; controlled by `spark.sql.statistics.ndv.maxError` (default $0.05$)
    - Memory - $O(\log \log N)$ per column regardless of cardinality
    - Fast - single pass over data; no sort required
- HLL estimate stored in metastore; used as `ndv` in `ColumnStat`

### ANALYZE Cost

- Full table scan required - expensive for large tables
- Must be run after significant data changes; no automatic invalidation
- For partitioned tables - partition-level stats more accurate; run per-partition analyze for critical tables
- Automation pattern -
```python
    # Analyze after each write in pipeline
    df.write.mode("overwrite").saveAsTable("events")
    spark.sql("ANALYZE TABLE events COMPUTE STATISTICS FOR COLUMNS user_id, event_date")
```

---

## Stats Estimation in Logical Plan

- __Stats estimation__ - how CBO propagates row count and size estimates through logical plan nodes; each operator has rules that compute output statistics from input statistics
- Computed during Logical Optimization phase; stored in `LogicalPlan.stats: Statistics`

### Statistics Object

- `Statistics(sizeInBytes, rowCount, attributeStats, hints)` -
    - `sizeInBytes` - estimated total byte size of output
    - `rowCount: Option[BigInt]` - estimated row count
    - `attributeStats: Map[Attribute, ColumnStat]` - per-column stats for output columns

### Per-Operator Estimation Rules

- __`LogicalRelation` (table scan)__ -
    - Uses metastore statistics directly if available
    - Falls back to file size / default row size estimate if no stats
    - Partition pruning applied first - only statistics for matching partitions used

- __`Filter` selectivity estimation__ -
    - Simple equality `col = value` - selectivity = $1 / ndv(col)$
    - Range `col > value` - selectivity estimated from histogram bucket boundaries
        - Without histogram - uniform distribution assumed; `(max - value) / (max - min)`
        - With histogram - sum of fractions of buckets above `value`
    - `IN (v1, v2, v3)` - selectivity = $min(|values|, ndv) / ndv$
    - `IS NULL` - selectivity = `nullCount / rowCount`
    - `AND` - selectivity = product of individual selectivities (independence assumption)
    - `OR` - selectivity = $1 - (1 - s1) \times (1 - s2)$ (inclusion-exclusion)
    - UDFs and complex expressions - selectivity = hardcoded default ($1/3$ for `!=`; $1/3$ for other predicates)
    - Output `rowCount = inputRowCount × selectivity`
    - Output `ndv` for filtered column updated - `min(ndv, outputRowCount)` (cannot have more distinct values than rows)

- __`Join` cardinality estimation__ -
    - Inner equi-join on single key -
        - `outputRowCount = leftRows × rightRows / max(ndv_left, ndv_right)`
        - Intuition - each left value matches `rightRows / ndv_right` right rows on average
    - Multi-key join - multiply selectivity per key (independence assumption)
    - Left outer join - `max(innerJoinRows, leftRows)` (at least all left rows preserved)
    - Right outer join - `max(innerJoinRows, rightRows)`
    - Full outer join - `max(innerJoinRows, leftRows + rightRows)`
    - Cross join - `leftRows × rightRows`

- __`Aggregate` cardinality estimation__ -
    - Output row count = product of NDV of grouping keys (upper bound on distinct combinations)
    - Capped at input row count - cannot produce more output rows than input rows
    - Example - `GROUP BY city, gender` where `ndv(city)=100`, `ndv(gender)=2` → estimated $200$ output rows

- __`Project`__ - passes through parent statistics; updates `attributeStats` for projected columns
- __`Sort`__ - passes through statistics unchanged; does not affect cardinality
- __`Limit(n)`__ - `rowCount = min(n, parentRowCount)`
- __`Union`__ - `rowCount = sum of children rowCounts`

### Independence Assumption and Its Failures

- CBO assumes columns are statistically independent - correlation between columns not modeled
- Example failure - `WHERE country = 'US' AND state = 'CA'` -
    - True selectivity - very high (most CA rows are US)
    - CBO estimated selectivity - `selectivity(country='US') × selectivity(state='CA')` = much lower than reality
    - CBO overestimates filter output → may choose wrong join algorithm
- Histograms help within a single column but inter-column correlation is not modeled

---

## Join Ordering with CBO

- __Join ordering__ - deciding in what order to execute a multi-way join; different orderings have dramatically different intermediate result sizes
- Enabled via `spark.sql.cbo.joinReorder.enabled=true` (requires `spark.sql.cbo.enabled=true`)
- Without join reorder - Catalyst joins left-to-right as written; `A JOIN B JOIN C` always does `(A JOIN B) JOIN C` regardless of sizes

### Why Join Order Matters

- `A (1M rows) JOIN B (1B rows) JOIN C (100 rows)` -
    - Left-to-right - `(A × B) × C` = $10^{15}$ intermediate rows then filtered
    - Optimal - `(A × C) × B` = $10^8$ intermediate rows (assuming C filters aggressively)
- Wrong join order = orders of magnitude more intermediate data = shuffle amplification = OOM or extremely slow job

### Dynamic Programming Algorithm

- Spark CBO uses a Selinger-style DP algorithm -
    1. Start with individual tables as $N$ single-table "plans" (leaf nodes)
    2. Build all pairwise join combinations - evaluate cost of each
    3. Build all 3-table join combinations using best 2-table plans - evaluate cost
    4. Continue until all $N$ tables joined
    5. Return the minimum-cost complete join plan
- Cost function - `sizeInBytes` of intermediate result (lower = better)
- `spark.sql.cbo.joinReorder.dp.threshold` (default $12$) - max number of tables in join reorder DP; beyond this, DP is too expensive ($2^N$ combinations); falls back to left-to-right order

### Join Graph Considerations

- Only inner joins can be freely reordered - outer joins constrain ordering (preserving null semantics)
- Cross joins (no condition) excluded from reordering - too expensive regardless of order
- Filters pushed before join reordering - more accurate cardinality estimates for reordering decisions

### Example

```sql
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
JOIN regions r ON c.region_id = r.id
```

- Without CBO - joins in written order: `((orders JOIN customers) JOIN products) JOIN regions`
- With CBO and accurate stats -
    - If `regions` is tiny (10 rows) and `products` is small (1000 rows) - reordered to start with smallest tables
    - `((regions JOIN customers) JOIN products) JOIN orders` - smallest intermediate results first
    - Reduces shuffle data volume dramatically

---

## CBO Limitations & Stale Stats

### Stale Statistics

- Statistics NOT automatically updated when data changes -
    - `INSERT INTO table` - new rows not reflected in stats
    - Partition add/drop - partition stats not updated automatically
    - `OVERWRITE` write - old stats remain until `ANALYZE TABLE` run again
- Stale stats lead to wrong decisions -
    - Underestimated table size → Catalyst broadcasts a large table → executor OOM
    - Overestimated filter selectivity → wrong join algorithm chosen → unnecessary sort-merge join
    - Wrong join ordering → huge intermediate results → job slowdown or failure

### Broadcast Trap with Stale Stats

- Most dangerous stale-stats failure mode -
    - Table had $1M$ rows when last analyzed → stats say $10 MB$ → below broadcast threshold
    - Table now has $10B$ rows → actual size $100 GB$ → CBO still thinks it's $10 MB$
    - Catalyst broadcasts $100 GB$ table → executor OOM
- Detection - monitor executor OOM errors with `BroadcastHashJoin` in stack trace
- Mitigation -
    - `spark.sql.autoBroadcastJoinThreshold=-1` to disable auto-broadcast for critical queries
    - Regular `ANALYZE TABLE` after significant data changes
    - Use `broadcast()` hint explicitly when you know the table is small; never rely on auto-broadcast for mission-critical jobs

### Independence Assumption Failures

- CBO assumes column independence within and across predicates
- Correlated columns produce wrong selectivity estimates -
    - `WHERE year = 2024 AND month = 1` - if data only has 2024 data, selectivity underestimated
    - `WHERE city = 'NYC' AND state = 'NY'` - highly correlated; CBO multiplies selectivities incorrectly
- No fix in current Spark CBO - multi-column statistics not supported
- Workaround - use `broadcast()` hints and explicit `repartition()` for known-tricky joins

### NDV Approximation Errors

- HyperLogLog NDV is approximate ($\pm 5%$ error)
- For very low-cardinality columns (boolean, status flags with $2-5$ values) - HLL error negligible
- For high-cardinality columns (UUID, timestamp) - $5%$ error on $10M$ distinct values = $500K$ error in NDV estimate
- Impact on selectivity for equality filter `col = value` -
    - Estimated selectivity = $1 / ndv$
    - NDV error of $5%$ → selectivity error of $5%$ → usually acceptable

### Histogram Limitations

- Histograms disabled by default (`spark.sql.statistics.histogram.enabled=false`) -
    - Require additional sort pass during `ANALYZE TABLE` - expensive
    - Must be explicitly enabled for skewed columns
- Without histograms, range predicate selectivity assumes uniform distribution -
    - For skewed data (power-law distributions), this is catastrophically wrong
    - `WHERE amount > 1000` on a column where $99%$ of values are $< 100$ → CBO estimates $0%$ selectivity; actual is close to $0%$ which is correct; but for `WHERE amount < 1000` → CBO estimates $100%$ selectivity; actual is $99%$ - acceptable
    - For intermediate thresholds on skewed data - estimates can be off by $10-100×$

### AQE as CBO Complement

- __Adaptive Query Execution (AQE)__ partially compensates for CBO limitations -
    - CBO makes decisions at planning time with potentially stale/approximate stats
    - AQE re-plans at runtime using actual shuffle statistics
    - `DynamicJoinSelection` converts `SortMergeJoin` → `BroadcastHashJoin` at runtime if actual size is small
    - `OptimizeSkewedJoin` handles skew that CBO missed during planning
- AQE does NOT fix join ordering - that decision is made at planning time only
- Best practice - use both CBO (for planning) + AQE (for runtime correction)

### When to Use CBO

- Use CBO when -
    - Complex multi-way joins where join ordering matters
    - Tables have significantly different sizes and join algorithm selection is important
    - Query plans have suboptimal join algorithms that AQE cannot fix (join ordering)
    - You can commit to running `ANALYZE TABLE` regularly
- Skip CBO when -
    - Simple queries with 1-2 table joins
    - Tables change frequently and `ANALYZE TABLE` cadence cannot keep up with data changes
    - AQE alone handles the optimization (skew, partition coalescing)

### Key CBO Configurations

| Configuration                                  | Default | What It Controls                                                 |
| ---------------------------------------------- | ------- | ---------------------------------------------------------------- |
| `spark.sql.cbo.enabled`                        | `false` | Master switch for Cost-Based Optimizer (CBO)                     |
| `spark.sql.cbo.joinReorder.enabled`            | `false` | Enables dynamic programming (DP) join reordering                 |
| `spark.sql.cbo.joinReorder.dp.threshold`       | `12`    | Maximum number of tables considered in DP join reordering        |
| `spark.sql.statistics.histogram.enabled`       | `false` | Enables collection of equi-height histograms                     |
| `spark.sql.statistics.histogram.numBins`       | `254`   | Number of histogram buckets                                      |
| `spark.sql.statistics.ndv.maxError`            | `0.05`  | HyperLogLog NDV (number of distinct values) error rate           |
| `spark.sql.statistics.size.autoUpdate.enabled` | `false` | Automatically updates size statistics on write (limited support) |
