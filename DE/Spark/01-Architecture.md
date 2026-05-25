# Spark Architecture

- Distributed in-memory DAG execution engine — your code runs on the Driver, data is processed in parallel across Executors
- Lazy evaluation — transformations build a DAG, nothing executes until an action (count(), collect(), write()) is called
- Fault tolerance via lineage recomputation — any lost partition is rebuilt by replaying its DAG chain from source, no data replication needed
- In-memory execution — intermediate results stay in RAM, spill to disk only under memory pressure

## Components

- __Driver__ - 
    - Runs `main()`, owns `SparkSession`, DAGScheduler, TaskScheduler
    - Single point of failure for the job
    - Assembles final results for actions like `collect()`
- __Executors__ - 
    - JVM processes on worker nodes
    - Run tasks as threads
    - Store cached partitions
    - Serve shuffle blocks to other executors
    - Stateless from the Driver's perspective
- __Cluster Manager__ - 
    - Pure resource broker - has zero knowledge of job logic
    - Allocates containers for Driver and Executors
    - Eg - YARN / K8s / Standalone
- __SparkSession__ - 
    - User-facing entry point
    - Wraps `SparkContext`
    - Lives on the Driver
    - One per application
- __DAGScheduler__ - 
    - Converts the logical DAG into `Stage` objects
    - Determines stage boundaries by identifying shuffle dependencies
    - Computes preferred task locations for data locality
- __TaskScheduler__ - 
    - Receives a `TaskSet` from `DAGScheduler`
    - Assigns individual tasks to available executor slots
    - Handles delay scheduling, retries, speculative execution
- __BlockManager__ - 
    - One instance per executor and one on the Driver
    - Manages what lives in memory vs disk on that node
    - All shuffle data, cached partitions, and broadcast variables are tracked and served through it
- __External Shuffle Service__ - 
    - Optional daemon running on each worker node, outside executor JVMs
    - Serves shuffle files independently of executor lifecycle
    - This is what makes dynamic allocation safe — executor can die, shuffle data survives

> [!NOTE]
> `DAGScheduler` thinks in data dependencies and partition lineage. 
> 
> `TaskScheduler` thinks in slots, resources, and scheduling policy.

## Execution Flow

- Action called -
    - Catalyst optimizes → Physical Plan
    - DAGScheduler cuts plan into Stages at every shuffle boundary
    - TaskScheduler assigns Tasks (1 per partition) to Executor slots
    - Executors run tasks as threads, each processing one partition
    - shuffle data written to local disk between stages
    - Final result written to sink or sent back to Driver

### 1. Startup

- `spark-submit` → 
    - Driver JVM starts → 
    - `SparkSession` (and`SparkContext`) initializes → 
    - Driver registers with Cluster Manager → 
    - Cluster Manager launches Executor JVMs on worker nodes → 
    - Executors heartbeat back to Driver
- Once all executors have registered, Driver has full topology — executor ID, node, cores, memory

### 2. DAG construction (lazy)

- Every transformation (`filter`, `select`, `join`, `groupBy`) adds a node to the DAG — zero execution happens
- DataFrame/SQL path - 
    - Action triggers Catalyst pipeline — Unresolved Logical Plan → 
    - Analyzer resolves column names and types → 
    - Optimizer applies rules (predicate pushdown, column pruning, constant folding) → 
    - Planner selects physical operators → 
    - Physical Plan
- RDD path - DAG built directly from RDD lineage, no Catalyst involved
- Physical plan is a tree of operators - `FileScan`, `Filter`, `SortMergeJoin`, `HashAggregate`, `Exchange` (shuffle node)

### 3. DAG → Stages

- DAGScheduler walks the physical plan bottom-up and cuts a new stage at every wide dependency
- Narrow dependency — 
    - One input partition maps to exactly one output partition
    - Eg - `filter`, `select`, `map`, `union`
    - Pipelines within a stage with zero network I/O
- Wide dependency — 
    - One output partition depends on multiple input partitions
    - Requires full network redistribution
    - Eg - `groupBy`, `join`, `repartition`, `distinct`
- Everything within a stage runs as a fully pipelined chain on a single partition — no intermediate materialization, no network
- DAGScheduler also tags each task with preferred locations — HDFS block locations for input stages, cached partition locations for stages reading persisted data

### 4. Stages → Tasks

- Each stage produces a TaskSet — one task per partition, all tasks in the set are independent and can run in parallel
- TaskScheduler assigns tasks using delay scheduling — waits briefly for a data-local slot (partition already on the same node), falls back to rack-local, then any available slot
- Task is serialized along with its closure — every variable the task lambda references gets captured and shipped to the executor

> [!TIP]
> Large objects in closures are a common source of OOM and slow task dispatch

### 5. Task Execution

- Executor runs tasks as threads, not processes — 
    - All tasks on an executor share one JVM heap
    - GC pressure from one task affects all concurrent tasks
- `spark.executor.cores` controls concurrency — that many tasks run simultaneously per executor
- One task processes exactly one partition — reads it, applies the full chain of narrow operators in a single pass, produces output
- Shuffle stage - output written to local disk
- Final stage - output written to sink, or serialized and sent back to Driver

### 6. Shuffle

- Map-side - 
    - Each task writes one sorted shuffle file to local disk with a separate partition index file
    - One file per mapper regardless of partition count (`SortShuffle` default)
- Reduce-side  
    - Each task fetches its partition's slice from every map-side node over HTTP
    - BlockManager on each executor handles serving these blocks
- Shuffle is the most expensive operation in Spark — full network transfer, disk I/O on both sides, sort overhead
- With External Shuffle Service -
    - Shuffle files are served by the node daemon, not the executor JVM
    - Executor can be killed and relaunched without losing shuffle output
    - This is the prerequisite for safe dynamic allocation

### 7. Result

- `collect()`, `take()`, `first()` — 
    - Executor tasks serialize results and send back to Driver over the network
    - Driver assembles and returns to user code
    - Risk of Driver OOM if result set is large
- `df.write()` — 
    - Executors write directly to the sink (HDFS, S3, etc.)
    - Driver only tracks task completion

## Partitions

- Every RDD/DataFrame is divided into partitions — partition is the atomic unit of parallelism
- One task processes one partition - partition count directly determines parallelism
- Input partitions are determined by source - 
    - HDFS file splits (default $128 MB$ blocks)
    - JDBC ranges (`numPartitions`)
    - Kafka topic partition count
- Shuffle output partitions — 
    - Controlled by `spark.sql.shuffle.partitions`, default $200$
    - Default $200$ is almost always wrong
- Too few partitions → tasks too large, executor OOM, stragglers, no parallelism
- Too many partitions → scheduler overhead dominates, tiny tasks, excessive small shuffle files, high metadata load on Driver

> [!TIP]
> Target $~128 MB$ per partition
>
> Partition count should be a multiple of total executor cores to avoid wasted slots in the last wave

## Fault Tolerance

- Lineage is the recomputation backbone — 
    - Spark tracks the full transformation chain for every partition
    - Recovery is deterministic replay, not data redundancy
- __Driver failure__ — 
    - Job is dead
    - No automatic recovery in batch
    - Structured Streaming recovers from checkpoint on restart and replays from last committed offset
- __Executor failure__ — 
    - Driver detects via missed heartbeat
    - Tasks rescheduled on surviving executors
    - If that executor held shuffle output and no External Shuffle Service is running, parent stage reruns to regenerate lost shuffle data
- __Stage failure__ — 
    - DAGScheduler resubmits the stage
    - f shuffle data from a parent stage is also lost, parent stages rerun recursively
- __Task failure__ — 
    - Retried up to `spark.task.maxFailures` (default $4$) on a different executor
    - All retries exhausted → stage fails

## SparkConf

- A key-value store for Spark configuration — every tunable in Spark (memory, cores, serializer, shuffle behavior, etc.) flows through `SparkConf`
- Immutable - changes after `SparkContext` is created have no effect
- Lives on the Driver — not distributed, not shared with executors directly
    - Executor-side config is serialized into `TaskDescription` and shipped with each task

### Priority order (highest to lowest)

- Programmatic — `new SparkConf().set("spark.executor.memory", "4g")`
- `spark-submit` flags — `--executor-memory 4g` (these set the same keys programmatically under the hood)
- `spark-defaults.conf` — file on the Driver's classpath (`$SPARK_HOME/conf/spark-defaults.conf`)
- Spark's hardcoded defaults — defined in source, used when nothing else sets the key

### Creating SparkConf

- Python -
    ```
    from pyspark import SparkConf, SparkContext
    from pyspark.sql import SparkSession

    # via SparkConf
    conf = SparkConf() \
        .setAppName("MyApp") \
        .setMaster("yarn") \
        .set("spark.executor.memory", "4g")

    # Preferred in modern Spark — builder handles SparkConf internally
    spark = SparkSession.builder \  
        .appName("MyApp") \
        .master("yarn") \
        .config("spark.executor.memory", "4g") \
        .getOrCreate()
    ```

- Scala -
    ```
    import org.apache.spark.SparkConf
    import org.apache.spark.sql.SparkSession

    // via SparkConf
    val conf = new SparkConf()
        .setAppName("MyApp")
        .setMaster("yarn")
        .set("spark.executor.memory", "4g")

    // Preferred
    val spark = SparkSession.builder()
        .appName("MyApp")
        .master("yarn")
        .config("spark.executor.memory", "4g")
        .config(conf)   // merge an existing SparkConf
        .getOrCreate()
    ```

- Java -
    ```
    import org.apache.spark.SparkConf;
    import org.apache.spark.sql.SparkSession;

    // via SparkConf
    SparkConf conf = new SparkConf()
        .setAppName("MyApp")
        .setMaster("yarn")
        .set("spark.executor.memory", "4g");

    // Preferred
    SparkSession spark = SparkSession.builder()
        .appName("MyApp")
        .config(conf)
        .getOrCreate();
    ```

### How config gets to Executors

- `SparkConf` lives on the Driver
- When the Driver launches executors, it passes a filtered subset of config via the Cluster Manager
- Executor JVM starts with those configs as system properties
- Task-level config travels inside `TaskDescription` serialized with each task

### Accessing config at runtime

- Python -
    ```
    conf = spark.sparkContext.getConf()
    conf.get("spark.executor.memory")             # throws if not set
    conf.get("spark.some.key", "default_value")   # with fallback
    dict(conf.getAll())                           # all as dict
    ```

- Scala -
    ```
    val conf = spark.sparkContext.getConf
    conf.get("spark.executor.memory")             // throws if not set
    conf.getOption("spark.some.key")              // Option[String]
    conf.getAll                                   // Array[(String, String)]
    ```

- Java -
    ```
    SparkConf conf = spark.sparkContext().getConf();
    conf.get("spark.executor.memory");
    conf.getOption("spark.some.key");             // scala.Option<String>
    conf.getAll();                                // Tuple2<String,String>[]
    ```

## SparkContext

- The entry point to Spark's core engine — before `SparkSession` existed, `SparkContext` was the only way into Spark
- One `SparkContext` per JVM — enforced hard
    - Attempting to create a second one throws an exception unless the first is stopped
- Owns the connection to the cluster — all communication with the Cluster Manager flows through it
- Lives on the Driver — initializes and holds references to every core internal component - `DAGScheduler`, `TaskScheduler`, `BlockManager`, `SparkEnv`, `MapOutputTracker`, `BroadcastManager`

### What SparkContext initializes

- When `SparkContext` is created -
    - Validates and locks the `SparkConf` — frozen from this point
    - Sets up `SparkEnv` — initializes serializer, RPC environment, `BlockManager`, `MapOutputTracker`, shuffle manager, memory manager
    - Creates `DAGScheduler`
    - Creates `TaskScheduler` + `SchedulerBackend` (cluster-manager-specific)
    - Starts `BlockManagerMaster` on Driver
    - Initializes `BroadcastManager`
    - Registers with Cluster Manager and requests initial executors
    - Starts heartbeat thread to executors

> [!NOTE]
> This is why `SparkContext` creation is expensive and slow — it's not just an object instantiation, it's bootstrapping the entire distributed runtime.

### Creating SparkContext

- In modern Spark you get it via `spark.sparkContext` i.e. `SparkSession` wraps `SparkContext`
- Direct creation is only needed for pure RDD workloads or testing
- Python -
    ```
    # Modern - access SparkContext from session
    sc = spark.sparkContext                    

    # Direct — only for pure RDD or testing
    sc = SparkContext(conf=sparkConf)
    ```

- Scala -
    ```
    val sc = spark.sparkContext
    val sc = new SparkContext(sparkConf)
    ```

- Java -
    ```
    SparkContext sc = spark.sparkContext();
    SparkContext sc = new SparkContext(sparkConf);
    ```

### Use-cases

| Category              | What you can do                                                                        | Key methods                                                                                                                                   |
| --------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **RDD Creation**      | Create RDDs from collections, files, Hadoop formats, ranges, or other RDDs             | `parallelize`, `textFile`, `wholeTextFiles`, `binaryFiles`, `sequenceFile`, `hadoopFile`, `newAPIHadoopFile`, `range`, `union`, `emptyRDD`    |
| **Shared Variables**  | Broadcast read-only data to executors and maintain distributed counters                | `broadcast`, `accumulator`, `longAccumulator`, `doubleAccumulator`, `collectionAccumulator`                                                   |
| **Job Control**       | Group, describe, monitor, and cancel running jobs                                      | `setJobGroup`, `cancelJobGroup`, `cancelJob`, `cancelAllJobs`, `setJobDescription`                                                            |
| **Scheduling**        | Assign jobs to Fair Scheduler pools and configure thread-local scheduling properties   | `setLocalProperty`, `getLocalProperty`                                                                                                        |
| **Dynamic Resources** | Ship files and JARs to executors dynamically at runtime                                | `addFile`, `addJar`, `listFiles`, `listJars`, `SparkFiles.get`                                                                                |
| **Cluster Info**      | Inspect application metadata, executor state, deployment mode, and cluster parallelism | `applicationId`, `applicationAttemptId`, `master`, `deployMode`, `version`, `defaultParallelism`, `getExecutorMemoryStatus`, `getExecutorIds` |
| **Status Tracking**   | Query live job, stage, and executor information                                        | `statusTracker.getActiveJobIds`, `getActiveStageIds`, `getJobInfo`, `getStageInfo`, `getExecutorInfos`                                        |
| **Persistence**       | View all currently cached or persisted RDDs                                            | `getPersistentRDDs`                                                                                                                           |
| **Checkpointing**     | Configure checkpoint directories for RDD lineage truncation and fault tolerance        | `setCheckpointDir`, `getCheckpointDir`                                                                                                        |
| **Configuration**     | Access the immutable runtime `SparkConf`                                               | `getConf`, `getConf().get`, `getConf().getAll`                                                                                                |
| **Lifecycle**         | Create, inspect, and terminate the `SparkContext`                                      | `getOrCreate`, `isStopped`, `stop`                                                                                                            |
