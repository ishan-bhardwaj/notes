# Spark Architecture

- Distributed in-memory DAG execution engine - 
    - Code runs on the Driver
    - Data is processed in parallel across Executors
- Lazy evaluation - 
    - Transformations build a DAG
    - Nothing executes until an action (`count()`, `collect()`, `write()`) is called
- Fault tolerance via lineage recomputation - 
    - Any lost partition is rebuilt by replaying its DAG chain from source
    - No data replication needed
- In-memory execution - 
    - Intermediate results stay in memory
    - Spill to disk only under memory pressure

## Components

- __Driver__ - 
    - Runs `main()`, owns `SparkSession`, `DAGScheduler`, `TaskScheduler`
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
    - This is what makes dynamic allocation safe - executor can die, shuffle data survives

---

## Execution Flow

- Action called -
    - Catalyst optimizes → Physical Plan
    - DAGScheduler cuts plan into Stages at every shuffle boundary
    - TaskScheduler assigns Tasks ($1$ per partition) to Executor slots
    - Executors run tasks as threads, each processing one partition
    - Shuffle data written to local disk between stages
    - Final result written to sink or sent back to Driver

### 1. Startup

- `spark-submit` → 
    - Driver JVM starts → 
    - `SparkSession` (and`SparkContext`) initializes → 
    - Driver registers with Cluster Manager → 
    - Cluster Manager launches Executor JVMs on worker nodes → 
    - Executors heartbeat back to Driver
- Once all executors have registered, Driver has full topology - executor ID, node, cores, memory

### 2. DAG construction (lazy)

- Every transformation (`filter`, `select`, `join`, `groupBy`) adds a node to the DAG - zero execution happens
- DataFrame/SQL path - 
    - Action triggers Catalyst pipeline - 
        - Unresolved Logical Plan
        - Analyzer resolves column names and types
        - Optimizer applies rules (predicate pushdown, column pruning, constant folding)
        - Planner selects physical operators
        - Physical Plan
- RDD path - 
    - DAG built directly from RDD lineage
    - No Catalyst involved

### 3. DAG → Stages

- DAGScheduler walks the physical plan bottom-up and cuts a new stage at every wide dependency
- Narrow dependency - 
    - One input partition maps to exactly one output partition
    - Pipelines within a stage with zero network I/O
    - Eg - `filter`, `select`, `map`, `union`
- Wide dependency - 
    - One output partition depends on multiple input partitions
    - Requires full network redistribution
    - Eg - `groupBy`, `join`, `repartition`, `distinct`
- DAGScheduler also tags each task with preferred locations - HDFS block locations for input stages, cached partition locations for stages reading persisted data

### 4. Stages → Tasks

- Each stage produces a `TaskSet` - 
    - One task per partition
    - All tasks in the set are independent and can run in parallel
- TaskScheduler assigns tasks using delay scheduling - 
    - Waits briefly for a data-local slot (partition already on the same node)
    - Falls back to rack-local, then any available slot
- Task is serialized along with its closure - every variable the task lambda references gets captured and shipped to the executor

> [!WARNING]
> Large objects in closures are a common source of OOM and slow task dispatch

### 5. Task Execution

- Executor runs tasks as threads, not processes - 
    - All tasks on an executor share one JVM heap
    - GC pressure from one task affects all concurrent tasks
- `spark.executor.cores` controls concurrency - that many tasks run simultaneously per executor
- One task processes exactly one partition - reads it, applies the full chain of narrow operators in a single pass, produces output
- Shuffle stage - output written to local disk
- Final stage - output written to sink, or serialized and sent back to Driver

### 6. Shuffle

- Map-side - 
    - Each task writes one sorted shuffle file to local disk with a separate partition index file
    - One file per mapper regardless of partition count (`SortShuffle` default)
- Reduce-side  
    - Each task fetches its partition's slice from every map-side node over HTTP
    - BlockManager on each executor handles serving these blocks
- With External Shuffle Service -
    - Shuffle files are served by the node daemon, not the executor JVM
    - Executor can be killed and relaunched without losing shuffle output
    - This is the prerequisite for safe dynamic allocation

### 7. Result

- `collect()`, `take()`, `first()` - 
    - Executor tasks serialize results and send back to Driver over the network
    - Driver assembles and returns to user code
    - Risk of Driver OOM if result set is large
- `df.write()` - 
    - Executors write directly to the sink (HDFS, S3, etc.)
    - Driver only tracks task completion

---

## Partitions

- Every RDD/DataFrame is divided into partitions
- One task processes one partition - partition count directly determines parallelism
- Input partitions are determined by source - 
    - HDFS file splits (default $128 MB$ blocks)
    - JDBC ranges (`numPartitions`)
    - Kafka topic partition count
- Shuffle output partitions - 
    - Controlled by `spark.sql.shuffle.partitions`, default $200$
    - Default $200$ is almost always wrong
- Too few partitions → tasks too large, executor OOM, stragglers, no parallelism
- Too many partitions → scheduler overhead dominates, tiny tasks, excessive small shuffle files, high metadata load on Driver

> [!TIP]
> Target $~128 MB$ per partition
>
> Partition count should be a multiple of total executor cores to avoid wasted slots in the last wave

---

## Fault Tolerance

- Lineage is the recomputation backbone - 
    - Spark tracks the full transformation chain for every partition
    - Recovery is deterministic replay, not data redundancy
- __Driver failure__ - 
    - Job is dead
    - No automatic recovery in batch
    - Structured Streaming recovers from checkpoint on restart and replays from last committed offset
- __Executor failure__ - 
    - Driver detects via missed heartbeat
    - Tasks rescheduled on surviving executors
    - If that executor held shuffle output and no External Shuffle Service is running, parent stage reruns to regenerate lost shuffle data
- __Stage failure__ - 
    - DAGScheduler resubmits the stage
    - If shuffle data from a parent stage is also lost, parent stages rerun recursively
- __Task failure__ - 
    - Retried up to `spark.task.maxFailures` (default $4$) on a different executor
    - All retries exhausted - stage fails

---

## SparkConf

- A key-value store for Spark configuration - every tunable in Spark config (memory, cores, serializer, shuffle behavior, etc.) flows through it
- Immutable - changes after `SparkContext` is created have no effect
- Lives on the Driver - not distributed, not shared with executors directly
    - Executor-side config is serialized into `TaskDescription` and shipped with each task

### Priority order (highest to lowest)

- Programmatic - `new SparkConf().set("spark.executor.memory", "4g")`
- `spark-submit` flags - `--executor-memory 4g` (these set the same keys programmatically under the hood)
- `spark-defaults.conf` - file on the Driver's classpath (`$SPARK_HOME/conf/spark-defaults.conf`)
- Spark's hardcoded defaults - defined in Spark source code

> [!NOTE]
> `spark-defaults.conf` is read only by `spark-submit` on the _submitting_ machine. In cluster mode, the remote Driver does not load its own node-local `spark-defaults.conf`
>
> In both client and cluster mode -
>   - `spark-submit` loads configs locally
>   - Builds the final Spark configuration
>   - Sends that configuration to the Driver/cluster

### Creating SparkConf

- Python -
    ```python
    from pyspark import SparkConf, SparkContext
    from pyspark.sql import SparkSession

    # via SparkConf
    conf = SparkConf() \
        .setAppName("MyApp") \
        .setMaster("yarn") \
        .set("spark.executor.memory", "4g")

    # Preferred in modern Spark - builder handles SparkConf internally
    spark = SparkSession.builder \  
        .appName("MyApp") \
        .master("yarn") \
        .config("spark.executor.memory", "4g") \
        .getOrCreate()
    ```

- Scala -
    ```scala
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

### How config gets to Executors

- `SparkConf` lives on the Driver
- When the Driver launches executors, it passes a filtered subset of config via the Cluster Manager
- Executor JVM starts with those configs as system properties
- Task-level config travels inside `TaskDescription` serialized with each task

### Accessing config at runtime

- Python -
    ```python
    conf = spark.sparkContext.getConf()
    conf.get("spark.executor.memory")               # throws if not set
    conf.get("spark.some.key", "default_value")     # with fallback
    dict(conf.getAll())                             # all as dict
    ```

- Scala -
    ```scala
    val conf = spark.sparkContext.getConf
    conf.get("spark.executor.memory")               // throws if not set
    conf.get("spark.some.key", "default_value")     // with fallback
    conf.getOption("spark.some.key")                // Option[String]
    conf.getAll                                     // Array[(String, String)]
    ```

---

## SparkContext

- The entry point to Spark's core engine - before `SparkSession` existed, `SparkContext` was the only way into Spark
- One `SparkContext` per JVM - enforced hard
    - Attempting to create a second one throws an exception unless the first is stopped
- Owns the connection to the cluster - all communication with the Cluster Manager flows through it
- Lives on the Driver - initializes and holds references to every core internal component - `DAGScheduler`, `TaskScheduler`, `BlockManager`, `SparkEnv`, `MapOutputTracker`, `BroadcastManager`

### What SparkContext initializes

- When `SparkContext` is created -
    - Validates and locks the `SparkConf` - frozen from this point
    - Sets up `SparkEnv` - initializes serializer, RPC environment, `BlockManager`, `MapOutputTracker`, shuffle manager, memory manager
    - Creates `DAGScheduler`
    - Creates `TaskScheduler` + `SchedulerBackend` (cluster-manager-specific)
    - Starts `BlockManagerMaster` on Driver
    - Initializes `BroadcastManager`
    - Registers with Cluster Manager and requests initial executors
    - Starts heartbeat thread to executors

> [!NOTE]
> This is why `SparkContext` creation is expensive and slow - it's not just an object instantiation, it's bootstrapping the entire distributed runtime.

### Creating SparkContext

- In modern Spark you get it via `spark.sparkContext` - `SparkSession` wraps `SparkContext`
- Direct creation is only needed for pure RDD workloads or testing
- Python -
    ```python
    # Modern - access SparkContext from session
    sc = spark.sparkContext                    

    # Direct - only for pure RDD or testing
    sc = SparkContext(conf=sparkConf)
    ```

- Scala -
    ```scala
    val sc = spark.sparkContext
    val sc = new SparkContext(sparkConf)
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

---

## SparkSession

- Unified entry point to Spark introduced in Spark 2.0 - replaced `SparkContext`, `SQLContext`, `HiveContext`, `StreamingContext` as separate entry points
- One `SparkSession` per application by default - but multiple sessions with isolated configs are possible via `newSession()`
- Wraps `SparkContext` - 
    - There is always exactly one `SparkContext` underneath
    - Multiple `SparkSession` instances share the same `SparkContext`
- Lives on the Driver

### What SparkSession owns

| Component       | Role                                                                                 |
| --------------- | ------------------------------------------------------------------------------------ |
| `SparkContext`  | Core engine - cluster connection, scheduling, RDDs                                   |
| `SessionState`  | Per-session state - catalog, analyzer, optimizer, planner, function registry         |
| `SharedState`   | Cross-session shared state - `SparkContext`, Hive metastore connection, shared cache |
| `RuntimeConfig` | Thin wrapper over `SparkConf` + session-level SQL configs                            |
| `Catalog`       | User-facing API to databases, tables, functions, caches                              |

> [!NOTE]
> `SessionState` is per session - each `newSession()` gets its own analyzer, optimizer, temp views, and function registry.
>
> `SharedState` is shared across all sessions in the same JVM - same `SparkContext`, same metastore connection, same cached tables.

### Creating SparkSession

- Python -
    ```python
    from pyspark.sql import SparkSession

    spark = SparkSession.builder \
        .appName("MyApp") \
        .master("yarn") \
        .config("spark.executor.memory", "4g") \
        .getOrCreate()                              # returns existing session if one exists
    ```

- Scala -
    ```scala
    import org.apache.spark.sql.SparkSession

    val spark = SparkSession.builder()
        .appName("MyApp")
        .master("yarn")
        .config("spark.executor.memory", "4g")
        .getOrCreate()                              // returns existing session if one exists
    ```

- `getOrCreate` vs `newSession` vs `cloneSession`
    - `getOrCreate` - returns existing session if one exists in this JVM
    - `newSession` - new session with isolated `SessionState`, but same `SharedState`
    - `cloneSession` - internal; used by Spark internally for parallel query execution

### RuntimeConfig - `spark.conf`

```python
# Set SQL-level config at runtime (subset of configs are mutable at runtime)
spark.conf.set("spark.sql.shuffle.partitions", "400")
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", str(50 * 1024 * 1024))

# Get
spark.conf.get("spark.sql.shuffle.partitions")
spark.conf.get("spark.some.key", "default_value")           # with fallback

# Check if modifiable at runtime
spark.conf.isModifiable("spark.sql.shuffle.partitions")     # True
spark.conf.isModifiable("spark.executor.memory")            # False - JVM already started

# Get all
spark.conf.getAll                                           # dict in Python
```

> [!NOTE]
> Not all configs are mutable at runtime 
>   - Anything that controls JVM startup (`spark.executor.memory`, `spark.executor.cores`) is frozen at `SparkContext` creation.
>   - SQL optimizer configs (`spark.sql.shuffle.partitions`, `spark.sql.autoBroadcastJoinThreshold`) are mutable per session.

### Stopping SparkSession

- `spark.stop()` -
    - Stops `SparkSession` and underlying `SparkContext`
    - Releases all cluster resources
    - Always call in `finally` block in scripts

---

## Catalog API

```python
# Databases
spark.catalog.listDatabases()
spark.catalog.setCurrentDatabase("analytics")
spark.catalog.currentDatabase()

# Tables
spark.catalog.listTables()                              # tables in current DB
spark.catalog.listTables("other_db")                    # tables in specific DB
spark.catalog.tableExists("my_table")
spark.catalog.tableExists("other_db", "my_table")

# Columns
spark.catalog.listColumns("my_table")

# Functions
spark.catalog.listFunctions()
spark.catalog.functionExists("my_udf")

# Cache management
spark.catalog.cacheTable("my_table")
spark.catalog.cacheTable("my_table", storageLevel=StorageLevel.MEMORY_AND_DISK)
spark.catalog.uncacheTable("my_table")
spark.catalog.isCached("my_table")
spark.catalog.clearCache()                              # unpersist all cached tables

# Refresh - force re-read of metadata after external changes
spark.catalog.refreshTable("my_table")                  # clears cached metadata + data
spark.catalog.refreshByPath("hdfs://path/")             # refresh tables pointing to this path
```

---

## SparkEnv

- The runtime environment container for a Spark process - holds every infrastructure-level component a Driver or Executor needs to function
- One `SparkEnv` per JVM process - one on the Driver, one on each Executor
- Created during `SparkContext` initialization on the Driver
- Created on each Executor when the Executor JVM starts and registers with the Driver
- Not part of the public API

### What SparkEnv holds

| Component                 | Role                                                                                                    |
| ------------------------- | ------------------------------------------------------------------------------------------------------- |
| `SerializerManager`       | Manages serialization for shuffle, RDD persistence, broadcasts - picks codec, handles compression       |
| `RpcEnv`                  | Netty-based RPC layer - all Driver ↔ Executor communication goes through this                           |
| `BlockManager`            | Manages storage of blocks (shuffle data, cached RDD partitions, broadcast chunks) in memory and on disk |
| `MapOutputTracker`        | Tracks shuffle map output locations - which executor holds which output partition                       |
| `ShuffleManager`          | Manages shuffle write and read - `SortShuffleManager` is the default                                    |
| `BroadcastManager`        | Creates and distributes broadcast variables - uses `TorrentBroadcast` by default                        |
| `MemoryManager`           | Controls memory allocation between execution and storage - `UnifiedMemoryManager` by default            |
| `OutputCommitCoordinator` | Ensures only one task commits output for a given partition - prevents duplicate writes                  |
| `SecurityManager`         | Handles authentication, encryption, ACLs                                                                |
| `MetricsSystem`           | Collects and reports executor/Driver metrics to sinks (`Graphite`, `Prometheus`, etc.)                  |

### Driver SparkEnv vs Executor SparkEnv

- Driver and Executor both have a `SparkEnv` but they are configured differently
- Driver `SparkEnv` runs `MapOutputTrackerMaster`, `BlockManagerMaster` - the authoritative coordinators
- Executor `SparkEnv` runs `MapOutputTrackerWorker`, `BlockManagerSlave` - the worker-side counterparts that report to the Master
- Same class, different roles determined by the `isDriver` flag at construction time
- How `SparkEnv` is created -
    - Driver side - 
        ```scala
        // Inside SparkContext initialization - not public API
        val env = SparkEnv.createDriverEnv(...)
        SparkEnv.set(env)
        ```

    - Executor side -
        ```scala
        // Inside CoarseGrainedExecutorBackend.main() - called when Executor JVM starts
        val env = SparkEnv.createExecutorEnv(...)
        ```

### Accessing SparkEnv

- `SparkEnv` is not accessible from PySpark tasks - 
    - Python tasks run in a separate Python process (not the Executor JVM)
    - `SparkEnv` lives in the JVM
    - `py4j` bridge only works on the Driver side
- Python (via `py4j` - internal access only) -
    ```python
    # Not recommended in production - internal API, can break across versions
    jvm_env = spark.sparkContext._jvm.org.apache.spark.SparkEnv.get()
    block_manager = jvm_env.blockManager()
    shuffle_manager = jvm_env.shuffleManager()
    memory_manager = jvm_env.memoryManager()
    ```

- Scala -
    ```scala
    import org.apache.spark.SparkEnv

    // Get current SparkEnv (works on both Driver and Executor)
    val env = SparkEnv.get

    // Access individual components
    val blockManager = env.blockManager
    val shuffleManager = env.shuffleManager
    val memoryManager = env.memoryManager
    val mapOutputTracker = env.mapOutputTracker
    val rpcEnv = env.rpcEnv
    val serializer = env.serializer
    val serializerManager = env.serializerManager
    val broadcastManager = env.broadcastManager
    val outputCommitCoordinator = env.outputCommitCoordinator
    val metricsSystem = env.metricsSystem
    ```

---

## SparkEnv Components

### SerializerManager

- Sits above the raw serializer (`KryoSerializer` / `JavaSerializer`) and adds compression, encryption, and per-block decisions about whether to compress
- Decides whether to compress a block based on its type - 
    - Shuffle data is always compressed if `spark.shuffle.compress=true`
    - Broadcast data compressed if `spark.broadcast.compress=true`
    - RDD partitions compressed if `spark.rdd.compress=true`
- Handles spill compression separately - `spark.shuffle.spill.compress=true`

```scala
val sm = SparkEnv.get.serializerManager

// Serialize a stream with compression - used internally for shuffle writes
val compressed = sm.wrapForCompression(blockId, outputStream)

// Check if a block should be encrypted
sm.shouldEncrypt(blockId)
```

### RpcEnv

- Netty-based message-passing layer - every Driver ↔ Executor interaction is an RPC call through this
- Replaces the old Akka-based system (removed in Spark 2.0)
- `RpcEndpoint` - 
    - Named actor-like object that handles messages
    - `DAGScheduler`, `BlockManagerMaster`, `MapOutputTrackerMaster` are all `RpcEndpoint`s registered in the Driver's `RpcEnv`
- `RpcEndpointRef` - 
    - Serializable reference to an endpoint
    - Executors hold refs to Driver endpoints

```scala
val rpcEnv = SparkEnv.get.rpcEnv

// Look up an endpoint by name - used internally
val ref = rpcEnv.setupEndpointRef(driverAddress, "MapOutputTracker")

// Ask - request/response (Future-based)
val result = ref.ask[MapOutputStatistics](GetMapOutputStatistics(shuffleId))

// Send - fire and forget
ref.send(ExecutorRegistered(executorId))
```

### BlockManager

- Every block of data in Spark - RDD partition, shuffle file chunk, broadcast piece - has a `BlockId` and is managed by `BlockManager`
- On each executor, `BlockManager` decides whether a block lives in memory (`MemoryStore`) or on disk (`DiskStore`)
- `BlockManagerMaster` on the Driver maintains a registry of all blocks across all executors - knows which executor holds which block

```scala
val bm = SparkEnv.get.blockManager

// Put a block into memory
bm.putSingle(blockId, value, storageLevel, tellMaster = true)

// Get a block
val block = bm.getSingle[MyType](blockId)

// Get remote block from another executor
val remoteBlock = bm.getRemoteBytes(blockId)

// Remove a block
bm.removeBlock(blockId, tellMaster = true)

// Check block status
bm.getStatus(blockId)   // Option[BlockStatus] - mem size, disk size, storage level
```

### MapOutputTracker

- After a shuffle map stage completes, each map task registers its output location with `MapOutputTrackerMaster` on the Driver
- Reduce tasks query `MapOutputTrackerWorker` on their executor - worker fetches locations from Master if not cached locally
- This is how reduce tasks know which executor to fetch each shuffle partition from

```scala
// On Driver - MapOutputTrackerMaster
val tracker = SparkEnv.get.mapOutputTracker.asInstanceOf[MapOutputTrackerMaster]

// Get all output locations for a shuffle
val statuses = tracker.getMapSizesByExecutorId(shuffleId, startPartition, endPartition)

// Check if all map outputs for a shuffle are registered
tracker.containsShuffle(shuffleId)

// Unregister all outputs for a shuffle (called when shuffle data is lost)
tracker.unregisterShuffle(shuffleId)
```

### ShuffleManager

- Single pluggable interface for shuffle write and read - default is `SortShuffleManager`
- `SortShuffleManager` chooses one of three shuffle write paths -
  - `SortShuffleWriter` (default) -
    - Writes records into an in-memory sorter, spills sorted runs to disk, then merges them into final shuffle output.
  - `BypassMergeSortShuffleWriter` -
    - Fast path for simple shuffles - does not sort records in memory
    - Instead, writes directly into one temporary file per reduce partition, then concatenates/merges them into the final shuffle data file
    - Used when -
      - There is no map-side aggregation
      - Number of reduce partitions is small enough - `<= spark.shuffle.sort.bypassMergeThreshold` (default $200$)
  - `UnsafeShuffleWriter` -
    - Optimized binary shuffle writer
    - Stores records as serialized binary data and sorts binary pointers instead of JVM objects
    - Used when -
      - There is no map-side aggregation
      - Serializer supports object relocation
      - Partition count is within supported limits

```scala
val sm = SparkEnv.get.shuffleManager

// Register a shuffle - returns a ShuffleHandle that determines write path
val handle = sm.registerShuffle(shuffleId, numMaps, dependency)

// Get a writer for a map task
val writer = sm.getWriter[K, V](handle, mapId, context, metrics)
writer.write(records.iterator)
writer.stop(success = true)

// Get a reader for a reduce task
val reader = sm.getReader[K, V](handle, startPartition, endPartition, context, metrics)
val result = reader.read()
```

### MemoryManager

- Controls the split between execution memory (shuffle, sort, aggregation) and storage memory (cached RDDs, broadcast)
- `UnifiedMemoryManager` (default since Spark 1.6) - single pool, execution and storage share memory and can borrow from each other
- `StaticMemoryManager` (legacy) - fixed split, no borrowing

```scala
val mm = SparkEnv.get.memoryManager

// Acquire execution memory for a task
val granted = mm.acquireExecutionMemory(numBytes, taskAttemptId, MemoryMode.ON_HEAP)

// Acquire storage memory for a block
val success = mm.acquireStorageMemory(blockId, numBytes, MemoryMode.ON_HEAP)

// Release
mm.releaseExecutionMemory(numBytes, taskAttemptId, MemoryMode.ON_HEAP)
mm.releaseStorageMemory(numBytes, MemoryMode.ON_HEAP)

// Inspect
mm.executionMemoryUsed    // bytes currently used for execution
mm.storageMemoryUsed      // bytes currently used for storage
mm.maxOnHeapStorageMemory // total available for storage
```

### OutputCommitCoordinator

- Solves the duplicate output problem - if a task is retried (speculative execution or failure), two task attempts could both try to commit output for the same partition
- `OutputCommitCoordinator` on the Driver grants commit permission to exactly one task attempt per partition; the other is aborted
- Without this, speculative execution would produce duplicate output files

```scala
val occ = SparkEnv.get.outputCommitCoordinator

// Task asks for permission to commit
val canCommit = occ.canCommit(stageId, stageAttemptId, partitionId, taskAttemptId)

// Driver revokes permission when a task is killed
occ.taskCompleted(stageId, stageAttemptId, partitionId, taskAttemptId, TaskKilled)
```

---

## Deploy Modes

- Controls where the Driver process runs
- __Client mode__ -
    - `spark-submit` starts the Driver JVM on the submitting machine
    - Driver registers with Cluster Manager from outside the cluster
    - Cluster Manager launches Executor JVMs on worker nodes
    - All Driver↔Executor RPC traffic (task launch, heartbeat, shuffle metadata, results) crosses the external network between your machine and the cluster

    ```
    // Using spark-submit - client mode is default - can be omitted
    spark-submit --deploy-mode client

    // SparkConf
    .config("spark.submit.deployMode", "client")
    ```

- __Cluster mode__ -
    - `spark-submit` sends the job to the Cluster Manager
    - Cluster Manager allocates a container on a worker node and launches the Driver JVM inside it
    - Executors are launched on nearby worker nodes
    - All Driver↔Executor traffic stays inside the cluster network - low latency, high bandwidth
    - `spark-submit` process exits immediately after submission - it is not the Driver

    ```
    // Using spark-submit
    spark-submit --deploy-mode cluster

    // SparkConf
    .config("spark.submit.deployMode", "cluster")
    ```

- Comparison -
    | Aspect                                      | Client Mode                 | Cluster Mode               |
    | ------------------------------------------- | --------------------------- | -------------------------- |
    | Driver runs on                              | Submitting machine          | Worker node inside cluster |
    | Driver resources tracked by Cluster Manager | No                          | Yes                        |
    | Driver ↔ Executor network                   | External (WAN/VPN)          | Internal cluster network   |
    | Job survives submitting machine death       | No                          | Yes                        |
    | `stdout` / `stderr`                         | Visible in terminal         | In cluster logs only       |
    | Log access after job                        | Local terminal              | `yarn logs`, K8s pod logs  |
    | Use case                                    | Dev, debug, notebooks       | All production jobs        |
    | `spark-submit` process after submission     | Stays alive (is the Driver) | Exits immediately          |
    | Driver memory capped by Cluster Manager     | No                          | Yes                        |

---

## spark-submit

- A shell script (`$SPARK_HOME/bin/spark-submit`) that bootstraps and launches a Spark application
- Not a daemon, not a server - a one-shot process that either becomes the Driver (client mode) or submits the Driver to the cluster (cluster mode) and exits
- Handles dependency resolution, classpath construction, cluster-manager-specific submission protocol, and JVM launch - before a single line of your code runs
- `spark-submit my_job.py --master yarn --deploy-mode cluster` -
    1. `spark-submit` shell script invoked
    2. Finds `JAVA_HOME`, sets up base classpath
    3. Calls spark-class `org.apache.spark.deploy.SparkSubmit` (JVM process starts)
    4. `SparkSubmit` parses all arguments
    5. Builds final classpath - Spark jars + user jars + dependencies
    6. Determines submission path based on `--master` and `--deploy-mode`
    7.1. Client mode  → prepares environment, launches Driver JVM in this process (`exec`)
    7.2. Cluster mode → submits to Cluster Manager, exits after acknowledgement

> [!NOTE]
> `spark-submit` in cluster mode exits before the job finishes - 
>   - The exit code of `spark-submit` only tells you whether submission succeeded, not whether the job succeeded;
>   - Always poll YARN/K8s API or use `--conf spark.yarn.submit.waitAppCompletion=true` for synchronous behavior in pipelines

### SparkSubmit Working

- `org.apache.spark.deploy.SparkSubmit` is the real entry point
    - The shell script is just a thin wrapper that sets up the JVM and calls this class
- __Argument Parsing__ -
    - Parses `--master`, `--deploy-mode`, `--class`, `--jars`, `--files`, `--conf`, `--driver-memory`, `--executor-memory` etc
    - Merges with `spark-defaults.conf` - file-level defaults applied here, before programmatic config
    - Validates required args - throws `SparkException` for missing `--class` on JVM apps, invalid master URLs etc
- __Classpath construction__ -
    - Assembles the full classpath in this order -
        - Spark assembly JAR / individual Spark JARs (`$SPARK_HOME/jars/`)
        - Hadoop configuration directory (if on YARN)
        - `--jars` specified by user
        - `--driver-class-path` extra classpath entries
        - User application JAR
    - For Python - constructs `PYTHONPATH` from `--py-files` and Spark's `pyspark.zip` + `py4j` JAR
- __Submission path selection__ - determined by `--master` + `--deploy-mode` -
    | `--master`    | `--deploy-mode` | Submission path                                       |
    | ------------- | --------------- | ----------------------------------------------------- |
    | `yarn`        | `client`        | `YarnClientSchedulerBackend` - Driver runs locally    |
    | `yarn`        | `cluster`       | `YarnClusterApplication` - submits to YARN RM         |
    | `k8s://...`   | `cluster`       | `KubernetesClientApplication` - submits to K8s API    |
    | `spark://...` | `client`        | `StandaloneSchedulerBackend` - Driver runs locally    |
    | `spark://...` | `cluster`       | `RestSubmissionClient` - submits to Standalone Master |
    | `local[*]`    | `client`        | Runs everything in one JVM, no cluster                |

- __Dependency handling__ -
    ```
    # fat JAR with all dependencies
    target/my-job-assembly-1.0.jar

    # Thin JAR + separate dependencies
    --jars hdfs://path/dep1.jar,hdfs://path/dep2.jar

    # Maven coordinates - Spark downloads and adds to classpath
    --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.0
    --repositories https://repo1.maven.org/maven2

    # Exclude transitive dependencies that conflict
    --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.4.0
    --exclude-packages org.slf4j:slf4j-log4j12

    # Multiple Python files
    --py-files utils.py,helpers.py

    # Zip package
    --py-files my_package.zip

    # Egg file
    --py-files my_package.egg

    # Requirements via conda/venv (Spark 3.1+ with YARN)
    --archives hdfs://path/environment.tar.gz#environment
    --conf spark.pyspark.python=./environment/bin/python
    ```

- __Config files and resources__ -
    ```python
    --files hdfs://path/config.json,hdfs://path/lookup.csv

    # Access in job
    from pyspark import SparkFiles
    config_path = SparkFiles.get("config.json")
    ```

> [!NOTE]
> Fat JAR vs thin JAR - `provided` scope is critical - 
>   - Spark JARs must be marked `provided` in your build
>   - Including them in the fat JAR causes classpath conflicts (`NoSuchMethodError`, `ClassCastException`) between your bundled Spark version and the cluster's Spark version

> [!TIP]
> Set `spark.jars.ivy` to a local Ivy cache to provide jars to `spark-submit`
