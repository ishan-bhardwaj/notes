# Structured API

## SparkTypes

- __Spark Types__ - the type system that describes the schema of DataFrames and Datasets; maps between JVM/Python types and Spark's internal binary representation
- Every column in a DataFrame has a `DataType`; every `DataType` maps to a specific binary layout in `UnsafeRow`
- Types are used by -
    - Catalyst optimizer - type inference, expression validation, implicit casting rules
    - Encoder generation - code-generated serializers/deserializers between JVM objects and `UnsafeRow`
    - `DataFrameReader`/`Writer` - schema inference and enforcement

### Numeric Types

- `ByteType` - 1-byte signed integer; JVM `Byte`; stored as 8 bytes in `UnsafeRow` fixed section
- `ShortType` - 2-byte signed integer; JVM `Short`
- `IntegerType` - 4-byte signed integer; JVM `Int`
- `LongType` - 8-byte signed integer; JVM `Long`
- `FloatType` - 4-byte IEEE 754 float; JVM `Float`
- `DoubleType` - 8-byte IEEE 754 double; JVM `Double`
- `DecimalType(precision, scale)` - arbitrary-precision decimal -
    - `precision ≤ 18` → stored as `Long` in fixed section (compact representation)
    - `precision > 18` → stored as `Decimal` object in variable section (slower, more memory)

### String and Binary Types

- `StringType` - UTF-8 encoded byte sequence; stored as offset+length in fixed section, bytes in variable section
    - NOT Java `String` internally - `UTF8String` (Spark's own class wrapping a byte array); avoids UTF-16 overhead of Java strings
- `BinaryType` - raw byte array; same layout as `StringType` in `UnsafeRow`

### Boolean and Date/Time Types

- `BooleanType` - 1-bit value stored as `Long` in fixed section
- `DateType` - days since Unix epoch (`1970-01-01`); stored as `Int`
- `TimestampType` - microseconds since Unix epoch; stored as `Long`
- `TimestampNTZType` (Spark 3.4+) - timestamp without timezone; microseconds since epoch; no timezone conversion

### Complex Types

- `ArrayType(elementType, containsNull)` - variable-length; stored in variable section -
    - Layout - number of elements (8 bytes) + null bitmap + fixed-size slots per element + variable data
    - `containsNull` - whether elements can be null; affects null bitmap allocation
- `MapType(keyType, valueType, valueContainsNull)` - two arrays (keys + values) stored in variable section
- `StructType(fields: Array[StructField])` - nested row; stored as a nested `UnsafeRow` in variable section
    - `StructField(name, dataType, nullable, metadata)` - one field descriptor

### Null Representation

- Every field in `UnsafeRow` has a corresponding null bit in the null bitmap
- Null values have their null bit set; the fixed-section slot is zeroed
- `nullable` flag on `StructField` - hint to optimizer; does NOT prevent nulls at runtime; used for optimization decisions

### Type Hierarchy

```
DataType
├── AtomicType
│   ├── NumericType (ByteType, ShortType, IntegerType, LongType, FloatType, DoubleType, DecimalType)
│   ├── StringType
│   ├── BinaryType
│   ├── BooleanType
│   └── DatetimeType (DateType, TimestampType, TimestampNTZType)
└── ComplexType
├── ArrayType
├── MapType
└── StructType
```

---

## Schema & StructType

- __`StructType`__ - the schema of a DataFrame; an ordered sequence of `StructField`s; itself a `DataType` (enables nested schemas)
- Every DataFrame has a schema - accessible via `df.schema` which returns `StructType`

### StructField

- `StructField(name: String, dataType: DataType, nullable: Boolean, metadata: Metadata)` -
    - `name` - column name; case-sensitive by default (`spark.sql.caseSensitive=false` by default → case-insensitive matching)
    - `dataType` - the column's data type
    - `nullable` - whether the column can contain nulls; used by optimizer for null-elimination rules
    - `metadata` - arbitrary key-value annotations (eg - ML feature metadata, column comments)

### Creating Schemas

- Python -
```python
    from pyspark.sql.types import StructType, StructField, StringType, IntegerType, ArrayType

    schema = StructType([
        StructField("id", IntegerType(), nullable=False),
        StructField("name", StringType(), nullable=True),
        StructField("scores", ArrayType(IntegerType()), nullable=True),
        StructField("address", StructType([
            StructField("city", StringType(), nullable=True),
            StructField("zip", StringType(), nullable=True)
        ]), nullable=True)
    ])

    # DDL string shorthand
    schema = "id INT NOT NULL, name STRING, scores ARRAY<INT>"

    # From JSON schema string
    import json
    schema = StructType.fromJson(json.loads(schema_json_str))
```

- Scala -
```scala
    import org.apache.spark.sql.types._

    val schema = StructType(Array(
        StructField("id", IntegerType, nullable = false),
        StructField("name", StringType, nullable = true),
        StructField("scores", ArrayType(IntegerType), nullable = true),
        StructField("address", StructType(Array(
            StructField("city", StringType, nullable = true),
            StructField("zip", StringType, nullable = true)
        )), nullable = true)
    ))

    // DSL shorthand
    import org.apache.spark.sql.types.DataTypes._
    val schema = new StructType()
        .add("id", IntegerType, nullable = false)
        .add("name", StringType)
        .add("scores", ArrayType(IntegerType))
```

### Schema Operations

- `schema.fields` - array of `StructField`s
- `schema.fieldNames` - array of field name strings
- `schema.apply("fieldName")` - returns `StructField` for field name
- `schema.fieldIndex("fieldName")` - returns 0-based index of field
- `schema.toDDL` - converts to DDL string representation
- `schema.json` / `schema.prettyJson` - JSON representation

### Schema Enforcement vs Inference

- __Schema inference__ - Spark reads a sample of data and infers types; convenient but -
    - Slow for large files (requires a read pass)
    - Inferred types may be wrong (eg - integer column with nulls inferred as `LongType`)
    - Inferred schema may change between runs if sample data changes
- __Explicit schema__ - always specify schema for production code -
    - Faster reads (no inference pass)
    - Type safety guaranteed
    - Catches data quality issues early (file doesn't match expected schema)

> [!TIP]
> In production, always define schemas explicitly. Schema inference is acceptable only for exploratory analysis. A schema mismatch discovered at query time is far more expensive than one caught at read time.

---

## Row & UnsafeRow

### Row (External API)

- __`Row`__ - the external, user-facing representation of a DataFrame row; a sequence of `Any` values accessible by index or field name
- `Row` is a trait; concrete implementation is `GenericRow` (array-backed) or `GenericRowWithSchema` (array + schema)
- Used when working with DataFrames via `.collect()`, `.take()`, user-defined functions returning rows, etc.

### Row Access

- Python -
```python
    row = Row(id=1, name="Alice", score=95.5)
    row["name"]         # field access by name
    row[1]              # field access by index
    row.name            # attribute access (Python only)
    row.asDict()        # convert to dict

    # From collect()
    rows = df.collect()         # List[Row]
    for row in rows:
        print(row["id"], row["name"])
```

- Scala -
```scala
    val row: Row = Row(1, "Alice", 95.5)
    row.getInt(0)           // typed access by index
    row.getString(1)        // typed access
    row.getAs[Double](2)    // generic typed access
    row.getAs[String]("name")   // access by field name (requires schema)
    row.schema              // Option[StructType]
```

### Row vs UnsafeRow

- `Row` (external) - JVM object; field access via `Array[Any]`; boxing overhead; GC pressure; used in user code
- `UnsafeRow` (internal) - binary memory region; field access via direct memory reads at fixed offsets; zero boxing; used inside execution engine
- Conversion path -
    - `df.collect()` → executor serializes `UnsafeRow` to bytes → Driver deserializes to `GenericRow`
    - UDF input → `UnsafeRow` decoded to JVM objects before passing to UDF
    - UDF output → JVM objects encoded back to `UnsafeRow`
- This conversion is the primary overhead of UDFs - the boundary between Tungsten binary world and JVM object world

### UnsafeRow (Internal)

- Covered in detail in Tungsten section
- Key points for Structured API context -
    - All physical operators pass `UnsafeRow` - `Filter`, `Project`, `SortMergeJoin`, `HashAggregate` operate on binary directly
    - `df.schema` defines the layout - field `i` accessed at `baseOffset + 8 + i * 8` (fixed section)
    - `UnsafeRow.copy()` required when row must outlive iteration - operators reuse same object

---

## Encoders

- __`Encoder[T]`__ - the bridge between JVM type `T` and Spark's internal `UnsafeRow` binary format; tells Spark how to serialize a `T` to binary and deserialize binary back to `T`
- Every `Dataset[T]` has an `Encoder[T]`
- `DataFrame` is `Dataset[Row]` - uses `RowEncoder`

### Encoder Types

- __`ExpressionEncoder[T]`__ - the only concrete implementation; generates code at runtime using Catalyst expressions to perform serialization/deserialization
- Generated code is compiled to bytecode and cached - first use has JIT compilation cost; subsequent uses are fast

### Built-in Encoders

- Scala -
```scala
    import org.apache.spark.sql.Encoders

    Encoders.INT            // Int ↔ IntegerType
    Encoders.STRING         // String ↔ StringType
    Encoders.LONG           // Long ↔ LongType
    Encoders.product[MyCaseClass]   // case class ↔ StructType (auto-derived)
    Encoders.bean[MyBean]           // Java bean ↔ StructType
    Encoders.kryo[MyClass]          // arbitrary class via Kryo serialization (opaque BinaryType)
    Encoders.javaSerialization[T]   // arbitrary class via Java serialization (BinaryType)
```

- Python -
    - Python DataFrames always use `Row`-based encoding internally
    - No explicit `Encoder` API in PySpark - schema inferred from data or specified via `StructType`

### ExpressionEncoder Internals

- `ExpressionEncoder[T]` contains two expression trees -
    - `serializer: Seq[NamedExpression]` - expressions that extract fields from `T` and produce `InternalRow`
    - `deserializer: Expression` - expression that reconstructs `T` from `InternalRow`
- For a case class `case class Person(id: Int, name: String)` -
    - Serializer extracts `id` (via getter reflection) → `IntegerType` field; `name` → `StringType` field
    - Deserializer reads int field at index 0, string field at index 1, calls `Person(id, name)` constructor
- Code generation compiles these expression trees to bytecode via Janino
- `ResolvedEncoder` - encoder with schema bound to a specific `StructType`; ready for use

### Encoder vs Schema

- `Encoder` knows both the schema AND how to convert between JVM type and binary
- `StructType` (schema) only knows the column names and types - no conversion logic
- `df.as[T]` - applies encoder `T` to a DataFrame; reinterprets binary rows as `T` objects
    - Does NOT copy or convert data - just changes how rows are interpreted when an action materializes them

### Kryo and Java Serialization Encoders

- `Encoders.kryo[T]` - serializes entire `T` object using Kryo; stored as opaque `BinaryType` column
    - Loses column-level schema; cannot use SQL expressions on fields; entire object opaque to optimizer
    - Use only as last resort for types that cannot be expressed as `StructType`
- `Encoders.javaSerialization[T]` - same but uses Java serialization; slower; larger output

---

## DataFrame Internals

- __`DataFrame`__ - `Dataset[Row]`; a distributed collection of `Row` objects with a schema; the primary Structured API type
- Not a specialized class - `type DataFrame = Dataset[Row]` in Scala
- Built on three layers -
    - __Logical plan__ - `LogicalPlan` tree describing what to compute (no physical decisions)
    - __Physical plan__ - `SparkPlan` tree describing how to compute (after Catalyst optimization)
    - __Execution__ - `RDD[InternalRow]` produced by the physical plan; uses `UnsafeRow` throughout

### DataFrame Creation

- Python -
```python
    # From collection
    df = spark.createDataFrame([(1, "Alice"), (2, "Bob")], ["id", "name"])
    df = spark.createDataFrame(data, schema=schema)

    # From RDD
    df = rdd.toDF(["id", "name"])
    df = spark.createDataFrame(rdd, schema)

    # From file
    df = spark.read.parquet("hdfs://path/")
    df = spark.read.json("hdfs://path/")

    # From existing DataFrame
    df2 = df.select("id", "name").filter(col("id") > 1)
```

- Scala -
```scala
    // From collection
    val df = spark.createDataFrame(Seq((1, "Alice"), (2, "Bob"))).toDF("id", "name")
    val df = spark.createDataFrame(data, schema)

    // From RDD
    val df = rdd.toDF("id", "name")

    // From file
    val df = spark.read.parquet("hdfs://path/")
```

### Logical Plan

- Every DataFrame operation builds a new `LogicalPlan` node on top of the previous plan
- Plans are lazy - no execution until an action (`.collect()`, `.count()`, `.write`)
- Key logical plan nodes -
    - `Project(projectList, child)` - SELECT
    - `Filter(condition, child)` - WHERE
    - `Join(left, right, joinType, condition)` - JOIN
    - `Aggregate(groupingExprs, aggregateExprs, child)` - GROUP BY + aggregation
    - `Sort(order, global, child)` - ORDER BY
    - `Limit(limitExpr, child)` - LIMIT
    - `Union(children)` - UNION ALL
    - `Distinct(child)` - DISTINCT (becomes `Aggregate` in optimizer)
    - `SubqueryAlias(name, child)` - aliased subquery
    - `LogicalRelation(relation, output, catalogTable)` - leaf node for file/table sources

### Catalyst Pipeline

- Action triggers `QueryExecution` -
    1. __Analysis__ - `Analyzer` resolves column names, types, functions against catalog
    2. __Logical optimization__ - `Optimizer` applies rule-based rewrites (predicate pushdown, column pruning, constant folding, etc.)
    3. __Physical planning__ - `SparkPlanner` converts logical plan to `SparkPlan` (physical operators)
    4. __Code generation__ - `WholeStageCodegen` fuses operators into single generated Java class
    5. __Execution__ - physical plan's `execute()` returns `RDD[InternalRow]`

### QueryExecution

- `df.queryExecution` - exposes all plan stages for debugging -
```python
    df.queryExecution.logical          # unresolved logical plan
    df.queryExecution.analyzed         # after Analysis
    df.queryExecution.optimizedPlan    # after Optimizer
    df.queryExecution.sparkPlan        # physical plan (before codegen)
    df.queryExecution.executedPlan     # final physical plan (with codegen)
    df.explain(extended=True)          # prints all plan stages
```

### DataFrame Operations - Transformation Semantics

- Every transformation returns a new `DataFrame` with a new logical plan node
- Plans are immutable - transformations do not modify existing plans
- The entire plan tree is reoptimized when an action triggers execution
- Multiple actions on the same DataFrame re-execute from scratch unless the DataFrame is `persist()`ed

---

## Dataset Internals

- __`Dataset[T]`__ - a distributed collection of objects of type `T`; combines type safety of RDDs with the optimization of DataFrames
- `DataFrame` is `Dataset[Row]` - untyped; `Dataset[Person]` is typed
- JVM only - Python and R have only DataFrames (untyped)

### Dataset vs DataFrame vs RDD

| Aspect | RDD[T] | DataFrame (Dataset[Row]) | Dataset[T] |
| --- | --- | --- | --- |
| Type safety | Compile-time | None | Compile-time |
| Catalyst optimization | No | Yes | Yes |
| Tungsten binary format | No | Yes | Yes (with encoder) |
| Schema | No | Yes | Yes |
| Language | All | All | JVM only |
| UDF overhead | None (native) | High (Row↔object) | Low (encoded) |

### Dataset Creation

- Scala -
```scala
    case class Person(id: Int, name: String)

    // From DataFrame
    val ds: Dataset[Person] = df.as[Person]         // applies encoder

    // From collection
    val ds = spark.createDataset(Seq(Person(1, "Alice"), Person(2, "Bob")))

    // From RDD
    val ds = rdd.toDS()     // requires implicit Encoder[T] in scope
```

### Typed vs Untyped Operations

- __Typed operations__ - use JVM types; Catalyst cannot optimize inside the lambda -
```scala
    ds.filter(p => p.id > 1)            // typed; lambda opaque to Catalyst
    ds.map(p => p.name.toUpperCase)     // typed; returns Dataset[String]
```
- __Untyped operations__ (Column-based) - use `Column` expressions; fully visible to Catalyst -
```scala
    ds.filter(col("id") > 1)            // untyped; Catalyst can push down
    ds.select(col("name"))              // returns DataFrame (Dataset[Row])
```
- __Mixed__ - `ds.where(col("id") > 1).map(p => p.name)` - filter optimized, map not

### Encoder Interaction at Runtime

- `ds.map(f)` -
    1. Physical plan deserializes `UnsafeRow` → `T` using `encoder.deserializer`
    2. Calls `f(t: T)` → produces `U`
    3. Serializes `U` → `UnsafeRow` using output encoder
- Deserialization/serialization happens once per record at the typed boundary
- Avoid repeatedly crossing the typed/untyped boundary in tight loops - each crossing pays encode/decode cost

### Dataset Caching

- `ds.cache()` - caches as `UnsafeRow` binary (Tungsten format) regardless of type parameter
- Accessing cached `Dataset[T]` deserializes from binary on each read
- `ds.persist(StorageLevel.MEMORY_ONLY_SER)` - already serialized; no additional serialization cost

---

## Column & Expression

- __`Column`__ - the user-facing API for referring to, transforming, and composing DataFrame columns; wraps a Catalyst `Expression`
- Every column operation builds a tree of `Expression` nodes; the tree is what Catalyst optimizes

### Column Creation

- Python -
```python
    from pyspark.sql.functions import col, lit, expr

    col("id")                   # reference to column named "id"
    df["id"]                    # same; subscript access on DataFrame
    df.id                       # same; attribute access
    lit(42)                     # literal value → Column wrapping Literal(42)
    expr("id + 1")              # parse SQL expression string → Column
    col("address.city")         # nested field access
    col("array_col")[0]         # array element access
    col("map_col")["key"]       # map value access
```

- Scala -
```scala
    import org.apache.spark.sql.functions._

    col("id")
    $"id"                       // Scala DSL shorthand; requires import spark.implicits._
    df("id")                    // DataFrame-scoped column reference
    lit(42)
    expr("id + 1")
    col("address.city")         // nested field
```

### Column Operations

- Arithmetic - `col("a") + col("b")`, `col("price") * lit(1.1)`, `col("x") / col("y")`
- Comparison - `col("age") > 18`, `col("name") === "Alice"`, `col("id").isin(1, 2, 3)`
- Logical - `col("a") && col("b")`, `col("x") || col("y")`, `!col("flag")`
- String - `col("name").startsWith("A")`, `col("name").contains("son")`, `col("name").like("%son%")`
- Null handling - `col("a").isNull`, `col("a").isNotNull`, `col("a").eqNullSafe(col("b"))`
- Casting - `col("id").cast(StringType)`, `col("str_id").cast("int")`
- Aliasing - `col("id").alias("user_id")`, `col("price").as("cost")`
- Sort - `col("id").asc`, `col("name").desc`, `col("id").asc_nulls_last`

### Catalyst Expression Tree

- `Column` is a thin wrapper over `Expression` - `Column(expr: Expression)`
- Expression tree for `(col("price") * lit(1.1)).alias("adjusted_price")` -
    
    ```
    Alias("adjusted_price")
    └── Multiply
    ├── AttributeReference("price", DoubleType)
    └── Literal(1.1, DoubleType)
    ```

- `Expression` types -
    - `Literal(value, dataType)` - constant value
    - `AttributeReference(name, dataType, nullable)` - column reference
    - `Alias(child, name)` - rename expression
    - `BinaryArithmetic` subclasses - `Add`, `Subtract`, `Multiply`, `Divide`
    - `BinaryComparison` subclasses - `EqualTo`, `GreaterThan`, `LessThan`, `In`
    - `BinaryLogic` subclasses - `And`, `Or`
    - `Cast(child, dataType)` - type casting
    - `If(predicate, trueValue, falseValue)` - conditional
    - `CaseWhen(branches, elseValue)` - CASE WHEN
    - `ScalaUDF` / `PythonUDF` - user-defined function wrapper

### Expression Evaluation

- Expressions evaluated via `expr.eval(internalRow)` - returns JVM value
- Code generation - `WholeStageCodegen` generates Java source for expression trees; no virtual dispatch per row
- Interpreted mode fallback - for unsupported expressions; much slower; avoid in hot paths

### Functions Library

- `org.apache.spark.sql.functions` (Scala/Java) / `pyspark.sql.functions` (Python) - hundreds of built-in functions
- All return `Column` - fully visible to Catalyst optimizer
- Categories -
    - Aggregate - `sum`, `avg`, `count`, `min`, `max`, `collect_list`, `collect_set`, `approx_count_distinct`
    - String - `concat`, `substring`, `trim`, `upper`, `lower`, `regexp_replace`, `regexp_extract`, `split`
    - Date/Time - `current_date`, `current_timestamp`, `date_add`, `datediff`, `date_format`, `to_timestamp`
    - Array - `array_contains`, `explode`, `flatten`, `array_distinct`, `array_union`, `array_intersect`, `transform`, `filter`, `aggregate`
    - Map - `map_keys`, `map_values`, `map_from_entries`, `explode`
    - Math - `abs`, `ceil`, `floor`, `round`, `sqrt`, `pow`, `log`
    - Window - `rank`, `dense_rank`, `row_number`, `lag`, `lead`, `first`, `last`
    - Misc - `when`, `coalesce`, `isnull`, `isnan`, `monotonically_increasing_id`, `spark_partition_id`

---

## DataFrameReader

- __`DataFrameReader`__ - the entry point for reading data into DataFrames; accessed via `spark.read`
- Fluent API - methods return `this` for chaining; actual read triggered by terminal method (`csv`, `parquet`, `json`, etc.)

### Core API

- Python -
```python
    df = spark.read \
        .format("parquet") \
        .option("mergeSchema", "true") \
        .schema(schema) \
        .load("hdfs://path/")

    # Format-specific shortcuts
    df = spark.read.parquet("hdfs://path/")
    df = spark.read.json("hdfs://path/")
    df = spark.read.csv("hdfs://path/", header=True, inferSchema=True)
    df = spark.read.orc("hdfs://path/")
    df = spark.read.text("hdfs://path/")                # RDD-like; one column "value"
    df = spark.read.table("database.table_name")        # Hive/catalog table
    df = spark.read.jdbc(url, table, properties=props)  # JDBC source
```

- Scala -
```scala
    val df = spark.read
        .format("parquet")
        .option("mergeSchema", "true")
        .schema(schema)
        .load("hdfs://path/")

    val df = spark.read.parquet("hdfs://path/")
    val df = spark.read.option("header", "true").csv("hdfs://path/")
    val df = spark.read.table("database.table_name")
```

### Options - Common and Format-Specific

- CSV options -
    - `header` (default `false`) - first row is header
    - `inferSchema` (default `false`) - infer types from data (slow; avoid in production)
    - `delimiter` (default `,`) - field delimiter
    - `quote` (default `"`) - quote character
    - `escape` (default `\`) - escape character
    - `nullValue` - string to treat as null
    - `nanValue` - string to treat as NaN
    - `dateFormat` - pattern for parsing dates
    - `timestampFormat` - pattern for parsing timestamps
    - `mode` - `PERMISSIVE` (default; corrupt rows go to `_corrupt_record`), `DROPMALFORMED`, `FAILFAST`
- JSON options -
    - `multiLine` (default `false`) - single JSON object spans multiple lines
    - `allowComments` (default `false`) - allow `//` and `/* */` comments
    - `allowUnquotedFieldNames` (default `false`)
    - `timestampFormat` - pattern for parsing timestamps
    - `mode` - same as CSV
- Parquet options -
    - `mergeSchema` (default `spark.sql.parquet.mergeSchema`, usually `false`) - merge schemas across files
    - `datetimeRebaseModeInRead` - handle legacy datetime in Parquet
- JDBC options -
    - `url` - JDBC connection URL
    - `dbtable` - table name or subquery
    - `numPartitions` - number of parallel read partitions
    - `partitionColumn` - column to partition on (must be numeric, date, or timestamp)
    - `lowerBound` / `upperBound` - range for partition column
    - `fetchsize` - rows fetched per round trip (default $0$ = driver default)
    - `driver` - JDBC driver class name

### Schema Specification

- `reader.schema(schema)` - explicit schema; skips inference; fastest; required for production
- `reader.schema("id INT, name STRING")` - DDL string shorthand
- Without schema - inference triggered; reads data twice for some formats

### Multi-Path Read

- `spark.read.parquet("path1", "path2", "path3")` - reads multiple paths as one DataFrame
- `spark.read.parquet("hdfs://path/year=2023/", "hdfs://path/year=2024/")` - explicit partitions

### DataSource V2 API

- Modern data sources implement `DataSourceV2` (Spark 3.0+) -
    - Supports batch and streaming reads in same interface
    - Column pruning and filter pushdown negotiated between Spark and source at planning time
    - Supports partition pruning without reading all files
- Legacy `DataSourceV1` (Hadoop `InputFormat` based) - still used by some older connectors

---

## DataFrameWriter

- __`DataFrameWriter`__ - the entry point for writing DataFrames to storage; accessed via `df.write`
- Fluent API - terminal methods (`parquet`, `csv`, `json`, `save`, `saveAsTable`, `insertInto`) trigger the write

### Core API

- Python -
```python
    df.write \
        .format("parquet") \
        .mode("overwrite") \
        .option("compression", "snappy") \
        .partitionBy("year", "month") \
        .save("hdfs://output/path/")

    # Format-specific shortcuts
    df.write.parquet("hdfs://output/", mode="overwrite")
    df.write.csv("hdfs://output/", header=True, mode="append")
    df.write.json("hdfs://output/")
    df.write.orc("hdfs://output/")
    df.write.saveAsTable("database.table_name")     # write to Hive/catalog
    df.write.insertInto("table_name")               # append to existing table
    df.write.jdbc(url, table, mode="overwrite", properties=props)
```

- Scala -
```scala
    df.write
        .format("parquet")
        .mode(SaveMode.Overwrite)
        .option("compression", "snappy")
        .partitionBy("year", "month")
        .save("hdfs://output/path/")

    df.write.saveAsTable("database.table_name")
```

### Save Modes

- `SaveMode.ErrorIfExists` (default) - throw exception if output path/table exists
- `SaveMode.Append` - append to existing data; adds new files
- `SaveMode.Overwrite` - delete existing data and write new; for partitioned writes, only overwrites affected partitions if `spark.sql.sources.partitionOverwriteMode=dynamic`
- `SaveMode.Ignore` - no-op if output already exists

### Partitioning

- `partitionBy("col1", "col2")` - writes data partitioned into subdirectories -
    ```
    output/
    ├── year=2023/
    │   ├── month=01/
    │   │   ├── part-00000.parquet
    │   │   └── part-00001.parquet
    │   └── month=02/
    └── year=2024/
    ```

- Partition columns excluded from data files; stored in directory names
- Partition discovery on read - Spark automatically detects and reads partition columns from directory names
- `spark.sql.sources.partitionOverwriteMode` -
    - `static` (default) - `overwrite` mode deletes entire output directory before writing
    - `dynamic` - `overwrite` mode only overwrites partitions present in the new data; other partitions untouched

### Bucketing

- `bucketBy(numBuckets, "col1", "col2").sortBy("col1")` - writes data into a fixed number of buckets
- Only supported for `saveAsTable` (not file path writes)
- Bucket files named `part-XXXXX-bucketId.parquet`
- Enables bucket join (shuffle-free join) and bucket sort (sort-free sort) in downstream queries

### File Size Control

- `df.repartition(n).write.parquet(...)` - control number of output files (= number of partitions)
- `df.coalesce(n).write.parquet(...)` - reduce output files without full shuffle
- `spark.sql.files.maxRecordsPerFile` (default $0$ = unlimited) - split output files at record count threshold

---

## DataFrameStatFunctions

- __`DataFrameStatFunctions`__ - statistical functions accessible via `df.stat`
- All methods are actions - they trigger computation and return results to Driver

### Correlation and Covariance

- `df.stat.corr("col1", "col2", method="pearson")` - Pearson correlation coefficient between two numeric columns
    - Returns `Double` in $[-1, 1]$
    - `method` - only `"pearson"` supported currently
- `df.stat.cov("col1", "col2")` - covariance between two numeric columns
    - Returns `Double`

### Frequency and Approximate Statistics

- `df.stat.freqItems(["col1", "col2"], support=0.01)` - finds items that appear in more than `support` fraction of rows
    - Uses Count-Min Sketch; approximate; default support $0.01$ (items appearing in >1% of rows)
    - Returns DataFrame with columns `col1_freqItems`, `col2_freqItems` (arrays of frequent values)
- `df.stat.approxQuantile("col", probabilities=[0.25, 0.5, 0.75], relativeError=0.01)` -
    - Computes approximate quantiles using Greenwald-Khanna algorithm
    - `probabilities` - array of quantiles to compute (0.0 to 1.0)
    - `relativeError` - accuracy; $0.0$ = exact (sorts full dataset); $0.01$ = 1% error; higher = faster
    - Returns `List[Double]`

### Crosstab and Stratified Sampling

- `df.stat.crosstab("col1", "col2")` - computes cross-tabulation (contingency table) of two columns
    - Returns DataFrame where rows are distinct values of `col1`, columns are distinct values of `col2`, cells are counts
    - Memory-intensive for high-cardinality columns
- `df.stat.sampleBy("col", fractions={"A": 0.5, "B": 0.2}, seed=42)` - stratified sample
    - `fractions` - dict mapping column value to sampling fraction
    - Returns DataFrame with sampled rows; each value sampled at its specified rate

### Bloom Filter and Count-Min Sketch

- Python -
```python
    # Bloom filter - probabilistic set membership
    bf = df.stat.bloomFilter("id", expectedNumItems=1000000, fpp=0.01)
    bf.mightContain(12345)      # True/False (may have false positives)

    # Count-Min Sketch - approximate frequency counting
    cms = df.stat.countMinSketch("id", eps=0.01, confidence=0.95, seed=42)
    cms.estimateCount(12345)    # approximate count of value 12345
```

---

## DataFrameNaFunctions

- __`DataFrameNaFunctions`__ - functions for handling null and NaN values; accessed via `df.na`

### Drop Operations

- `df.na.drop()` - drops rows where ANY column is null or NaN
- `df.na.drop("any")` - same as above (default)
- `df.na.drop("all")` - drops rows where ALL columns are null or NaN
- `df.na.drop(thresh=3)` - drops rows with fewer than `thresh` non-null values
- `df.na.drop(subset=["col1", "col2"])` - only consider specified columns for null check

- Python -
```python
    df.na.drop()                                    # any null → drop row
    df.na.drop("all")                               # all null → drop row
    df.na.drop(thresh=2)                            # fewer than 2 non-null → drop row
    df.na.drop(subset=["age", "salary"])            # null in age or salary → drop row
```

### Fill Operations

- `df.na.fill(value)` - replace null/NaN with `value` in all compatible columns
- `df.na.fill(value, subset=["col1", "col2"])` - only specified columns
- `df.na.fill({"col1": 0, "col2": "unknown"})` - per-column fill values (dict/map)

- Python -
```python
    df.na.fill(0)                                   # fill all numeric nulls with 0
    df.na.fill("N/A")                               # fill all string nulls with "N/A"
    df.na.fill(0, subset=["age", "salary"])
    df.na.fill({"age": 0, "name": "unknown", "active": False})
```

- Scala -
```scala
    df.na.fill(0)
    df.na.fill("N/A")
    df.na.fill(Map("age" -> 0, "name" -> "unknown"))
```

### Replace Operations

- `df.na.replace(cols, replacements)` - replace specific values with other values; not just nulls

- Python -
```python
    df.na.replace(["NA", "N/A", "null"], None)              # normalize missing value strings to null
    df.na.replace({"old_val": "new_val"}, subset=["col1"])
    df.na.replace([1, 2, 3], [10, 20, 30])                  # replace multiple values
```

### NaN vs Null

- `null` - SQL null; absence of value; propagates through most operations
- `NaN` - IEEE 754 not-a-number; only applies to `FloatType` and `DoubleType`; result of `0.0/0.0`
- `df.na.drop()` treats both as missing
- `df.na.fill(0.0)` fills both `null` and `NaN` in float/double columns
- `isnan(col("x"))` vs `col("x").isNull` - different checks; use both for robust null/NaN handling

> [!NOTE]
> `df.na.fill` and `df.na.drop` are the most commonly used in production for data quality pipelines. The `subset` parameter is critical - filling all columns with a single value when only some columns need it produces incorrect results. Always specify `subset` explicitly.

> [!TIP]
> For complex null handling in production -
> ```python
> # Pattern - handle nulls explicitly per column type
> df = df.na.fill({
>     "numeric_col": 0,
>     "string_col": "unknown",
>     "flag_col": False
> }).na.drop(subset=["required_col1", "required_col2"])
> ```
> This fills optional columns with defaults first, then drops rows missing required columns.
