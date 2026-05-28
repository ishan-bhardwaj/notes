# Spark Architecture

- **Distributed in-memory DAG execution engine**
- **Lazy evaluation** — transformations build DAG, actions trigger execution
- **Fault tolerance** — lineage recomputation, not replication
- In-memory execution with disk spill under pressure

## Core Components

| Component                | Responsibility                                                                  |
| ------------------------ | ------------------------------------------------------------------------------- |
| Driver                   | Owns `SparkSession`, Coordination, scheduling, DAG construction                 |
| Executors                | Task execution, cache data, serve shuffle blocks                                |
| Cluster Manager          | Resource allocation                                                             |
| DAGScheduler             | Splits DAG into stages at shuffle boundaries                                    |
| TaskScheduler            | Asigns tasks to executor slots via delay scheduling                             |
| BlockManager             | Manages cached / shuffle / broadcast blocks per executor                        |
| External Shuffle Service | Serves shuffle files from node daemon; prerequisite for safe dynamic allocation |

## Execution Flow

### 1. Startup

- `spark-submit` launches Driver
- Driver creates `SparkSession` / `SparkContext`
- Driver registers with Cluster Manager
- Cluster Manager launches Executor JVMs on worker nodes
- Executors heartbeat to Driver

### 2. Lazy DAG construction

- Every transformation adds a node to the DAG - no execution until action
- DataFrame/SQL path - Action → Catalyst → Physical plan → RDD lineage
- RDD API bypasses Catalyst

### 3. DAG → Stages

- DAGScheduler cuts stages at shuffle boundaries -
    - Narrow dependency (`filter`, `select`, `union`) _pipeline_ within a stage
    - Wide dependency (`groupBy`, `join`, `repartition`) require _shuffle_
- DAGScheduler also tracks locality preferences

### 4. Stages → Tasks

- Each stage produces a `TaskSet` - one task per partition
- TaskScheduler handles locality-aware scheduling
- Tasks serialize closure + metadata to executors

> [!WARNING]
> Large closures increase serialization overhead and executor memory pressure

### 5. Task Execution

- Executor = JVM process
- Tasks = threads sharing same heap
- Shuffle output written to disk locally
- Final output written to sink or returned to Driver

> [!NOTE]
> GC pauses affect all tasks within executor

### 6. Shuffle

- Map tasks write shuffle files locally
- Reduce tasks fetch remote blocks over network
- BlockManager serves shuffle blocks
- External Shuffle Service -
    - Shuffle files are served by the node daemon, not the executor JVM
    - Shuffle files survive executor death
    - Enables safe dynamic allocation

### 7. Result Handling

- `collect()` - data returned to driver
- `write()` - executors write directly to storage

> [!WARNING]
> Large `collect()` can crash Driver with OOM

## Fault Tolerance

- Spark recovers partitions via lineage recomputation
- Task failures - retry
- Executor failure - partition recomputation
- Driver failure usually kills application

## SparkConf

- Immutable application configuration key-value store
- Finalized during `SparkContext` creation
- Driver distributes relevant configs to executors
    - Driver passes a subset of config to executors via the Cluster Manager
    - Task-level config travels inside `TaskDescription` serialized with each task
- Config priority (highest to lowest) -
    - Programmatic - `new SparkConf().set("spark.executor.memory", "4g")`
    - `spark-submit` flags - `--executor-memory 4g` (these set the same keys programmatically under the hood)
    - `spark-defaults.conf` - file on the Driver's classpath (`$SPARK_HOME/conf/spark-defaults.conf`)
    - Spark's hardcoded defaults - defined in Spark source code

> [!NOTE]
> `spark-defaults.conf` is read only by `spark-submit` on the _submitting_ machine.
>
> In both client and cluster mode -
>   - `spark-submit` loads configs locally
>   - Builds the final Spark configuration
>   - Sends that configuration to the Driver/cluster

| Mutable at Runtime             | Immutable After Startup |
| ------------------------------ | ----------------------- |
| SQL configs                    | Executor/JVM configs    |
| `spark.sql.shuffle.partitions` | `spark.executor.memory` |

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

> [!TIP]
> `spark.conf.isModifiable("<config-key>")` returns `true` if the config is modifiable at runtime

## SparkContext

- Spark core engine entry point
- Lives on Driver
- One per application/JVM
- Initializes and holds references to every core internal component like `DAGScheduler`, `TaskScheduler`, `BlockManager`, `SparkEnv`, `MapOutputTracker`, `BroadcastManager` etc
- All communication with the Cluster Manager flows through it
- Starts heartbeat thread to executors
- Creating `SparkContext` - 
    ```python
    # Modern - access SparkContext from session
    sc = spark.sparkContext                    

    # Direct - only for pure RDD or testing
    sc = SparkContext(conf=sparkConf)
    ```

## SparkSession

- Unified Spark entry point (Spark 2+)
- Wraps single `SparkContext`- multiple `SparkSession` instances share the same `SparkContext`
- `SparkSession` owns -

    | Component       | Role                                                                                 |
    | --------------- | ------------------------------------------------------------------------------------ |
    | `SparkContext`  | Core engine - cluster connection, scheduling, RDDs                                   |
    | `SessionState`  | Per-session state - catalog, analyzer, optimizer, planner, function registry         |
    | `SharedState`   | Cross-session shared state - `SparkContext`, Hive metastore connection, shared cache |
    | `RuntimeConfig` | Thin wrapper over `SparkConf` + session-level SQL configs                            |
    | `Catalog`       | User-facing API to databases, tables, functions, caches                              |

- Creating `SparkSession` -
    ```python
    from pyspark.sql import SparkSession

    spark = SparkSession.builder \
        .appName("MyApp") \
        .master("yarn") \
        .config("spark.executor.memory", "4g") \
        .getOrCreate()                              # returns existing session if one exists
    ```

> [!TIP]
> `SparkSession#newSession` creates new session with isolated `SessionState`, but same `SharedState`
 
- Stopping `SparkSession` - `spark.stop()`- releases all cluster resources

## SparkEnv

- Internal runtime container created on driver and each executor
- Components -
    | Component        | Responsibility           |
    | ---------------- | ------------------------ |
    | RpcEnv           | RPC communication        |
    | BlockManager     | Cache/shuffle storage    |
    | ShuffleManager   | Shuffle handling         |
    | MapOutputTracker | Shuffle metadata         |
    | BroadcastManager | Broadcast variables      |
    | MemoryManager    | Execution/storage memory |

### Accessing SparkEnv

- Python API -
    - Python tasks run in a separate Python process (not the Executor JVM where `SparkEnv` resides)
    - `py4j` bridge only works on the Driver side
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
    ```

## SparkEnv Components

### SerializerManager

- Sits above the raw serializer (`KryoSerializer` / `JavaSerializer`) and adds compression, encryption, and per-block decisions about whether to compress
- Decides whether to compress a block based on its type -
    ```python
    spark.shuffle.compress=true             # shuffle data is compressed
    spark.broadcast.compress=true           # broadcast data is compressed
    spark.rdd.compress=true                 # RDD partitions are compressed
    spark.shuffle.spill.compress=true       # spill data is compressed
    ```

### RpcEnv

- Netty-based message-passing layer - every Driver ↔ Executor interaction is an RPC call through this
- `RpcEndpoint` - 
    - Named actor-like object that handles messages
    - `DAGScheduler`, `BlockManagerMaster`, `MapOutputTrackerMaster` are all `RpcEndpoint`s registered in the Driver's `RpcEnv`
- `RpcEndpointRef` - 
    - Serializable reference to an endpoint
    - Executors hold refs to Driver endpoints

### BlockManager

- Every block of data in Spark - RDD partition, shuffle file chunk, broadcast piece - has a `BlockId` and is managed by `BlockManager`
- On each executor, `BlockManager` decides whether a block lives in memory (`MemoryStore`) or on disk (`DiskStore`)
- `BlockManagerMaster` on the Driver maintains a registry of all blocks across all executors - knows which executor holds which block

### MapOutputTracker

- After a shuffle map stage completes, each map task registers its output location with `MapOutputTrackerMaster` on the Driver
- Reduce tasks query `MapOutputTrackerWorker` to fetch locations from Master if not cached locally
- This is how reduce tasks know which executor to fetch each shuffle partition from

### ShuffleManager

- ShuffleManager handles shuffle write/read operations - default implementation is `SortShuffleManager`
- `SortShuffleWriter` sorts records in memory, spills if needed, then merges final shuffle files

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

## Deploy Modes

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

## spark-submit

- A shell script (`$SPARK_HOME/bin/spark-submit`) - bootstraps and launches a Spark application
- Internally calls `org.apache.spark.deploy.SparkSubmit`
- Determines submission path based on `--master` and `--deploy-mode`
    - Client mode  - prepares environment, launches Driver JVM in this process (`exec`)
    - Cluster mode - submits to Cluster Manager, exits after acknowledgement
- Responsibilities -
    - Builds classpath
    - Parse configs
    - Dependency shipping
    - JVM startup
    - Cluster submission

> [!NOTE]
> Fat JAR vs thin JAR -
>   - Spark JARs must be marked `provided` in your build
>   - Including them in the fat JAR causes classpath conflicts (`NoSuchMethodError`, `ClassCastException`) between your bundled Spark version and the cluster's Spark version

> [!TIP]
> Set `spark.jars.ivy` to a local Ivy cache to provide jars to `spark-submit`
