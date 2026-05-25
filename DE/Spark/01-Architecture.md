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
