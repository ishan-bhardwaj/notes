# DataFrames
- Spark is a unified computing engine & libraries for distributed data processing.
- A `DataFrame` is a distributed collection of `Rows` conforming to a schema. A schema is a list describing the column names and their types.
- DataFrames are immutable - can't be changed once created. We can create other DFs using transformations.
- `SparkSession` allows creating, reading and writing DataFrames. Creating a `SparkSession` -
```
import org.apache.spark.sql.SparkSession
val spark = SparkSession.builder()
    .appName("My App")
    .config("spark.master", "local")
    .getOrCreate()
```
- Reading a DataFrame from JSON file -
```
val df: DataFrame = spark.read
    .format("json")
    .option("inferSchema", "true")
    .load("<json_file_path>")
```
- Printing out few rows of the DataFrame - `df.show()`
- Printing out the schema - `df.printSchema()`
- Printing out few rows as sequence - `df.take(10).foreach(println)`

### Spark Types 
- We have Spark's internal types like - `LongType`, `DoubleType`, `StringType`, `DateType`, `StructType` etc. We can use these types using the import - `import org.apache.spark.sql.types._`
- Defining schema -
```
val dfSchema = StructType(Array(
    StructField("Name", StringType, nullable = true),
    StructField("Model", StringType, nullable = true),
    StructField("Price", DoubleType, nullable = true)
))
```

- To obtain schema from existing DataFrame -
```
val schema: StructType = df.schema
```

> [!WARNING]
> It's not a good practise to use `inferSchema` on production as it can parse the data into incorrect types.

- Reading DataFrame with our own schema -
```
val df: DataFrame = spark.read
    .format("json")
    .schema(dfSchema)
    .load("<json_file_path>")
```

- Creating `Row` and `DataFrame` manually -
```
import org.apache.spark.sql.Row

// Creating a Row
val myRow = Row("Ishan", "Bhardwaj", 25, 1994)

// Creating a DataFrame
val records = Seq(
    ("Ishan", "Bhardwaj", 25, 1994)
)
val df = spark.createDataFrame(records) // Column names will be _1, _2, _3, _4 etc
```

> [!NOTE]
> When creating the `DataFrame` from `Seq[Tuple]`, we don't need to define schema because Spark can infer it automatically from the tuple's datatypes.

> [!NOTE]
> Schemas are only applicable to `DataFrame`, but not to `Row`. Rows are unstructured data.

- Creating DataFrames with implicits - 
```
import spark.implicits._

val df = records.toDF("first_name", "last_name", "id", "year_of_birth")
```

### Transformations
- Narrow - one input partition contributes to at most one output partition (e.g. `map`)
- Wide - input partitions (one or more) create many ouput partitions (e.g. sort)

### Shuffle
- Data exchanges between cluster nodes
- Occurs in wide transformations

### Computing DataFrames
- Lazy Evaluation - Spark waits till the last moment to execute the DF transformations.
- Planning - Spark compiles the DF transformations into a graph before running any code. Spark compiles 2 plans -
    - Logical Plan - DF dependency graph + narrow/wide transformations sequence
    - Physical Plan - Optimized sequence of steps for nodes in the cluster
- Spark also applies some optimizations in the planning stage.

### Transformations vs Actions
- Transformations describe how new DFs are obtained e.g. `select`, `withColumn` etc.
- Actions actually start executing Spark code e.g. `count()` etc.

