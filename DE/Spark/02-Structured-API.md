# Structured API - Datasets, DataFrames, Spark SQL

## Datasets

- Strongly typed distributed collection -
    - Scala - `Dataset[T]`
    - Java - `Dataset<T>`

- Not available in Python and R because they are dynamically typed
- Example -
    ```
    case class Person(
        name: String,
        age: String
    )

    val personDF = spark.read.parquet("/data/persons.parquet")
    val persons = personDF.as[Flight]
    ```

- Spark must inspect the type and build a schema automaticall, so Dataset types can only be -
    - Scala case class
    - Java JavaBean-style classes

- Actions like `collect()` or `take()` on a Dataset returns typed objects, not generic `Row` -
    ```
    val result: Array[Person] = persons.take(5)
    ```

## DataFrames

- Spark’s main abstraction for working with structured/tabular data
- A DataFrame is a dataset organized into named columns -
    - Scala - `DataFrame` = `Dataset[Row]`
    - Java - `DataFrame` = `Dataset<Row>`
- Immutable - every operation creates a new DataFrame
- DataFrame consists of rows and schema -
- __Schema__ - 
    - Collection of column names + datatypes
    - Can be defined manually or read from a data source (schema-on-read) 

## Datasets vs DataFrames

| Feature       | Dataset                        | DataFrame                       |
| ------------- | ------------------------------ | ------------------------------- |
| Type          | Strongly typed                 | Un-typed / Rows                 |
| API           | Functional (map, filter, etc.) | Columnar operations / SQL-style |
| Optimizations | Spark SQL engine optimizations | Same as Dataset                 |
| Languages     | Scala, Java                    | Scala, Java, Python, R          |

## Partitions

- Spark splits data into smaller chunks called _partitions_
- A partition is a collection of rows that sit on one physical machine in your cluster
- Each partition is processed by an executor/core in parallel
- Spark maintains a lineage (logical plan/DAG of transformations) for a DataFrame -
    - Lost partitions are recomputed by reapplying the same transformations on the original input data

## Transformations

- Operations that define how data should be modified
- Lazily evaluated
- Two types - narrow and wide
- __Narrow transformations__ -
    - No data movement across executors
    - One input partition contributes to _one_ output partition
    - Spark performs _pipelining_ automatically - multiple narrow transformations are combined together
    - Eg - `filter`, `map`, `select`, `withColumn`
- __Wide transformations__ -
    - One input partition contributes to _many_ output partition
    - Causes shuffles - redistribution of data across cluster
    - Eg - `groupBy`, `join`, `sort`, `distinct`
    - By default, shuffling creates $200$ output partitions
        - Set number of shuffle partitions - `spark.conf.set("spark.sql.shuffle.partitions", "5")`

- __Lazy Evaluation__ -
    - Spark does not execute transformations immediately
    - Instead, Spark builds an execution plan
    - Execution starts only when an action is called

- Access spark plan - `df.explain()`

## Actions

- Actions trigger execution
- One action = one Spark job
- Eg - `count()`, `show()`, `collect()`, `take()`

### DataFrameReader

- API used to read external data into Spark DataFrames
- Accessed through - `SparkSession#read`
- Main responsibilities -
    - Resolve datasource
    - Read metadata/schema
    - Discover partitions/files
    - Build scan plan
- Generic API -
    ```
    spark.read
        .format("parquet")
        .load("/data")
    ```

- Convenience APIs -
    ```
    spark.read.csv(...)
    spark.read.json(...)
    spark.read.parquet(...)

    // all internally call
    format(...).load(...)
    ```

- __Schema Inference__ -
    - Spark samples data and guesses datatypes
    - Enable - `.option("inferSchema", "true")`
    - Always define schema explicitly in production -
        - faster reads
        - stable pipelines
        - predictable types

## Spark SQL

- Any DataFrame can be registered as a temporary SQL view -
    ```
    df.createOrReplaceTempView("table_name")
    ```

- Spark SQL queries and DataFrame API compile to the same logical/physical execution plan
    - No inherent performance difference between them

- __Register Temp View__ -
  - Runs sql queries programmatically.
  - Session-scoped and will disappear if the session that creates it terminates.
    ```
    df.createOrReplaceTempView("people")
    sqlDF = spark.sql("SELECT * FROM people")
    sqlDF.show()
    ```

- __Global Temporary View__ -
  - Session-independent temporary view, lives until Spark app terminates.
  - Shared among all sessions.
  - Global temporary view is tied to a system preserved database `global_temp`, and we must use the qualified name to refer it, e.g. `SELECT * FROM global_temp.view1`.
    ```
    df.createGlobalTempView("people")
    spark.sql("SELECT * FROM global_temp.people").show()
    spark.newSession().sql("SELECT * FROM global_temp.people").show()   # Global temporary view is cross-session
    ```

## Catalyst Engine

- Spark internally uses the Catalyst optimizer
- User code in Scala/Python/SQL is converted into Catalyst expressions/logical plans
- Eg - `df.select(df.col("number") + 10)` is converted into Catalyst expression internally - `Add(Column(number), Literal(10))`
