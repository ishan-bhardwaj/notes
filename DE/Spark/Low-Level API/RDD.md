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

| Method               | Usage                    | Description                                                                                                  | Caveats / Notes                                                    |
| -------------------- | ------------------------ | ------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| `id()`               | `rdd.id()`               | Returns a unique identifier for this RDD within its SparkContext. Useful for debugging and lineage tracking. | Read-only; primarily for internal/debugging use.                   |
| `name()`             | `rdd.name()`             | Returns the name assigned to the RDD. Helps identify RDDs in Spark UI.                                       | Returns `None` if not set.                                         |
| `setName()`          | `rdd.setName("name")`    | Assigns a human-readable name to the RDD for logging and UI visibility.                                      | Optional; can be updated anytime before action execution.          |
| `getNumPartitions()` | `rdd.getNumPartitions()` | Returns the number of partitions in the RDD. Determines parallelism and task scheduling.                     | Changing partition count requires `repartition()` or `coalesce()`. |
| `context`            | `rdd.context`            | SparkContext that created this RDD. Useful for accessing configuration and resources.                        | Read-only.                                                         |
| `toDebugString()`    | `rdd.toDebugString()`    | Returns a lineage string showing the dependency DAG of the RDD, including parent RDDs and transformations.   | Useful to analyze lineage and optimize shuffles.                   |

## Transformations

| Method                                                          | Usage                                                   | Description                                                                                  | Caveats / Notes                                                           |
| --------------------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `map(f)`                                                        | `rdd.map(lambda x: x*2)`                                | Applies a function `f` to every element and returns a new RDD with transformed elements.     | Narrow transformation; does not shuffle.                                  |
| `flatMap(f)`                                                    | `rdd.flatMap(lambda x: [x, x*2])`                       | Applies `f` returning an iterable per element and flattens results.                          | Saves extra flattening step vs `map().flatten()`.                         |
| `filter(f)`                                                     | `rdd.filter(lambda x: x>5)`                             | Returns elements that satisfy a predicate function.                                          | Narrow transformation.                                                    |
| `distinct([numPartitions])`                                     | `rdd.distinct()`                                        | Removes duplicate elements across partitions.                                                | Wide transformation; involves shuffle and sort.                           |
| `union(other)`                                                  | `rdd.union(rdd2)`                                       | Returns a new RDD with elements from both RDDs.                                              | Does not remove duplicates.                                               |
| `intersection(other)`                                           | `rdd.intersection(rdd2)`                                | Returns elements common to both RDDs.                                                        | Wide transformation; shuffle-heavy.                                       |
| `subtract(other)`                                               | `rdd.subtract(rdd2)`                                    | Returns elements in this RDD that are not in the other.                                      | Wide transformation; requires shuffle.                                    |
| `sample(withReplacement, fraction, seed)`                       | `rdd.sample(False, 0.1, 42)`                            | Returns a sampled subset of the RDD with optional replacement.                               | Random; seed controls reproducibility.                                    |
| `randomSplit(weights, seed)`                                    | `rdd.randomSplit([0.7, 0.3], 42)`                       | Splits RDD into multiple RDDs using weights.                                                 | Weighted splits; reproducible with seed.                                  |
| `sortBy(keyfunc, ascending=True, numPartitions=None)`           | `rdd.sortBy(lambda x: x[1])`                            | Sorts RDD by a key function.                                                                 | Wide transformation; expensive shuffle.                                   |
| `sortByKey(ascending=True, numPartitions=None)`                 | `rdd.sortByKey()`                                       | Sorts key-value RDD by keys.                                                                 | Requires shuffle.                                                         |
| `mapValues(f)`                                                  | `rdd.mapValues(lambda v: v*2)`                          | Applies function to values of a pair RDD, keeping keys unchanged.                            | Preserves partitioner; no shuffle if partitioner unchanged.               |
| `flatMapValues(f)`                                              | `rdd.flatMapValues(lambda v: [v, v*2])`                 | Like `mapValues` but flattens iterables returned.                                            | Preserves partitioner.                                                    |
| `keyBy(f)`                                                      | `rdd.keyBy(lambda x: x%2)`                              | Converts each element into a key-value pair `(key, element)` using `f`.                      | May trigger shuffle in subsequent key-based operations.                   |
| `mapPartitions(f)`                                              | `rdd.mapPartitions(lambda iter: [x*2 for x in iter])`   | Applies `f` to each partition instead of individual elements.                                | Useful for expensive initialization (DB/API connections).                 |
| `mapPartitionsWithIndex(f)`                                     | `rdd.mapPartitionsWithIndex(lambda idx, iter: iter)`    | Like `mapPartitions` but passes partition index to `f`.                                      | Can be used for partition-specific logic.                                 |
| `zip(other)`                                                    | `rdd.zip(rdd2)`                                         | Zips two RDDs element-wise into pairs.                                                       | RDDs must have identical number of partitions and elements per partition. |
| `zipWithIndex()`                                                | `rdd.zipWithIndex()`                                    | Zips elements with sequential index.                                                         | Adds unique index; preserves partitioning.                                |
| `zipWithUniqueId()`                                             | `rdd.zipWithUniqueId()`                                 | Zips elements with globally unique 64-bit IDs.                                               | IDs may not be sequential per partition.                                  |
| `cartesian(other)`                                              | `rdd.cartesian(rdd2)`                                   | Returns all pairs of elements (Cartesian product).                                           | Extremely expensive; memory-intensive.                                    |
| `coalesce(numPartitions, shuffle=False)`                        | `rdd.coalesce(2)`                                       | Reduces number of partitions. Avoids full shuffle if `shuffle=False`.                        | Use `repartition()` to increase partitions.                               |
| `repartition(numPartitions)`                                    | `rdd.repartition(10)`                                   | Randomly reshuffles data to create the specified number of partitions.                       | Always triggers a shuffle.                                                |
| `repartitionAndSortWithinPartitions(partitioner)`               | `rdd.repartitionAndSortWithinPartitions(partitioner)`   | Repartitions and sorts keys within each partition efficiently.                               | More efficient than separate `repartition` + `sortByKey`.                 |
| `groupBy(f)`                                                    | `rdd.groupBy(lambda x: x%2)`                            | Groups elements by a function. Returns `(key, iterable)` RDD.                                | Wide transformation; avoid for aggregation, prefer `reduceByKey`.         |
| `groupByKey([numPartitions])`                                   | `rdd.groupByKey()`                                      | Groups values for each key into iterable.                                                    | Shuffle-heavy; use `reduceByKey` or `aggregateByKey` for aggregation.     |
| `reduceByKey(func, [numPartitions])`                            | `rdd.reduceByKey(lambda x,y: x+y)`                      | Aggregates values per key using local combine before shuffle.                                | Preferred over `groupByKey`.                                              |
| `aggregateByKey(zeroValue)(seqFunc, combFunc, [numPartitions])` | `rdd.aggregateByKey(0)(lambda x,y:x+y, lambda x,y:x+y)` | Aggregates per key using separate functions for per-partition and cross-partition combining. | Efficient and flexible; avoids unnecessary allocations.                   |
| `join(other, [numPartitions])`                                  | `rdd.join(rdd2)`                                        | Returns `(K, (V,W))` pairs for keys present in both RDDs.                                    | Wide transformation; shuffle required.                                    |
| `leftOuterJoin(other, [numPartitions])`                         | `rdd.leftOuterJoin(rdd2)`                               | Returns all left keys with optional right values.                                            | Shuffle required.                                                         |
| `rightOuterJoin(other, [numPartitions])`                        | `rdd.rightOuterJoin(rdd2)`                              | Returns all right keys with optional left values.                                            | Shuffle required.                                                         |
| `fullOuterJoin(other, [numPartitions])`                         | `rdd.fullOuterJoin(rdd2)`                               | Returns all keys from both RDDs.                                                             | Shuffle required.                                                         |
| `cogroup(other, [numPartitions])`                               | `rdd.cogroup(rdd2)`                                     | Groups values from multiple RDDs sharing keys.                                               | Shuffle-intensive; alias `groupWith`.                                     |
| `groupWith(other, *others)`                                     | `rdd.groupWith(rdd2)`                                   | Alias for `cogroup` supporting multiple RDDs.                                                | Same as above.                                                            |
| `pipe(command, [envVars])`                                      | `rdd.pipe("cat")`                                       | Pipes each partition through an external shell command.                                      | Expensive; use only when necessary.                                       |

## Actions

| Method                                     | Usage                                              | Description                                                 | Caveats / Notes                                            |
| ------------------------------------------ | -------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------- |
| `collect()`                                | `rdd.collect()`                                    | Returns all elements to driver as a list.                   | Can OOM on large datasets; avoid unless small RDD.         |
| `count()`                                  | `rdd.count()`                                      | Counts number of elements.                                  | Triggers full DAG execution.                               |
| `first()`                                  | `rdd.first()`                                      | Returns first element.                                      | Equivalent to `take(1)[0]`.                                |
| `take(n)`                                  | `rdd.take(10)`                                     | Returns first `n` elements as list.                         | Safer than `collect()` for large RDDs.                     |
| `takeOrdered(n, [key])`                    | `rdd.takeOrdered(5)`                               | Returns `n` smallest elements by default or custom key.     | Triggers partial shuffle.                                  |
| `takeSample(withReplacement, num, seed)`   | `rdd.takeSample(False, 10, 42)`                    | Returns sampled elements as list.                           | Random; seed ensures reproducibility.                      |
| `reduce(func)`                             | `rdd.reduce(lambda x,y: x+y)`                      | Aggregates elements using commutative associative function. | Triggers shuffle for wide operations.                      |
| `sum()`                                    | `rdd.sum()`                                        | Adds all numeric elements.                                  | Shuffles under the hood if distributed.                    |
| `max()` / `min()`                          | `rdd.max()`                                        | Returns max or min element.                                 | Can accept `key` argument.                                 |
| `mean()`                                   | `rdd.mean()`                                       | Computes arithmetic mean.                                   | Uses `sum` and `count` internally.                         |
| `stats()`                                  | `rdd.stats()`                                      | Returns StatCounter object with count, mean, variance, etc. | Computed in one pass; distributed.                         |
| `countByValue()`                           | `rdd.countByValue()`                               | Returns dictionary of value counts.                         | Driver memory sensitive.                                   |
| `countByKey()`                             | `rdd.countByKey()`                                 | Returns dictionary of key counts.                           | Driver memory sensitive; for key-value RDDs.               |
| `collectAsMap()`                           | `rdd.collectAsMap()`                               | Returns key-value RDD as dictionary.                        | Assumes unique keys; OOM possible.                         |
| `lookup(key)`                              | `rdd.lookup(k)`                                    | Returns list of values for the key.                         | Runs separate job per key; expensive for many keys.        |
| `foreach(f)`                               | `rdd.foreach(lambda x: print(x))`                  | Runs function for side effects on each element.             | Do not modify driver variables directly; use Accumulators. |
| `foreachPartition(f)`                      | `rdd.foreachPartition(lambda iter: writeDB(iter))` | Runs function per partition.                                | Efficient for batched writes.                              |
| `toLocalIterator(prefetchPartitions=None)` | `rdd.toLocalIterator()`                            | Lazily iterates elements without collecting all at driver.  | Useful for streaming large datasets.                       |
| `top(n, [key])`                            | `rdd.top(5)`                                       | Returns top N elements according to key.                    | Wide operation; shuffle-intensive.                         |

## Persistence & Checkpointing

| Method                    | Usage                                       | Description                                              | Caveats / Notes                                            |
| ------------------------- | ------------------------------------------- | -------------------------------------------------------- | ---------------------------------------------------------- |
| `cache()`                 | `rdd.cache()`                               | Persist RDD in memory using default level (MEMORY_ONLY). | Avoid caching very large RDDs used once.                   |
| `persist([storageLevel])` | `rdd.persist(StorageLevel.MEMORY_AND_DISK)` | Persist with custom storage level.                       | Choose level based on memory and recomputation trade-offs. |
| `unpersist([blocking])`   | `rdd.unpersist()`                           | Remove RDD from memory/disk.                             | Use after last use.                                        |
| `checkpoint()`            | `rdd.checkpoint()`                          | Persist RDD to reliable storage to truncate lineage.     | Requires HDFS or similar.                                  |
| `localCheckpoint()`       | `rdd.localCheckpoint()`                     | Persist RDD locally in executor storage.                 | Faster but less fault-tolerant.                            |
| `isCheckpointed()`        | `rdd.isCheckpointed()`                      | Returns True if checkpoint exists.                       | Useful for DAG analysis.                                   |
| `getCheckpointFile()`     | `rdd.getCheckpointFile()`                   | Returns checkpoint file path.                            | Only if checkpointed.                                      |
| `getStorageLevel()`       | `rdd.getStorageLevel()`                     | Returns current persistence level.                       | Helpful for debugging memory usage.                        |

## Statistical Functions

| Method               | Usage                         | Description                                          | Caveats / Notes                      |
| -------------------- | ----------------------------- | ---------------------------------------------------- | ------------------------------------ |
| `variance()`         | `rdd.variance()`              | Computes variance of RDD elements.                   | Full pass over data.                 |
| `sampleVariance()`   | `rdd.sampleVariance()`        | Sample variance corrected for bias (divided by N-1). | Corrects bias for sample statistics. |
| `stdev()`            | `rdd.stdev()`                 | Computes standard deviation.                         | Derived from `variance()`.           |
| `sampleStdev()`      | `rdd.sampleStdev()`           | Sample standard deviation.                           | Bias-corrected.                      |
| `histogram(buckets)` | `rdd.histogram([0,10,20,30])` | Computes histogram over given buckets.               | Wide transformation; shuffle-heavy.  |

## Input/Output Operations

| Method                                                                      | Usage                             | Description                                 | Caveats / Notes                                       |
| --------------------------------------------------------------------------- | --------------------------------- | ------------------------------------------- | ----------------------------------------------------- |
| `saveAsTextFile(path[, codec])`                                             | `rdd.saveAsTextFile("out")`       | Writes elements as text files.              | Many small files can slow downstream; coalesce first. |
| `saveAsSequenceFile(path[, codec])`                                         | `rdd.saveAsSequenceFile("out")`   | Saves key-value RDD as Hadoop SequenceFile. | Only for key-value RDDs.                              |
| `saveAsPickleFile(path[, batchSize])`                                       | `rdd.saveAsPickleFile("out.pkl")` | Saves elements as serialized objects.       | Requires Python; driver read can be expensive.        |
| `saveAsHadoopFile()` / `saveAsNewAPIHadoopFile()` / `saveAsHadoopDataset()` | Various Hadoop APIs               | Output RDD as Hadoop dataset/files.         | Complex API; mainly for Hadoop integration.           |
| `pipe(cmd)`                                                                 | `rdd.pipe("cat")`                 | Pipes partitions to external shell process. | Expensive; only if necessary.                         |

## Partitioning & Execution Control

| Method                                        | Usage                            | Description                                                | Caveats / Notes                                |
| --------------------------------------------- | -------------------------------- | ---------------------------------------------------------- | ---------------------------------------------- |
| `partitionBy(numPartitions[, partitionFunc])` | `rdd.partitionBy(10)`            | Repartition RDD using a custom partitioner.                | Useful before joins to reduce shuffle.         |
| `glom()`                                      | `rdd.glom()`                     | Returns RDD where each partition is a list of elements.    | Materializes partitions; mainly for debugging. |
| `barrier()`                                   | `rdd.barrier()`                  | Marks stage as barrier stage; all tasks launched together. | Useful for synchronizing parallel stages.      |
| `withResources(profile)`                      | `rdd.withResources(profile)`     | Assigns a ResourceProfile to execution.                    | Optional; mostly advanced scheduling.          |
| `cleanShuffleDependencies([blocking])`        | `rdd.cleanShuffleDependencies()` | Removes shuffle files and non-persisted ancestors.         | Frees memory; useful for iterative pipelines.  |

## Performance Best-Practices

| Rule                                                     | Applies To      | Why It Matters                             | Tip / Recommendation                                            |
| -------------------------------------------------------- | --------------- | ------------------------------------------ | --------------------------------------------------------------- |
| Prefer `reduceByKey` / `aggregateByKey`                  | Aggregation     | Reduces shuffle compared to `groupByKey`   | Avoid `groupByKey` for summing/aggregating values               |
| Avoid `collect()` on large RDDs                          | Actions         | Can cause driver OOM                       | Use `take()`, `foreachPartition()`, or save to storage          |
| Cache reused RDDs                                        | Persistence     | Prevents recomputation                     | `cache()` / `persist()` only if reused                          |
| Use correct storage level                                | Persistence     | Trade-off between memory and recomputation | `MEMORY_ONLY` for fast access, `MEMORY_AND_DISK` for large RDDs |
| Use `mapPartitions()` for expensive init                 | Transformations | Avoid per-record setup overhead            | Initialize DB/API connections per partition                     |
| Use `sample()` or `sampleByKey()` early                  | Sampling        | Reduces dataset before heavy ops           | Helps debugging and mitigates skew                              |
| Avoid `cartesian()` unless tiny                          | Transformations | Explodes memory & shuffle                  | Only use for small datasets                                     |
| Use `coalesce()` to reduce partitions                    | Partitioning    | Avoids full shuffle                        | Only for decreasing partitions; use `repartition()` to increase |
| Repartition for load balancing                           | Partitioning    | Prevents skewed partitions                 | After filters or uneven splits                                  |
| Avoid large closures                                     | Serialization   | Increases task size and network cost       | Broadcast large objects instead                                 |
| Persist before multiple joins                            | Joins           | Avoid recomputation                        | Cache intermediate RDDs                                         |
| Use `treeReduce()` / `treeAggregate()` for large cluster | Aggregation     | Reduces driver bottleneck                  | Improves scalability                                            |
| Control shuffle partitions                               | Shuffles        | Default may be inefficient                 | Tune `spark.sql.shuffle.partitions` or RDD partitioning         |
| Avoid excessive actions                                  | Execution       | Each action triggers job                   | Chain transformations before action                             |

