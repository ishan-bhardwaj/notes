# Datasets

- Datasets are distributed collection of data.
- A DataFrame is a dataset organized into named columns.
- Scala - `DataFrame` = `Dataset[Row]`
- Java - `DataFrame` = `Dataset<Row>`

- __Key differences__ -

| Feature       | Dataset                        | DataFrame                       |
| ------------- | ------------------------------ | ------------------------------- |
| Type          | Strongly typed                 | Un-typed / Rows                 |
| API           | Functional (map, filter, etc.) | Columnar operations / SQL-style |
| Optimizations | Spark SQL engine optimizations | Same as Dataset                 |
| Languages     | Scala, Java                    | Scala, Java, Python, R          |

## SparkSession

- Entry point to all Spark functionality (2.0+).
- Creating SparkSession -
```
from pyspark.sql import SparkSession

spark = SparkSession \
    .builder \
    .appName("Python Spark SQL basic example") \
    .config("spark.some.config.option", "some-value") \
    .getOrCreate()
```

> [!NOTE]
> SparkSession in 2.0+ provides built-in support for Hive features - HiveQL queries, Hive UDFs, reading Hive tables. No existing Hive setup is needed to use these features.

## Creating DataFrames

- Can be created from existing RDDs, Hive tables, Spark data sources (such as JSON, CSV, Parquet, databases).
```
df = spark.read.json("examples/src/main/resources/people.json")
df.show()
```

## DataFrame Operations

- Python Column Access -
  - Attribute access - `df.age` - convenient.
  - Index access - `df['age']` - recommended, future-proof.

```
df.printSchema()                                    # print the schema in a tree format
df.select("name").show()                            # select column
df.select(df['name'], df['age'] + 1).show()         # add 1 to age
df.filter(df['age'] > 21).show()                    # filter by condition
df.groupBy("age").count().show()                    # group by age
```

## Running SQL Queries using `spark.sql()`

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

## Creating Datasets (Scala/Java)

- Similar to RDDs, however, instead of using Java serialization or Kryo, they use `Encoder` to serialize the objects for processing or transmitting over the network.
- Encoders are code generated dynamically and use a format that allows Spark to perform many operations like filtering, sorting and hashing without deserializing the bytes back into an object.
- Example -
```
case class Person(name: String, age: Long)

val persons = Seq(Person("Andy", 32)).toDS()
persons.show()

val primitiveDS = Seq(1,2,3).toDS()
primitiveDS.map(_ + 1).collect()                        // Array(2,3,4)

val path = "examples/src/main/resources/people.json"
val peopleDS = spark.read.json(path).as[Person]
peopleDS.show()
```

## Interoperating with RDDs

- Method 1 - __Inferring Schema via Reflection__ -
  - Works when schema known at compile time.
  - Rows are constructed by passing a list of key/value pairs as kwargs to the Row class. 
  - The keys of this list define the column names of the table, and the types are inferred by sampling the whole dataset.
  - Example -
    ```
    from pyspark.sql import Row

    lines = sc.textFile("examples/src/main/resources/people.txt")
    parts = lines.map(lambda l: l.split(","))
    people = parts.map(lambda p: Row(name=p[0], age=int(p[1])))

    schemaPeople = spark.createDataFrame(people)
    schemaPeople.createOrReplaceTempView("people")

    teenagers = spark.sql("SELECT name FROM people WHERE age >= 13 AND age <= 19")
    teenNames = teenagers.rdd.map(lambda p: "Name: " + p.name).collect()
    ```

- Method 2 - __Programmatically Specifying Schema__ -
  - Used when a dictionary of kwargs cannot be defined ahead of time.
  - Steps -
    - Create an RDD of tuples or lists from the original RDD;
    - Create the schema represented by a `StructType` matching the structure of tuples or lists in the RDD created in the step 1.
    - Apply the schema to the RDD via `createDataFrame` method provided by `SparkSession`.
  - Example -
    ```
    from pyspark.sql.types import StringType, StructType, StructField

    lines = sc.textFile("examples/src/main/resources/people.txt")
    people = lines.map(lambda l: tuple(l.split(",")))

    schemaString = "name age"
    fields = [StructField(field_name, StringType(), True) for field_name in schemaString.split()]
    schema = StructType(fields)

    schemaPeople = spark.createDataFrame(people, schema)
    schemaPeople.createOrReplaceTempView("people")
    results = spark.sql("SELECT name FROM people")
    results.show()
    ```

## Functions

- __Scalar Functions__ -
  - Return single value per row.
  - Built-in & user-defined scalar functions supported.

- __Aggregate Functions__ -
  - Return single value for a group of rows.
  - Built-in: `count()`, `count_distinct()`, `avg()`, `max()`, `min()`
  - User-defined aggregate functions also supported.

