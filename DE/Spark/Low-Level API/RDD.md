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

## Shuffle

- Shuffle is the process of redistributing data across partitions in Spark so that transformations requiring co-location, grouping, or aggregation can execute correctly.
- Spark tasks operate independently per partition. If the required data for an operation (like all values for a key) is scattered across multiple partitions or nodes, Spark must move the data to the correct partitions. This movement is called a shuffle.
- Common operations that trigger shuffle -
  - ByKey operations - `reduceByKey`, `groupByKey`, `aggregateByKey` — all values for a key must be combined in a single partition.
  - Join operations - `join`, `cogroup`, `leftOuterJoin`, `rightOuterJoin`, `fullOuterJoin` — data from different RDDs must be co-located by key.
  - Repartition operations - `repartition` (always triggers shuffle), `coalesce` (only if shuffle is enabled) — to increase or decrease partitions while redistributing data.

- __Shuffle Characteristics__ -
  - Partition determinism - The number of partitions in the resulting shuffled RDD is deterministic and predictable.
  - Element ordering - Elements within each partition are not ordered by default.
    - How to achieve ordering -
      - Use `mapPartitions` + `.sorted` to sort elements locally within each partition.
      - Use `repartitionAndSortWithinPartitions` to efficiently repartition and sort in one operation.
      - Use `sortBy` to achieve a global ordering across all partitions.

- __Performance Impact__ -
  - Why shuffles are expensive -
    - Disk I/O - If data does not fit in memory, Spark spills intermediate data to disk.
    - Serialization - Data must be serialized to be transferred across the network.
    - Network I/O - Data is physically transferred between executors/nodes.
  - Task behavior during shuffle -
    - Map tasks - Organize data by the target partition and prepare it for transfer.
    - Reduce tasks - Aggregate, join, or combine data in the correct partition after transfer.
  - Memory usage during shuffle -
    - Operations like `reduceByKey` and `aggregateByKey` use in-memory hash tables to combine values before sending them across partitions.
    - Many `ByKey` operations also maintain hash tables on the reduce side to merge incoming data.
    - If data exceeds available memory, Spark spills data to disk, increasing I/O and garbage collection (GC) overhead.
  - Intermediate files -
    - Spark generates many temporary files during shuffle, one per map task per reduce partition.
    - These files persist until the RDDs are garbage collected, which can take a long time in long-running jobs.
    - The temporary storage location is controlled by `spark.local.dir`.
    - Unmanaged disk usage can cause disk exhaustion in long-running jobs.

- __Tuning & Optimization__ -
  - Reduce shuffle size -
    - Filter or sample the data before shuffle to avoid unnecessary movement.
    - Prefer `reduceByKey` over `groupByKey` to combine values on the map side, reducing shuffle volume.
  - Repartitioning strategies -
    - Use `coalesce` without shuffle to reduce the number of partitions after filtering, minimizing unnecessary data movement.
    - Use `repartition` carefully; it always triggers a full shuffle and is expensive.
  - Sorting strategies -
    - Use `repartitionAndSortWithinPartitions` when you need sorted data and repartitioning, for maximum efficiency.
  - Memory management -
    - Increase executor memory if shuffles frequently spill to disk.
    - Tune `spark.shuffle.file.buffer` and `spark.reducer.maxSizeInFlight` to optimize shuffle performance.
  - Disk management -
    - Periodically clean temporary shuffle files in long-running applications.
    - Monitor `spark.local.dir` to prevent disk space exhaustion.

## RDD Persistence

- Allows a dataset (RDD) to be stored in memory across operations.
- When persisted -
  - Each node stores computed partitions locally.
  - Future actions reuse cached partitions instead of recomputing.
- To persist an RDD - `persist()` or `cache()`.
- Persistence is lazy - data is cached only when first action runs.
- Cache happens partition-wise on executors.
- Spark’s cache is fault-tolerant – if any partition of an RDD is lost, Spark recomputes it from lineage DAG.

> [!TIP] 
> Spark also automatically persists some intermediate data in shuffle operations (e.g. `reduceByKey`) to avoid recomputing the entire input if a node fails during the shuffle. Explicitly persisting the RDD is still recommended.

- __Storage Levels__ -

| Storage Level         | Meaning (Detailed)                                                                                                 | Performance                  | Memory Usage   | Fault Tolerance                                | When to Use                                                                            | When NOT to Use                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------ | ---------------------------- | -------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| MEMORY_ONLY           | Stores deserialized objects in JVM memory. If partitions don’t fit, they are recomputed on-the-fly. Default level. | Fastest                      | High           | Lost partitions recomputed from lineage        | Iterative algorithms, ML loops, small/medium datasets that fit in memory               | Large datasets exceeding memory; memory-constrained clusters |
| MEMORY_AND_DISK       | Stores deserialized objects in memory; partitions that don’t fit spill to disk and are read when needed.           | Fast (memory), medium (disk) | Medium-High    | Lost partitions read from disk or recomputed   | Slightly larger datasets than memory; expensive transformations; multi-stage pipelines | Disk I/O slow; cheap-to-recompute datasets                   |
| MEMORY_ONLY_SER       | Stores serialized objects (byte arrays) in JVM memory. Saves memory but adds CPU overhead for deserialization.     | Medium                       | Low            | Lost partitions recomputed                     | Memory-constrained clusters; large datasets fitting in memory                          | CPU-bound workloads; latency-sensitive interactive pipelines |
| MEMORY_AND_DISK_SER   | Stores serialized objects in memory first; excess spilled to disk. Reduces memory usage while allowing reuse.      | Medium                       | Low            | Lost partitions read from disk or recomputed   | Large datasets; memory-limited clusters; expensive to recompute                        | CPU-saturated workloads                                      |
| DISK_ONLY             | Stores partitions only on disk; no memory used. All reads go to disk.                                              | Slowest                      | Minimal        | Lost partitions read from disk                 | Huge datasets; memory unavailable; recomputation very expensive                        | Iterative or low-latency workloads                           |
| MEMORY_ONLY_2         | Same as MEMORY_ONLY **but each partition is replicated on 2 cluster nodes**. Replication factor = 2.               | Fast                         | Very High      | Replica used immediately; avoids recomputation | High-availability workloads; low-latency serving; mission-critical iterative jobs      | Memory-constrained clusters; extremely large datasets        |
| MEMORY_AND_DISK_2     | Same as MEMORY_AND_DISK **but replicated on 2 nodes**. Replication factor = 2.                                     | Fast (memory), medium (disk) | Very High      | Replica used immediately                       | Critical pipelines needing redundancy and fast recovery                                | Disk or memory limited clusters                              |
| MEMORY_ONLY_SER_2     | Same as MEMORY_ONLY_SER **with 2-node replication**. Replication factor = 2.                                       | Medium                       | Medium         | Replica used instantly                         | Memory-constrained but need redundancy; interactive serving                            | CPU-heavy workloads; clusters under high CPU load            |
| MEMORY_AND_DISK_SER_2 | Same as MEMORY_AND_DISK_SER **replicated on 2 nodes**. Replication factor = 2.                                     | Medium                       | Medium         | Replica used if node fails                     | Large critical datasets with memory pressure + high fault tolerance                    | CPU-heavy pipelines; low-latency workloads                   |
| DISK_ONLY_2           | Same as DISK_ONLY **replicated on 2 nodes**. Replication factor = 2.                                               | Slow                         | High           | Replica used                                   | Very large datasets; memory unavailable; reliability critical                          | Interactive or low-latency jobs                              |
| OFF_HEAP              | Stores serialized objects outside JVM heap in off-heap memory. Requires off-heap enabled. Replication factor = 1.  | Medium                       | Low (off-heap) | Lost partitions recomputed                     | Reduce GC pressure; large memory workloads; long-running jobs                          | Off-heap not configured; insufficient off-heap memory        |


> [!NOTE]
> In Python, stored objects will always be serialized with the Pickle library, so it does not matter whether you choose a serialized level. The available storage levels in Python include `MEMORY_ONLY`, `MEMORY_ONLY_2`, `MEMORY_AND_DISK`, `MEMORY_AND_DISK_2`, `DISK_ONLY`, `DISK_ONLY_2`, and `DISK_ONLY_3`.

- __Automatic Cache Monitoring__ -
  - Spark automatically monitors memory usage on each executor/node.
  - Cached RDD partitions are removed when memory pressure occurs using Least Recently Used (LRU) policy.

- __Manual RDD Removal with `unpersist()`__ -
  - Removes an RDD from memory and/or disk cache.
  - Non-blocking by default.
  - Blocking removal - `rdd.unpersist(blocking = true)`

> [!TIP]
> `cache()` internally calls `persist(StorageLevel.MEMORY_ONLY)`.


  