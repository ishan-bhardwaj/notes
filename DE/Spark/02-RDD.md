# RDD

- **RDD (Resilient Distributed Dataset)** — the core distributed data abstraction in Spark; an immutable, partitioned collection of records distributed across executor JVMs
- **Immutable** — once created, an RDD cannot be modified; every transformation produces a new RDD; this is what makes lineage-based fault tolerance possible
- **Lazy** — transformations record intent in the DAG; nothing materializes until an action forces execution
- **Typed** — `RDD[T]` carries the element type at compile time (Scala/Java); Python erases this at the JVM boundary
- **Schema-less** — unlike DataFrames, RDDs have no column metadata, no Catalyst optimization, no predicate pushdown; you own the data layout entirely
- **Low-level API** — gives direct control over partitioning, serialization, locality; used when DataFrame/Dataset abstractions are insufficient (custom binary formats, iterative ML algorithms, graph processing)

> [!NOTE]
> `DataFrame` is `Dataset[Row]` — a Dataset where the type parameter is an untyped `Row`.
>
> Under the hood, a DataFrame/Dataset compiles down to an `RDD` of `InternalRow` for execution. The optimizer operates above RDD level — by the time tasks run, it's RDD mechanics all the way down.

- When to use RDDs -
    - Custom partitioners that require business logic unavailable in DataFrame API
    - Processing non-tabular data — binary records, variable-length blobs, custom serialization formats
    - Iterative algorithms where you cache intermediate results and explicitly control recomputation
    - Interfacing with legacy Hadoop `InputFormat`/`OutputFormat` directly
    - Fine-grained control over data locality or task-level side effects

## RDD Creation

- **From collection** — `sc.parallelize(seq, numSlices)` — partition count defaults to `sc.defaultParallelism`
- **From file** — `sc.textFile(path, minPartitions)` — one partition per HDFS block by default
- **From Hadoop format** — `sc.hadoopFile`, `sc.newAPIHadoopFile` — any `InputFormat` supported
- **From another RDD** — transformation returns a new RDD; child holds reference to parent in lineage
- **From DataFrame** — `df.rdd` — converts to `RDD[Row]`; triggers schema erasure, loses Tungsten optimizations

```python
sc = spark.sparkContext

rdd = sc.parallelize([1, 2, 3, 4], numSlices=4)
rdd = sc.textFile("hdfs://path/data.txt", minPartitions=10)
rdd = sc.sequenceFile("hdfs://path/", keyClass="...", valueClass="...")
rdd = df.rdd                                                                 # DataFrame → RDD[Row]

- Scala -
```scala
val rdd = sc.parallelize(Seq(1,2,3,4), numSlices = 4)
val rdd = sc.textFile("hdfs://path/data.txt", minPartitions = 10)
val rdd = df.rdd                                // RDD[Row]
```