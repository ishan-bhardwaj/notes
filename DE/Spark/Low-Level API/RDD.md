# Resilient Distributed Datasets (RDDs)

- RDD is Spark’s core data structure — an immutable, fault-tolerant, partitioned collection of elements that can be operated on in parallel.
- Two ways of creating RDDs -
  - _parallelizing_ an existing collection in driver program.
  - Loading a dataset from external storage system (any data source offering a Hadoop InputFormat like HDFS, HBase etc).

## `SparkContext`

- Entry-point to Spark.

- Creating a `SparkContext` -
  ```
  conf = SparkConf().setAppName(app_name).setMaster(master)
  sc = SparkContext(conf=conf)
  ```

  - where -
    - `app_name` - application name visible in Spark UI.
    - `master` - cluster url or `"local"` / `"local[*]"` for local testing. 
      - `"local[2]"` - uses 2 cores on local.  


## Parallelizing an existing collection

```
data = [1, 2, 3, 4, 5]
rdd = sc.parallelize(data)          # can be operated on in parallel
```

- A parallel collection is divided into number of _partitions_, and then Spark runs on _task_ for each partition.
- `SparkContext#parallelize(data, num_partitions)` - to specify the number of partitions.
  - By default, Spark tries to set the number of partitions automatically based on your cluster.
  - Typically, you want 2-4 partitions for each CPU in your cluster. 

## External Datasets

### Text Files 

- `SparkContext#textFile("file_path")` - creates a text file RDD, returning one record per line in each file.
- Notes -
  - Local filesystem files must be accessible at the same path on worker nodes - either copy the files to all workers or use a network-mounted shared file system.
  - Spark’s file-based input methods support running on directories, compressed files, and wildcards, eg -
    - `SparkContext#textFile("/my/directory")`
    - `SparkContext#textFile("/my/directory/*.txt")`
    - `SparkContext#textFile("/my/directory/*.gz")`
  - `SparkContext#textFile("file_path", num_partitions)` - to specify the number of partitions manually.
    - By default, Spark creates one partition for each block of the file (`128MB` is the default block size in HDFS).
    - Note that you cannot have fewer partitions than blocks.

- `SparkContext#wholeTextFiles` - reads a directory containing multiple small text files, and returns each of them as `(filename, content)` pairs.

- `RDD#saveAsPickleFile` and `SparkContext#pickleFile` - saves an RDD in a simple format consisting of pickled Python objects.
  - Batching is used on pickle serialization, with default batch size `10`.

### Sequence Files
  
- PySpark supports Hadoop SequenceFiles, which store key-value pairs.
- __Reading__ - 
  - Loads an RDD of key-value pairs from the Java SequenceFile.
  - onverts Java Writables → base Java types, and then "pickles" Java objects → Python objects for use in PySpark.
  - Example -
  ```
  rdd = sc.sequenceFile("file_path")        # RDD of (key, value) Python objects
  ```

- __Writing__ -
  - Takes an RDD of key-value pairs in Python.
  - Unpickles Python objects → Java objects, and then converts Java objects → Java Writables for SequenceFile storage.
  - Example -
  ```
  rdd.saveAsSequenceFile("file_path")
  ```

- __Automatic Writable Conversions__ -

| Writable Type     | Python Type |
| ----------------- | ----------- |
| `Text`            | `str`       |
| `IntWritable`     | `int`       |
| `FloatWritable`   | `float`     |
| `DoubleWritable`  | `float`     |
| `BooleanWritable` | `bool`      |
| `BytesWritable`   | `bytearray` |
| `NullWritable`    | `None`      |
| `MapWritable`     | `dict`      |

- __Handling Arrays__ -
  - Custom `ArrayWritable` subtypes must be specified for arrays.
  - When reading - the default converter will convert custom `ArrayWritable` subtypes to Java `Object[]`, which then get pickled to Python tuples.
  - When writing - specify custom converters that convert arrays to custom `ArrayWritable` subtypes.

### Other Hadoop Input/Output Formats

- PySpark can read any Hadoop `InputFormat` or write any Hadoop `OutputFormat`.
- If required, a Hadoop configuration can be passed in as a Python `dict`.
- Example - Elasticsearch `ESInputFormat` -
```
conf = {"es.resource" : "index/type"}
rdd = sc.newAPIHadoopRDD("org.elasticsearch.hadoop.mr.EsInputFormat",
                         "org.apache.hadoop.io.NullWritable",
                         "org.elasticsearch.hadoop.mr.LinkedMapWritable",
                         conf=conf)

rdd.first()             # the result is a MapWritable that is converted to a Python dict
(u'Elasticsearch ID',
 {u'field1': True,
  u'field2': u'Some Text',
  u'field3': 12345})
```

- For custom serialized data (such as loading data from Cassandra / HBase) -
  - Data must be converted to pickle-compatible types (primitive types, arrays, dicts, etc.).
  - Extend the `Converter` trait and implement the `convert` with transformation logic -
    - In Java/Scala -
    ```
    class MyCustomConverter extends Converter[InputType, OutputType] {
      override def convert(input: InputType): OutputType = ???
    }
    ```

    - In Python, just create a class with a `convert` method -
    ```
    class MyCustomConverter:
      def convert(self, input_data):
        return input_data.decode("utf-8")
    ```

## RDD Operations

- Two types of operations -
  - Transformations - 
    - Create a new dataset from an existing one, eg - `map`, `filter`, `flatMap`.
    - Transformations are _lazy_ i.e. Spark does not compute them immediately. Instead, it builds a DAG (Directed Acyclic Graph) of transformations.
  - Actions - 
    - Return a value to the driver program after running a computation on the dataset, eg - `reduce`, `collect`, `count`.
    - Computation only happens when you run an action.

- __Persistence__ -
  - By default, every action recomputes the RDD.
  - To persist an RDD in memory - `RDD#persist()` or `RDD#cache()` - Spark will keep the elements around on the cluster for much faster access the next time you query it. 

## Passing Functions to Spark

- Three ways to pass functions to Spark -
  - Lambda expressions.
  - Local `def`s.
  - Top-level functions in a module.

- If you call a method of a class and it accesses instance fields or methods, the whole object needs to be sent to the cluster - creating unnecessary object serialization.
- Example - accessing instance method -
```
class MyClass(object):
  def func(self, s):
    return s

  def doStuff(self, rdd):
    return rdd.map(self.func)
```

- Example - accessing instance field -
```
class MyClass(object):
  def __init__(self):
    self.field = "Hello"
  
  def doStuff(self, rdd):
    return rdd.map(lambda s: self.field + s)
```

- Solution - copy the field to a local variable inside the method so that Spark will only serialize the local variable, not the whole object -
```
def doStuff(self, rdd):
  field = self.field
  return rdd.map(lambda s: field + s)
```

- Rule of thumb -
  - Keep functions picklable.
  - Use local variables, lambdas with local variables, or top-level functions (functions defined outside any class or at the top level of a module).

## Closures

- To execute jobs, Spark breaks up the processing of RDD operations into tasks, each of which is executed by an executor.
  - Prior to execution, Spark computes the task’s closure.
  - The __closure__ is those variables and methods which must be visible for the executor to perform its computations on the RDD.
  -  This closure is serialized and sent to each executor.
  - The variables within the closure sent to each executor are now copies.
  
- Example -
```
counter = 0

def increment_counter(x):
    global counter
    counter += x

rdd.foreach(increment_counter)
print("Counter value: ", counter)
```

- The `counter` referenced within the `foreach` function, it’s no longer the `counter` on the driver node. 
  - There is a `counter` in driver's memory, but it is not visible to the executors.
  - The executors only see the copy from the serialized closure.
  - Thus, the final value of `counter` will still be zero since all operations on counter were referencing the value within the serialized closure.

> [!WARNING]
> In local mode, sometimes the `foreach` function will actually execute within the same JVM as the driver and will reference the same original `counter`, and may actually update it.

- __Solution__ - Use an _accumulator_ instead if some global aggregation is needed.

## Printing elements of an RDD

- `rdd.foreach(println)` -
  - Local mode - generate the expected output and print all the RDD’s elements.
  - Cluster mode - use executor’s `stdout` instead, not the one on the driver, so `stdout` on the driver won’t show the RDD's elements.

- To print all elements on the driver -
  - Use `collect()` method to first bring the RDD to the driver node and then call `foreach`.
  - But this can cause the driver to run out of memory because `collect()` fetches the entire RDD to a single machine.
  - To print few elements of the RDD, use `take()` - `rdd.take(100).foreach(println)`

## Key-Value Pairs

- A few special operations are only available on RDDs of key-value pairs such as distributed “shuffle” operations, such as grouping or aggregating the elements by a key.
- These operations work on RDDs containing built-in Python tuples.
- Example -
```
lines = sc.textFile("data.txt")
pairs = lines.map(lambda s: (s, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)
```

## Core Metadata & Introspection

| Method               | Detailed Description                                                                                         |
| -------------------- | ------------------------------------------------------------------------------------------------------------ |
| `id()`               | Returns a unique identifier for the RDD within its SparkContext, useful for debugging execution plans.       |
| `name()`             | Retrieves the name assigned to the RDD, typically used for debugging or UI visibility.                       |
| `setName(name)`      | Assigns a human-readable name to the RDD so it is easier to track in Spark UI and logs.                      |
| `getNumPartitions()` | Returns how many partitions the RDD is split into, which directly affects parallelism.                       |
| `context`            | Property that returns the SparkContext used to create this RDD.                                              |
| `toDebugString()`    | Produces a lineage string showing how the RDD was built, including parent dependencies and execution stages. |

## Transformations

| Method                | Detailed Description                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------ |
| `map(f)`              | Applies a function to every element and returns a new RDD containing transformed elements. |
| `flatMap(f)`          | Applies a function that returns iterables and flattens them into a single RDD.             |
| `filter(f)`           | Returns a new RDD containing only elements that satisfy the predicate.                     |
| `distinct()`          | Eliminates duplicate elements across all partitions.                                       |
| `union(other)`        | Combines two RDDs into one without removing duplicates.                                    |
| `intersection(other)` | Returns only elements present in both RDDs.                                                |
| `subtract(other)`     | Returns elements from this RDD that do not exist in the other.                             |
| `sample()`            | Produces a randomly sampled subset of elements.                                            |
| `randomSplit()`       | Splits the RDD into multiple RDDs using weighted probabilities.                            |
| `sortBy()`            | Sorts elements based on a key function.                                                    |
| `sortByKey()`         | Sorts a key-value RDD by its keys.                                                         |
| `keyBy(f)`            | Converts each element into a `(key, value)` pair where key is computed.                    |
| `mapValues(f)`        | Applies a function only to values of a pair RDD without changing keys.                     |
| `flatMapValues(f)`    | Similar to mapValues but flattens iterable results.                                        |
| `zip(other)`          | Combines two RDDs element-wise into pairs.                                                 |
| `zipWithIndex()`      | Assigns a sequential index to each element.                                                |
| `zipWithUniqueId()`   | Assigns a unique 64-bit ID to each element.                                                |

## Actions

| Method            | Detailed Description                                                         |
| ----------------- | ---------------------------------------------------------------------------- |
| `collect()`       | Retrieves all elements to the driver program (dangerous for large datasets). |
| `first()`         | Returns the first element.                                                   |
| `take(n)`         | Returns the first n elements.                                                |
| `count()`         | Returns total number of elements.                                            |
| `reduce(f)`       | Aggregates elements using an associative function.                           |
| `sum()`           | Adds all numeric elements.                                                   |
| `max()` / `min()` | Returns largest or smallest element.                                         |
| `mean()`          | Computes arithmetic mean of numeric elements.                                |
| `stats()`         | Returns statistics object containing count, mean, variance, etc.             |
| `countByValue()`  | Returns dictionary with counts of each unique value.                         |
| `countByKey()`    | Returns dictionary of key frequencies.                                       |
| `collectAsMap()`  | Collects key-value pairs as a dictionary.                                    |
| `lookup(key)`     | Returns all values associated with a specific key.                           |
| `isEmpty()`       | Returns True if RDD has no elements.                                         |


## Aggregation Operations

| Method                 | Description                                                                                               |
| ---------------------- | --------------------------------------------------------------------------------------------------------- |
| `aggregate()`          | Performs aggregation using separate functions for per-partition accumulation and cross-partition merging. |
| `fold()`               | Similar to reduce but requires a neutral zero value.                                                      |
| `treeAggregate()`      | Aggregates in a hierarchical tree pattern to reduce communication overhead.                               |
| `treeReduce()`         | Tree-based reduce for improved performance on large clusters.                                             |
| `combineByKey()`       | Most general per-key aggregation allowing custom combiner logic.                                          |
| `reduceByKey()`        | Efficiently merges values per key using local aggregation before shuffle.                                 |
| `reduceByKeyLocally()` | Same as reduceByKey but returns results as a dictionary to driver.                                        |
| `aggregateByKey()`     | Aggregates values of each key with separate seq and comb functions.                                       |
| `foldByKey()`          | Per-key fold with neutral value.                                                                          |

## Joins & Grouping

| Method             | Description                                                      |
| ------------------ | ---------------------------------------------------------------- |
| `groupBy()`        | Groups elements according to a function.                         |
| `groupByKey()`     | Groups values for each key into a collection (can be expensive). |
| `cogroup()`        | Groups values from multiple RDDs sharing same key.               |
| `groupWith()`      | Alias for cogroup supporting multiple RDDs.                      |
| `join()`           | Inner join between two key-value RDDs.                           |
| `leftOuterJoin()`  | Returns all left keys and matching right values.                 |
| `rightOuterJoin()` | Returns all right keys and matching left values.                 |
| `fullOuterJoin()`  | Returns all keys from both RDDs with optional values.            |

## Partitioning & Execution Control

| Method                     | Description                                                       |
| -------------------------- | ----------------------------------------------------------------- |
| `repartition(n)`           | Reshuffles data to create exactly n partitions.                   |
| `coalesce(n)`              | Reduces partitions efficiently without full shuffle.              |
| `partitionBy()`            | Repartitions using a custom partitioner.                          |
| `glom()`                   | Converts each partition into a list of its elements.              |
| `mapPartitions()`          | Applies a function once per partition instead of per element.     |
| `mapPartitionsWithIndex()` | Same as above but includes partition index.                       |
| `barrier()`                | Marks stage as barrier stage where all tasks must start together. |

## Persistence & Checkpointing

| Method                | Description                                                         |
| --------------------- | ------------------------------------------------------------------- |
| `cache()`             | Persists RDD in memory using default storage level.                 |
| `persist(level)`      | Persists RDD using custom storage level.                            |
| `unpersist()`         | Removes cached data from memory/disk.                               |
| `checkpoint()`        | Saves RDD to reliable storage to truncate lineage.                  |
| `localCheckpoint()`   | Saves checkpoint using executor storage (less reliable but faster). |
| `isCheckpointed()`    | Returns whether checkpoint exists.                                  |
| `getCheckpointFile()` | Returns checkpoint file path.                                       |
| `getStorageLevel()`   | Returns current persistence level.                                  |

## Sampling & Approximate Algorithms

| Method                  | Description                                             |
| ----------------------- | ------------------------------------------------------- |
| `countApprox()`         | Returns approximate count within timeout.               |
| `countApproxDistinct()` | Estimates distinct count using probabilistic algorithm. |
| `meanApprox()`          | Approximates mean with confidence bounds.               |
| `sumApprox()`           | Approximates sum with statistical guarantees.           |
| `sampleByKey()`         | Performs stratified sampling by key.                    |

## Statistical Functions

| Method             | Description                                  |
| ------------------ | -------------------------------------------- |
| `variance()`       | Computes variance.                           |
| `sampleVariance()` | Sample variance corrected for bias.          |
| `stdev()`          | Standard deviation.                          |
| `sampleStdev()`    | Sample standard deviation.                   |
| `histogram()`      | Computes histogram across specified buckets. |

## Input / Output Operations

| Method                        | Description                               |
| ----------------------------- | ----------------------------------------- |
| `saveAsTextFile()`            | Saves elements as text files.             |
| `saveAsPickleFile()`          | Serializes and saves objects.             |
| `saveAsSequenceFile()`        | Saves as Hadoop SequenceFile.             |
| `saveAsHadoopFile()`          | Writes using old Hadoop API.              |
| `saveAsNewAPIHadoopFile()`    | Writes using newer Hadoop API.            |
| `saveAsHadoopDataset()`       | Writes key-value pairs as Hadoop dataset. |
| `saveAsNewAPIHadoopDataset()` | Same using new API.                       |
| `pipe(cmd)`                   | Sends elements to external shell command. |

## Execution Utilities

| Method                       | Description                                                    |
| ---------------------------- | -------------------------------------------------------------- |
| `foreach(f)`                 | Applies function to each element for side effects.             |
| `foreachPartition(f)`        | Applies function once per partition.                           |
| `cleanShuffleDependencies()` | Removes shuffle files and ancestors no longer needed.          |
| `getResourceProfile()`       | Returns resource profile attached to RDD.                      |
| `withResources(profile)`     | Assigns resource profile for execution.                        |
| `collectWithJobGroup()`      | Collects results while tagging job with group metadata.        |
| `toLocalIterator()`          | Returns iterator over elements without collecting all at once. |

## RDD Performance Best-Practice Rules

| Rule                                                       | Applies To          | Why It Matters                                                           | What To Do Instead / Tip                                |
| ---------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------- |
| Prefer `reduceByKey` over `groupByKey`                     | Aggregations        | `groupByKey` shuffles all values; `reduceByKey` does local combine first | Always use reduce-style aggregations when possible      |
| Avoid `collect()` on large RDDs                            | Actions             | Pulls entire dataset to driver → OOM risk                                | Use `take()`, `foreachPartition()`, or write to storage |
| Use `mapPartitions()` for expensive setup                  | Transformations     | Avoids per-record initialization overhead                                | Create DB/API connection once per partition             |
| Cache reused RDDs                                          | Persistence         | Prevents recomputation of lineage                                        | Use `cache()` or `persist()`                            |
| Do not cache if used once                                  | Persistence         | Wastes memory                                                            | Cache only reused RDDs                                  |
| Use correct storage level                                  | Persistence         | Memory vs CPU tradeoff                                                   | MEMORY_ONLY for fast; MEMORY_AND_DISK if large          |
| Prefer `coalesce()` when reducing partitions               | Partitioning        | Avoids full shuffle                                                      | Use `repartition()` only when increasing partitions     |
| Use `repartition()` for load balancing                     | Partitioning        | Ensures even distribution                                                | Useful after filters or skew                            |
| Avoid unnecessary `distinct()`                             | Transformations     | Requires shuffle + sorting                                               | Use keys or aggregations if possible                    |
| Avoid wide transformations when possible                   | Shuffles            | Shuffles are most expensive operations                                   | Prefer narrow transformations                           |
| Use `treeReduce()` or `treeAggregate()` for large clusters | Aggregation         | Reduces driver bottleneck                                                | Better scalability                                      |
| Prefer `mapValues()` over `map()` for pair RDDs            | Transformations     | Preserves partitioner                                                    | Avoids reshuffle                                        |
| Prefer `flatMap()` over `map().flatten()`                  | Transformations     | Saves extra pass                                                         | Use directly                                            |
| Use `sample()` before heavy operations                     | Sampling            | Reduce data size early                                                   | Helpful for debugging/testing                           |
| Checkpoint long lineage chains                             | Fault tolerance     | Prevents stack-deep recomputation                                        | Use `checkpoint()` periodically                         |
| Use `localCheckpoint()` for iterative jobs                 | Iterative workloads | Faster than reliable checkpoint                                          | Useful for ML loops                                     |
| Avoid large closures                                       | Serialization       | Increases task size + network cost                                       | Broadcast large objects                                 |
| Use broadcast variables                                    | Shared data         | Avoids shipping data to each task                                        | `sc.broadcast(obj)`                                     |
| Avoid nested RDD operations                                | Logic design        | Causes job explosion                                                     | Flatten logic                                           |
| Prefer `count()` to check emptiness? No                    | Actions             | Full scan required                                                       | Use `isEmpty()`                                         |
| Use `take(1)` instead of `first()` when safe               | Actions             | More predictable execution                                               | Prevents some edge issues                               |
| Avoid `lookup()` for many keys                             | Actions             | Executes separate job per key                                            | Join instead                                            |
| Avoid many small partitions                                | Scheduling          | Scheduler overhead dominates                                             | Tune partition count                                    |
| Avoid very large partitions                                | Parallelism         | Under-utilizes cluster                                                   | Target 100–200MB per partition                          |
| Use `partitionBy()` before joins                           | Joins               | Prevents reshuffle during join                                           | Especially for repeated joins                           |
| Persist before multiple joins                              | Joins               | Prevents recomputation                                                   | Cache intermediate results                              |
| Avoid `groupBy()` for aggregation                          | Grouping            | Moves entire dataset                                                     | Use reduce/fold                                         |
| Use `foreachPartition()` for output writes                 | Actions             | Batch writes improve throughput                                          | Especially DB writes                                    |
| Avoid `zip()` unless partitions match                      | Pair ops            | Requires identical partition counts                                      | Use join if unsure                                      |
| Avoid `cartesian()` unless dataset tiny                    | Transformations     | Produces N×M records                                                     | Explodes memory & shuffle                               |
| Prefer numeric aggregations over Python loops              | Aggregation         | Spark executes in JVM faster                                             | Push logic into Spark functions                         |
| Avoid excessive actions                                    | Execution           | Each action triggers full DAG execution                                  | Combine logic into one action                           |
| Use `glom()` only for debugging                            | Partition ops       | Materializes partitions                                                  | Avoid in production                                     |
| Avoid calling Spark inside transformations                 | Design              | Creates nested jobs                                                      | Compute outside                                         |
| Control shuffle partitions                                 | Shuffles            | Default may be inefficient                                               | Tune `spark.sql.shuffle.partitions` or RDD partitioning |
| Use approximate ops for large data                         | Approx APIs         | Much faster for analytics                                                | `countApproxDistinct`, `meanApprox`                     |
| Use `sampleByKey()` for skewed keys                        | Sampling            | Prevents dominant key bias                                               | Stratified sampling                                     |
| Sort only when necessary                                   | Sorting             | Sorting is expensive                                                     | Avoid unless required                                   |
| Avoid writing many small output files                      | Output              | Small files slow downstream systems                                      | Use coalesce before write                               |
| Name important RDDs                                        | Debugging           | Helps Spark UI analysis                                                  | `setName()`                                             |
| Use `toLocalIterator()` for streaming driver processing    | Actions             | Avoids collecting entire dataset                                         | Processes lazily                                        |
| Avoid frequent repartition chains                          | Partitioning        | Multiple shuffles                                                        | Combine partition changes                               |
| Avoid wide dependency chains                               | DAG                 | Slows scheduling                                                         | Cache checkpoints strategically                         |
| Monitor lineage using `toDebugString()`                    | Debugging           | Reveals inefficiencies                                                   | Inspect before optimization                             |

