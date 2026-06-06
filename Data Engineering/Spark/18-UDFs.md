# UDFs & Extensions

## UDF Internals & Catalyst Opacity

- __UDF (User-Defined Function)__ - a user-provided function registered with Spark and callable from DataFrame/SQL operations; wraps arbitrary JVM (Scala/Java) or Python logic as a Catalyst `Expression` node
- Registered via `spark.udf.register("name", func)` or `udf(func)` for inline use
- The fundamental problem with UDFs - they are opaque black boxes to Catalyst; no optimization passes through them

### ScalaUDF / JavaUDF Internals

- `udf(func: A => B)` → wraps in `ScalaUDF(func, returnType, children, inputEncoders, outputEncoder)`
- `ScalaUDF` extends `Expression` but does NOT implement `CodegenSupport`
- Evaluation path - `ScalaUDF.eval(row: InternalRow): Any` -
    1. Deserialize each input column from `InternalRow` → JVM type (via `inputDecoders`)
    2. Call `func(decodedInputs...)`
    3. Serialize return value → `InternalRow` compatible type (via `outputEncoder`)
- This deserialization/serialization happens ONCE PER ROW - the primary overhead

### Python UDF Internals

- `udf(func, returnType)` in PySpark → `PythonUDF` expression node
- Python UDF execution path -
    1. Spark batches rows (default $100$ rows per batch) into a byte stream
    2. Sends byte stream to Python worker process via socket (pickled or Arrow-encoded)
    3. Python worker deserializes, calls `func` per row, serializes results
    4. Sends results back to JVM via socket
    5. JVM deserializes and inserts into `InternalRow`
- Per-batch Python process communication - still extremely high overhead vs JVM operations

### Catalyst Opacity

- Catalyst cannot see inside `ScalaUDF` or `PythonUDF` -
    - Cannot push predicates through UDF - `filter(myUdf(col) > 0)` cannot push the filter below the UDF
    - Cannot prune columns used by UDF - all referenced columns must be materialized
    - Cannot constant-fold UDF results even if inputs are constants - UDF treated as non-deterministic by default
    - Cannot use UDF output statistics for CBO cardinality estimation
- `udf(...).asNondeterministic()` - explicitly marks UDF as non-deterministic; prevents any reordering
- `deterministic: Boolean` field on `ScalaUDF` - default `true` unless overridden; even deterministic UDFs cannot be optimized through

### WSCG Break

- `ScalaUDF` does NOT extend `CodegenSupport` → breaks Whole-Stage CodeGen pipeline
- Any operator containing a `ScalaUDF` cannot be fused into a WSCG stage
- Physical plan splits at UDF boundary -
    - Operators before UDF - fused WSCG stage
    - UDF itself - interpreted
    - Operators after UDF - new WSCG stage if no more UDFs
- Result - two WSCG stages instead of one; inter-stage materialization overhead

### UDF Registration

- Python -
```python
    from pyspark.sql.functions import udf
    from pyspark.sql.types import StringType, IntegerType

    # Inline UDF
    upper_udf = udf(lambda x: x.upper() if x else None, StringType())
    df.select(upper_udf(col("name")))

    # Registered UDF (callable from SQL)
    spark.udf.register("my_upper", lambda x: x.upper() if x else None, StringType())
    spark.sql("SELECT my_upper(name) FROM table")

    # Type hints (Spark 3.0+) - infers return type automatically
    from pyspark.sql.functions import udf
    @udf(returnType=StringType())
    def my_upper(x: str) -> str:
        return x.upper() if x else None
```

- Scala -
```scala
    import org.apache.spark.sql.functions.udf

    val upperUdf = udf((s: String) => if (s != null) s.toUpperCase else null)
    df.select(upperUdf(col("name")))

    // Register for SQL
    spark.udf.register("my_upper", (s: String) => if (s != null) s.toUpperCase else null)
    spark.sql("SELECT my_upper(name) FROM table")
```

### Null Handling in UDFs

- UDFs called even when input is null (unlike built-in expressions which short-circuit)
- Always handle null explicitly in UDF body - do not assume non-null inputs
- `Option[T]` return type in Scala automatically handles null (returns `null` for `None`)

---

## UDAF Internals

- __UDAF (User-Defined Aggregate Function)__ - extends aggregation with custom merge logic; called once per group across multiple rows; produces one output row per group
- Two UDAF APIs -
    - Legacy `UserDefinedAggregateFunction` (deprecated in Spark 3.0) - `Row`-based; slow
    - Modern `Aggregator[IN, BUF, OUT]` - typed; encoder-based; used via `udaf()`

### Aggregator[IN, BUF, OUT]

- `IN` - input row type; `BUF` - intermediate buffer type; `OUT` - output type
- Required methods -
    - `zero: BUF` - initial accumulator value
    - `reduce(b: BUF, a: IN): BUF` - merge one input into accumulator (called per row within partition)
    - `merge(b1: BUF, b2: BUF): BUF` - merge two accumulators across partitions (called during shuffle)
    - `finish(reduction: BUF): OUT` - extract final output from accumulator
    - `bufferEncoder: Encoder[BUF]` - encoder for intermediate buffer (serialization to state)
    - `outputEncoder: Encoder[OUT]` - encoder for final output

- Scala -
```scala
    import org.apache.spark.sql.expressions.Aggregator
    import org.apache.spark.sql.{Encoder, Encoders}

    case class Average(sum: Double, count: Long)

    object MyAverage extends Aggregator[Double, Average, Double] {
        def zero: Average = Average(0.0, 0L)
        def reduce(acc: Average, x: Double): Average = Average(acc.sum + x, acc.count + 1)
        def merge(a1: Average, a2: Average): Average = Average(a1.sum + a2.sum, a1.count + a2.count)
        def finish(acc: Average): Double = acc.sum / acc.count
        def bufferEncoder: Encoder[Average] = Encoders.product
        def outputEncoder: Encoder[Double] = Encoders.scalaDouble
    }

    val myAvg = udaf(MyAverage)
    df.groupBy("category").agg(myAvg(col("value")))
```

### UDAF Physical Execution

- `TypedImperativeAggregate` - the Catalyst expression class wrapping `Aggregator`
- Two-phase aggregation -
    - Partial phase (`reduce`) - runs on each partition; produces `BUF` per group
    - Final phase (`merge` + `finish`) - runs after shuffle; merges partial buffers; emits `OUT`
- Buffer serialized using `bufferEncoder` for shuffle transfer and state store storage
- Still opaque to Catalyst - `TypedImperativeAggregate` cannot be codegen'd through

---

## Pandas UDF (Arrow-based) Internals

- __Pandas UDF__ (vectorized UDF) - executes Python functions on batches of data using Apache Arrow for zero-copy serialization between JVM and Python; orders of magnitude faster than row-level Python UDFs

### Arrow-Based Data Transfer

- Without Pandas UDF - rows serialized via pickle per-row (Python object serialization); $100-1000×$ overhead
- With Pandas UDF - columnar Arrow `RecordBatch` transferred between JVM and Python -
    1. JVM `InternalRow`s accumulated into `ColumnarBatch` (Arrow format)
    2. Arrow buffer shared via socket (or memory map for local process)
    3. Python worker reads Arrow buffer as `pd.DataFrame` or `pd.Series` - ZERO COPY for numeric types
    4. Python function called once on entire batch (not per row)
    5. Result `pd.Series`/`pd.DataFrame` converted back to Arrow; returned to JVM
    6. JVM converts Arrow to `InternalRow`s

### Arrow Batch Size

- `spark.sql.execution.arrow.maxRecordsPerBatch` (default $10000$) - rows per Arrow batch
- Larger batches - better vectorization; more memory per batch
- Smaller batches - less memory; more Python function calls
- Tune based on data width (many columns = smaller batch for same memory)

### Python Worker Pool

- Python workers are long-lived processes reused across batches (not spawned per task)
- `spark.python.worker.reuse=true` (default) - worker reused; avoids Python startup overhead
- Worker processes communicate with JVM executor via socket

---

## Pandas UDF Types (Scalar, Grouped Map, Grouped Agg)

### Scalar Pandas UDF

- __`PandasUDFType.SCALAR`__ - maps `pd.Series` → `pd.Series`; one output per input row; preserves partition structure
- Most common Pandas UDF type; replaces element-wise row UDFs

- Python -
```python
    from pyspark.sql.functions import pandas_udf
    import pandas as pd

    @pandas_udf("double")
    def scale(s: pd.Series) -> pd.Series:
        return (s - s.mean()) / s.std()

    df.select(scale(col("value")))

    # Multi-column input
    @pandas_udf("double")
    def add(a: pd.Series, b: pd.Series) -> pd.Series:
        return a + b

    df.select(add(col("x"), col("y")))
```

- Execution - one batch per partition; `scale(series)` called once per batch; `pd.Series` vectorized

### Iterator of Series Pandas UDF

- `Iterator[pd.Series] → Iterator[pd.Series]` - useful for amortizing expensive setup over entire partition
- Python -
```python
    from typing import Iterator

    @pandas_udf("double")
    def transform_with_model(iterator: Iterator[pd.Series]) -> Iterator[pd.Series]:
        model = load_expensive_model()      # loaded ONCE per partition, not per batch
        for series in iterator:
            yield model.predict(series)

    df.select(transform_with_model(col("features")))
```

### Grouped Map Pandas UDF

- __`PandasUDFType.GROUPED_MAP`__ - `pd.DataFrame → pd.DataFrame`; called once per group; full group available as DataFrame; can change schema
- Used with `df.groupBy(...).applyInPandas(func, schema)` (Spark 3.0+ API)

- Python -
```python
    from pyspark.sql.types import StructType, StructField, StringType, DoubleType

    schema = StructType([
        StructField("category", StringType()),
        StructField("normalized_value", DoubleType())
    ])

    def normalize_group(df: pd.DataFrame) -> pd.DataFrame:
        df["normalized_value"] = (df["value"] - df["value"].mean()) / df["value"].std()
        return df[["category", "normalized_value"]]

    df.groupBy("category").applyInPandas(normalize_group, schema=schema)
```

- Execution -
    1. Data shuffled by group key (one Spark partition per group if `groupBy` not co-partitioned)
    2. All rows for a group collected into one `pd.DataFrame` in Python
    3. `func(group_df)` called
    4. Result `pd.DataFrame` converted back to Arrow; returned to JVM

### Grouped Aggregate Pandas UDF

- __`PandasUDFType.GROUPED_AGG`__ - custom aggregation over a `pd.Series`; called once per group; returns a scalar
- Used with `df.groupBy(...).agg(my_agg_udf(col(...)))`

- Python -
```python
    @pandas_udf("double", PandasUDFType.GROUPED_AGG)
    def weighted_avg(values: pd.Series, weights: pd.Series) -> float:
        return (values * weights).sum() / weights.sum()

    df.groupBy("category").agg(weighted_avg(col("value"), col("weight")))
```

### Pandas UDF vs Scalar UDF Performance

| Aspect | Row UDF | Pandas UDF (Scalar) |
| --- | --- | --- |
| Python function calls | 1 per row | 1 per batch (default 10K rows) |
| Serialization | Pickle per row | Arrow batch (zero-copy numerics) |
| Vectorization | No | Yes (numpy/pandas operations) |
| Typical speedup vs row UDF | 1× (baseline) | $10-100×$ |
| vs built-in expression | $10-100×$ slower | $2-10×$ slower |

---

## UDF vs Built-in Expression Performance

### Why Built-ins Are Faster

- Built-in expressions (`functions.*`) implement `CodegenSupport` → fuse into WSCG stage
- Generated code -
    - No function call overhead (inlined)
    - No JVM object creation (primitives in registers)
    - No serialization/deserialization
    - JIT can optimize entire operator chain as one compilation unit
- UDFs - interpreted; per-row function call; object creation; codec overhead

### Performance Comparison (Approximate)

| Operation | Time per $1M$ rows |
| --- | --- |
| Built-in `upper(col)` | $~10ms$ (codegen) |
| Scala UDF `udf(_.toUpperCase)` | $~150ms$ (JVM, no codegen) |
| Python row UDF | $~2000ms$ (pickle + socket) |
| Pandas UDF `@pandas_udf` | $~50ms$ (Arrow batch) |

### When UDFs Are Acceptable

- Complex business logic impossible to express with built-ins (custom parsing, ML inference, external API calls)
- Logic called infrequently (not in hot path)
- Pandas UDF when Python ecosystem libraries (sklearn, numpy, scipy) genuinely needed
- UDAF for custom aggregations with no built-in equivalent

### UDF Avoidance Patterns

- String manipulation - `regexp_replace`, `regexp_extract`, `substring`, `split`, `concat` cover most cases
- Conditional logic - `when().otherwise()` replaces most conditional UDFs
- Math - `functions.abs`, `sqrt`, `pow`, `log`, `round`, `floor`, `ceil`
- Date/time - `date_add`, `datediff`, `date_format`, `to_timestamp`, `window`
- Array/map - `transform`, `filter`, `aggregate`, `map_keys`, `map_values`, `explode`
- Custom aggregation - `Aggregator[IN, BUF, OUT]` runs in aggregate physical operator without row-level deserialization overhead

---

## Higher-Order Functions Internals

- __Higher-order functions__ - built-in SQL/DataFrame functions that take lambda expressions as arguments and apply them to elements of arrays or maps; introduced in Spark 2.4
- Unlike UDFs - higher-order function lambdas ARE visible to Catalyst; some optimization possible

### Available Higher-Order Functions

- `transform(array, element -> expr)` - maps each element; returns new array
- `filter(array, element -> predicate)` - keeps elements matching predicate
- `aggregate(array, initialValue, (acc, element) -> merge, acc -> finish)` - fold/reduce
- `exists(array, element -> predicate)` - returns boolean; true if any element matches
- `forall(array, element -> predicate)` - returns boolean; true if all elements match
- `zip_with(array1, array2, (e1, e2) -> expr)` - element-wise combine two arrays
- `map_filter(map, (key, value) -> predicate)` - filter map entries
- `map_zip_with(map1, map2, (key, v1, v2) -> expr)` - combine two maps

- Python -
```python
    from pyspark.sql.functions import transform, filter, aggregate, col, lit

    # Double each element in array
    df.select(transform(col("values"), lambda x: x * 2))

    # Filter array elements
    df.select(filter(col("tags"), lambda t: t != ""))

    # Sum array elements
    df.select(aggregate(col("amounts"), lit(0.0),
                        lambda acc, x: acc + x))

    # SQL syntax
    spark.sql("SELECT transform(values, x -> x * 2) FROM t")
    spark.sql("SELECT filter(tags, t -> t != '') FROM t")
```

### Internal Representation

- `transform(array, func)` → `ArrayTransform(array, LambdaFunction(func, args))` in Catalyst logical plan
- `LambdaFunction(function, arguments, hidden)` - represents the lambda expression
- Lambda variables bound as `NamedLambdaVariable` - `AttributeReference`-like but scoped to lambda body
- `LambdaFunction` IS a Catalyst `Expression` - visible to optimizer; predicate pushdown can look through it

### Codegen for Higher-Order Functions

- `ArrayTransform`, `ArrayFilter` etc. implement `CodegenSupport`
- Generated code - a tight Java loop over array elements; lambda body inlined
- Far more efficient than a UDF wrapping a Python loop over array elements
- Example generated code for `transform(array, x -> x * 2)` -
```java
    // Generated inline
    ArrayData result = new GenericArrayData(new Object[input_array.numElements()]);
    for (int i = 0; i < input_array.numElements(); i++) {
        long x = input_array.getLong(i);
        result.update(i, x * 2L);
    }
```

---

## SparkSessionExtensions API

- __`SparkSessionExtensions`__ - the official API for injecting custom Catalyst logic (analysis rules, optimizer rules, physical planning strategies, check rules) into a `SparkSession` without modifying Spark source
- Registered at `SparkSession` creation time via `withExtensions`; cannot modify existing session

### Extension Injection Points

- Scala -
```scala
    val spark = SparkSession.builder()
        .withExtensions { ext =>

            // During Analysis - resolve custom logical nodes
            ext.injectResolutionRule(session => MyResolutionRule(session))

            // After Analysis - validate resolved plan
            ext.injectPostHocResolutionRule(session => MyPostAnalysisValidator(session))

            // Check Analysis - throw AnalysisException for invalid plans
            ext.injectCheckRule(session => MyCheckRule(session))

            // During Logical Optimization - rewrite logical plan
            ext.injectOptimizerRule(session => MyOptimizerRule(session))

            // During Physical Planning - convert logical → physical
            ext.injectPlannerStrategy(session => MyPlannerStrategy(session))

            // Columnar processing rules (Spark 3.1+)
            ext.injectColumnar(session => MyColumnarRule(session))

            // Post-hoc physical plan rules
            ext.injectQueryStagePrepRule(session => MyQueryStagePrepRule(session))
        }
        .getOrCreate()
```

### Extension Loading via ServiceLoader

- Extensions can also be loaded via Java `ServiceLoader` without code changes -
    - `spark.sql.extensions` config - comma-separated list of extension class names
    - Each class must implement `SparkSessionExtensionsProvider` or have a default constructor accepting `SparkSessionExtensions`
```python
    spark = SparkSession.builder() \
        .config("spark.sql.extensions", "com.example.MyExtensions") \
        .getOrCreate()
```

### Extension Class Structure

- Scala -
```scala
    class MyExtensions extends (SparkSessionExtensions => Unit) {
        def apply(ext: SparkSessionExtensions): Unit = {
            ext.injectOptimizerRule(_ => MyOptimizerRule)
            ext.injectPlannerStrategy(session => new MyStrategy(session))
        }
    }
```

---

## Custom Catalyst Rules via Extensions

- A Catalyst rule is `Rule[LogicalPlan]` (for logical) or `Rule[SparkPlan]` (for physical prep)
- Applied by `RuleExecutor` during the appropriate pipeline phase

### Custom Optimizer Rule — Full Example

- Scala -
```scala
    import org.apache.spark.sql.catalyst.rules.Rule
    import org.apache.spark.sql.catalyst.plans.logical._
    import org.apache.spark.sql.catalyst.expressions._

    // Rule - rewrites calls to registered UDF "slow_upper" with built-in Upper
    // Demonstrates how to eliminate UDFs entirely via rule rewriting
    object EliminateSlowUpperUdf extends Rule[LogicalPlan] {
        override def apply(plan: LogicalPlan): LogicalPlan =
            plan transformAllExpressions {
                case ScalaUDF(_, _, Seq(child), _, _, Some("slow_upper"), _, _) =>
                    Upper(child)    // replace with built-in; now codegen-able
            }
    }

    // Rule - push a custom hint down into the plan
    case class MyHintRule(session: SparkSession) extends Rule[LogicalPlan] {
        override def apply(plan: LogicalPlan): LogicalPlan =
            plan transformDown {
                case UnresolvedHint("MY_HINT", params, child) =>
                    // Process hint; return rewritten plan
                    MyHintedNode(params, child)
                case other => other
            }
    }
```

### Custom Analysis Rule

- Scala -
```scala
    import org.apache.spark.sql.catalyst.analysis.UnresolvedRelation

    // Resolution rule - resolve a custom virtual table
    case class ResolveVirtualTables(session: SparkSession) extends Rule[LogicalPlan] {
        override def apply(plan: LogicalPlan): LogicalPlan =
            plan transformUp {
                case UnresolvedRelation(Seq("virtual", tableName)) =>
                    // Return a LogicalPlan for the virtual table
                    generateVirtualTablePlan(tableName, session)
            }
    }
```

### Custom Check Rule

- Scala -
```scala
    import org.apache.spark.sql.AnalysisException

    case class MyCheckRule(session: SparkSession) extends (LogicalPlan => Unit) {
        def apply(plan: LogicalPlan): Unit = plan foreach {
            case j: Join if j.joinType == Cross =>
                throw new AnalysisException(
                    "Cross joins are not allowed in this environment")
            case _ =>
        }
    }
```

### Rule Design Principles

- Rules must be idempotent - applying twice must give same result as applying once
- Return plan unchanged if pattern does not match - not `null`; not throw; just return `plan`
- Use `transformDown` when newly created nodes need to be visited by same rule
- Use `transformUp` (default) when children should be finalized before parent
- Test with `rule.apply(df.queryExecution.analyzed)` before registering

---

## Custom Physical Strategies via Extensions

- A physical planning strategy converts `LogicalPlan` nodes → `SparkPlan` nodes
- Registered via `ext.injectPlannerStrategy`
- Must return `Nil` for logical nodes it does not handle (lets other strategies try)

### Custom Physical Strategy — Full Example

- Scala -
```scala
    import org.apache.spark.sql.Strategy
    import org.apache.spark.sql.catalyst.plans.logical.LogicalPlan
    import org.apache.spark.sql.execution._
    import org.apache.spark.sql.execution.datasources._

    // Custom logical node
    case class IndexScan(
        tableName: String,
        indexColumn: String,
        value: Any,
        output: Seq[Attribute]
    ) extends LeafNode

    // Custom physical operator
    case class IndexScanExec(
        tableName: String,
        indexColumn: String,
        value: Any,
        override val output: Seq[Attribute]
    ) extends LeafExecNode {

        override protected def doExecute(): RDD[InternalRow] = {
            sparkContext.parallelize(
                fetchFromIndex(tableName, indexColumn, value)
                    .map(row => InternalRow.fromSeq(row))
            )
        }
    }

    // Planning strategy
    class IndexScanStrategy(session: SparkSession) extends Strategy {
        override def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {
            case IndexScan(table, col, value, output) =>
                IndexScanExec(table, col, value, output) :: Nil
            case _ =>
                Nil     // return Nil for unhandled nodes; let other strategies try
        }
    }

    // Registration
    val spark = SparkSession.builder()
        .withExtensions { ext =>
            ext.injectResolutionRule(s => ResolveIndexScans(s))
            ext.injectPlannerStrategy(s => new IndexScanStrategy(s))
        }
        .getOrCreate()
```

### planLater for Children

- `planLater(child: LogicalPlan)` - defers planning of child to the planner
- Required when custom operator has child logical nodes that need standard planning -
```scala
    case MyCustomJoin(left, right, condition, output) =>
        MyCustomJoinExec(
            planLater(left),    // standard planning for left child
            planLater(right),   // standard planning for right child
            condition,
            output
        ) :: Nil
```

### Codegen Support for Custom Physical Operators

- Custom `SparkPlan` can optionally implement `CodegenSupport` to participate in WSCG -
```scala
    case class MyFilterExec(condition: Expression, child: SparkPlan)
            extends UnaryExecNode with CodegenSupport {

        override def inputRDDs(): Seq[RDD[InternalRow]] =
            child.asInstanceOf[CodegenSupport].inputRDDs()

        override protected def doProduce(ctx: CodegenContext): String =
            child.asInstanceOf[CodegenSupport].produce(ctx, this)

        override def doConsume(ctx: CodegenContext, input: Seq[ExprCode],
                               row: ExprCode): String = {
            val condCode = condition.genCode(ctx)
            s"""
               |${condCode.code}
               |if (!${condCode.isNull} && ${condCode.value}) {
               |    ${consume(ctx, input)}
               |}
             """.stripMargin
        }

        // Standard doExecute for non-codegen fallback
        override protected def doExecute(): RDD[InternalRow] = {
            val evaluator = condition
            child.execute().mapPartitions { iter =>
                val pred = GeneratePredicate.generate(evaluator, child.output)
                iter.filter(pred.eval)
            }
        }
    }
```

### Testing Custom Extensions

- Python -
```python
    # Verify extension rule fires
    df = spark.sql("SELECT slow_upper(name) FROM users")
    plan = df.queryExecution.optimizedPlan

    # Check UDF was replaced with built-in
    assert "ScalaUDF" not in str(plan), "UDF not eliminated by rule"
    assert "Upper" in str(plan), "Built-in not substituted"

    # Check physical plan uses custom operator
    exec_plan = df.queryExecution.executedPlan
    print(exec_plan)    # should show custom operator
```

> [!NOTE]
> The single most impactful UDF optimization in practice is replacing row-level Python UDFs with Pandas UDFs (`@pandas_udf`). This alone typically produces $10-100×$ speedup by eliminating per-row Python-JVM communication and leveraging Arrow's zero-copy columnar transfer. But replacing any UDF (Scala or Python) with a built-in `functions.*` expression is even better - built-ins codegen through WSCG; even Pandas UDFs break the codegen stage boundary.

> [!TIP]
> Custom extension debugging workflow -
> ```python
> # Step 1 - verify rule fires during optimization
> spark.conf.set("spark.sql.planChangeLog.level", "TRACE")
> df.queryExecution.optimizedPlan   # triggers optimization; rule applications logged
>
> # Step 2 - verify custom physical operator in final plan
> df.queryExecution.executedPlan
>
> # Step 3 - verify WSCG participation (if custom op implements CodegenSupport)
> df.explain("codegen")   # custom op should appear inside *(N) codegen stage
>
> # Step 4 - disable conflicting built-in rules when testing
> spark.conf.set("spark.sql.optimizer.excludedRules",
>                "org.apache.spark.sql.catalyst.optimizer.SomeConflictingRule")
> ```