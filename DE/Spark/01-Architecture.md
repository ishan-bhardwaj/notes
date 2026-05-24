# Spark Architecture

- Distributed in-memory DAG execution engine
- Processes data in-memory across partitions - spills to disk only under memory pressure
- User writes transformations → Spark builds a DAG → optimizes it → executes across a cluster
- Lazy evaluation - nothing executes until an action is called
- Fault tolerance via lineage recomputation, not data replication

## Components

- __Cluster Manager__ - 
    - Owns cluster resources
    - Allocates containers for Driver and Executors
- __Driver__ - 
    - The brain
    - Owns the entire application lifecycle
- __Executors__ - 
    - The workers
    - Run tasks, store cached data, serve shuffle data
- __SparkSession__ - 
    - User's entry point into Spark
    - Lives on the Driver
- __DAGScheduler__ - 
    - Translates the logical DAG into physical stages
- __TaskScheduler__ - 
    - Assigns tasks from stages onto executor slots
- __BlockManager__ - 
    - Manages storage (memory + disk) on each executor
    - Also one on the Driver
- __Shuffle Service__ - 
    - Optionally runs as a separate daemon per node
    - Serves shuffle files independent of executor lifecycle

## Architecture

### 1. Application Startup

- User calls `spark-submit` (or creates a `SparkSession` in notebook/interactive mode)
- Driver process starts - runs `main()`, creates `SparkSession`, initializes `SparkContext`
- Driver contacts the Cluster Manager and requests executor containers
- Cluster Manager allocates containers on worker nodes and launches Executor JVMs
- Executors register back with the Driver - from this point the Driver knows the full topology

### 2. User Code → Execution Plan

- User writes DataFrame/SQL/RDD transformations - nothing runs yet
- On an action (`count()`, `collect()`, `write()` etc.) Spark triggers execution
- For DataFrame/SQL - code goes through the __Catalyst pipeline__ -
    - Unresolved Logical Plan → Resolved Logical Plan → Optimized Logical Plan → Physical Plan
- For RDD - DAG is built directly from the RDD lineage
- The physical plan is a tree of operators (`scan`, `filter`, `join`, `aggregate`, `exchange`/`shuffle`)

### 3. DAG → Stages

- `DAGScheduler` takes the physical plan / RDD DAG and slices it into __Stages__
- Stage boundary is cut at every __wide dependency__ (shuffle) - `groupBy`, `join`, `repartition`, `distinct` etc.
- Everything within a stage is a pipeline of __narrow transformations__ - no network, runs end-to-end on one partition
- `DAGScheduler` also computes __preferred locations__ for each task (data locality - HDFS block locations, cached partition locations)

### 4. Stages → Tasks

- Each stage is divided into __Tasks__ - one task per partition
- `DAGScheduler` submits a __TaskSet__ (all tasks for a stage) to the `TaskScheduler`
- `TaskScheduler` assigns tasks to available executor slots using __delay scheduling__ - waits briefly for a data-local slot before falling back to rack-local, then any available slot
- Tasks are serialized (along with the closure - all variables the task references) and sent to executors

### 5. Task Execution on Executors

- Each executor runs tasks as __threads__ within the JVM (not separate processes)
- Each executor has `spark.executor.cores` task slots - that many tasks run concurrently per executor
- A task processes exactly __one partition__ - reads it, applies the chain of narrow operators, produces output
- For shuffle stages - task writes output to local disk (shuffle write)
- For final stages - task either writes to external storage or sends result back to Driver

### 6. Shuffle

- Between stages, data must be redistributed across the network
- Map-side tasks write shuffle output to local disk - one sorted file per mapper with a partition index
- Reduce-side tasks fetch their partition's data from all map-side nodes via HTTP
- `BlockManager` on each executor tracks and serves these shuffle blocks
- If `External Shuffle Service` is running - shuffle files are served by the daemon, not the executor; this lets the executor die (for dynamic allocation) without losing its shuffle output

### 7. Result Back to Driver

- For actions that return data (`collect()`, `take()`, `first()`) - executor tasks send results back to Driver over the network
- Driver assembles the final result and returns it to user code
- For write actions (`df.write`) - executors write directly to the sink; Driver just coordinates and waits for all tasks to confirm completion

## Deploy Modes

- __Client mode__ - Driver runs on the submitting machine
    - Stdout/logs directly visible; good for interactive/debugging
    - If the machine dies or disconnects, the application dies
    - Network between Driver and executors goes over external network - high latency for heavy jobs like `collect()`

- __Cluster mode__ - Driver runs as a container on a worker node inside the cluster
    - Production standard - application lifecycle is fully within the cluster
    - Driver is co-located with executors - lower latency, more stable
    - Logs go to cluster log aggregation (YARN logs, K8s pod logs)

## Partition

- Every RDD/DataFrame is divided into partitions
- One task processes one partition - partition count directly determines parallelism
- Input partitions - determined by source (file splits, JDBC ranges, Kafka topic partitions)
- Shuffle output partitions - controlled by `spark.sql.shuffle.partitions` (default $200$, almost always wrong)
- Too few partitions → tasks are too large, executors OOM, no parallelism
- Too many partitions → thousands of tiny tasks, scheduler overhead dominates, many small shuffle files
- Target: $~128MB$ per partition - partition count should be a multiple of total executor cores

## Fault Tolerance

- __Task failure__ - 
    - Retried up to `spark.task.maxFailures` (default $4$) times, on a different executor if possible
- __Executor failure__ - 
    - Driver detects via missed heartbeat - tasks on that executor are rescheduled
    - If shuffle output was on that executor and no External Shuffle Service, parent stage is rerun
- __Stage failure__ - 
    - If all task retries exhausted - `DAGScheduler` resubmits the stage
    - Parent stages rerun if their shuffle data is lost
- __Driver failure__ - 
    - Application is lost
    - No automatic recovery in batch mode
    - Structured Streaming recovers from checkpoint on restart
- Lineage is the recomputation backbone - 
    - Spark knows how to rebuild any partition from its source by replaying the DAG

## Dynamic Allocation

- Spark can scale executor count up and down based on backlog
- Idle executors are released back to the cluster manager after `spark.dynamicAllocation.executorIdleTimeout`
- New executors are requested when tasks are waiting in queue
- Requires External Shuffle Service (or shuffle tracking on K8s) - otherwise releasing an executor loses its shuffle data
- Controlled by `spark.dynamicAllocation.enabled`, `spark.dynamicAllocation.minExecutors`, `spark.dynamicAllocation.maxExecutors`