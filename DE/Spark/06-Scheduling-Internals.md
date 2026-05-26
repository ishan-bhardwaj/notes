# Scheduling Internals

## MapOutputTracker

- __`MapOutputTracker`__ - tracks the output locations of shuffle map tasks; the registry that tells reduce tasks where to fetch their input data
- Without `MapOutputTracker`, reduce tasks have no way to know which executor holds which partition of shuffle output
- Two roles - master (Driver) and worker (Executor); same abstract class, different implementations
- Lives in `SparkEnv` - one instance per JVM process

### What MapOutputTracker Tracks

- For each shuffle, tracks an array of `MapStatus` - one per map task -
    - `MapStatus.location` - `BlockManagerId` of the executor that ran the map task
    - `MapStatus.getSizeForBlock(reduceId)` - compressed size estimate of output for partition `reduceId`
- Size estimates used by `ShuffleBlockFetcherIterator` to schedule fetches and by `DAGScheduler` to detect skew

### MapStatus Compression

- `MapStatus` stores partition sizes as compressed bytes to reduce Driver memory -
    - `CompressedMapStatus` - uses log-scale 8-bit compression via `MapStatus.compressSize(size)`
    - Sizes are approximate after compression - acceptable since they are only used for fetch scheduling, not correctness
    - Example - $10000$ map tasks × $1000$ partitions × $8$ bytes = $80 MB$ uncompressed; log-scale compression reduces this dramatically
- `HighlyCompressedMapStatus` - used when a map task has many output partitions ($> 2000$) -
    - Stores only the average size and a bitmap of non-empty partitions
    - Further reduces Driver memory at cost of less accurate size estimates

### Epoch Mechanism

- `epoch` counter incremented whenever map outputs are lost (executor failure)
- Each task carries the current epoch
- If executor's cached epoch is stale → re-fetches map statuses from Driver
- Prevents reduce tasks from using stale location data after executor failures

---

## MapOutputTrackerMaster

- __`MapOutputTrackerMaster`__ - Driver-side implementation; the authoritative registry for all shuffle map outputs
- Runs as `MapOutputTrackerMasterEndpoint` RPC endpoint on Driver
- Holds `shuffleStatuses: Map[Int, ShuffleStatus]` - maps `shuffleId` to `ShuffleStatus`
- `ShuffleStatus` holds `mapStatuses: Array[MapStatus]` - one slot per map task; `null` until task completes

### Registration and Updates

- `registerShuffle(shuffleId, numMaps)` - called by `DAGScheduler` when a new `ShuffleMapStage` is created; initializes `mapStatuses` array of size `numMaps`
- `registerMapOutput(shuffleId, mapId, status)` - called when a map task completes; stores `MapStatus` at `mapStatuses[mapId]`
- `unregisterMapOutput(shuffleId, mapId, bmAddress)` - called when executor dies; nulls out map outputs for that executor
- `unregisterShuffle(shuffleId)` - called after downstream stages complete; removes entire shuffle from registry

### Serving Reduce Tasks

- `MapOutputTrackerMasterEndpoint` receives `GetMapOutputStatuses(shuffleId)` from executor workers -
    - Serializes `mapStatuses` array for the shuffle
    - Caches serialized result (`cachedSerializedStatuses`) to avoid re-serializing on every request
    - Cache invalidated when new map outputs registered for the shuffle
- Serialized `mapStatuses` array sent once per executor per shuffle - workers cache locally

### Thread Pool for Serialization

- Serializing large `mapStatuses` arrays is CPU-intensive
- `MapOutputTrackerMaster` uses a dedicated thread pool (`spark.shuffle.mapOutput.dispatcher.numThreads`, default $8$) for serialization
- Prevents serialization from blocking the main RPC thread

---

## MapOutputTrackerWorker

- __`MapOutputTrackerWorker`__ - executor-side implementation; caches map output locations locally to avoid repeated RPC calls to Driver
- Holds `mapStatuses: Map[Int, Array[MapStatus]]` - local cache of fetched shuffle statuses

### Fetch and Cache

- `getMapSizesByExecutorId(shuffleId, startPartition, endPartition)` -
    1. Check local `mapStatuses` cache for `shuffleId`
    2. If miss - send `GetMapOutputStatuses(shuffleId)` to `MapOutputTrackerMasterEndpoint` on Driver
    3. Receive and deserialize full `mapStatuses` array; store in local cache
    4. Filter to requested partition range; group by `BlockManagerId`
    5. Return `Seq[(BlockManagerId, Seq[(BlockId, Long, Int)])]`
- Cache is per-shuffle - once fetched, all reduce tasks on the same executor reuse cached statuses
- Cache cleared on epoch change - stale data detected and purged before re-fetch

---

## DAGScheduler

- __`DAGScheduler`__ - translates the logical RDD DAG into a physical execution plan of `Stage` objects; orchestrates stage submission, dependency tracking, and fault recovery
- Operates entirely on the Driver
- Receives the final RDD when an action is called; walks lineage backwards recursively to discover the full DAG
- Thinks in terms of stages and data dependencies - has no knowledge of executor slots or resources

### Core Responsibilities

- __Stage construction__ - walks RDD lineage; cuts stage boundaries at every `ShuffleDependency`
- __Stage submission__ - submits stages only when all parent stages are complete
- __Preferred location computation__ - queries `BlockManagerMaster` and HDFS block locations to compute task locality hints
- __Fault recovery__ - handles executor loss, fetch failures, and stage resubmission
- __MapOutputTracker coordination__ - registers shuffles, records map outputs, invalidates lost outputs

### DAG Walking Algorithm

1. Action called on RDD → `SparkContext.runJob()` → `DAGScheduler.runJob()`
2. `DAGScheduler` calls `getOrCreateResultStage(finalRDD)` - builds stages backwards
3. Creates `ResultStage` for the final RDD
4. Walks each RDD's dependencies recursively -
    - Narrow dependency → same stage, keep walking back
    - `ShuffleDependency` → create new `ShuffleMapStage` for parent; record as stage dependency
5. Recursion terminates at source RDDs (file scans, `parallelize`) or checkpointed RDDs (lineage truncated)
6. Result - a DAG of stages where edges represent shuffle dependencies

---

## Job & Stage Lifecycle

### Job Lifecycle

- `SparkContext.runJob()` called by action
- `DAGScheduler.runJob()` creates a `JobWaiter` and posts `JobSubmitted` event to event loop
- `handleJobSubmitted` -
    - Creates `ResultStage` and all parent `ShuffleMapStage`s recursively
    - Creates `ActiveJob` wrapping the `ResultStage`
    - Adds job to `jobIdToActiveJob` and `resultStageToJob` maps
    - Calls `submitStage(resultStage)`
- `submitStage` checks all parent stages -
    - If parents not complete → add to `waitingStages`
    - If parents complete → `submitMissingTasks(stage)`
- `submitMissingTasks` computes preferred locations, creates `TaskSet`, submits to `TaskScheduler`
- As tasks complete, `DAGScheduler` receives `CompletionEvent`s
- When all tasks in `ResultStage` complete → `jobWaiter.taskSucceeded()` for each → job complete
- `cleanupStateForJobAndIndependentStages` - removes job and any stages not shared with other jobs

### Stage Lifecycle

- __Created__ - `DAGScheduler` instantiates `Stage` object during DAG construction
- __Waiting__ - in `waitingStages`; parent stages not yet complete
- __Running__ - in `runningStages`; tasks submitted to `TaskScheduler`
- __Available__ (ShuffleMapStage only) - all map outputs registered; can serve reduce tasks
- __Failed__ - `TaskSet` exhausted retries; stage aborted

---

## ResultStage

- __`ResultStage`__ - the final stage in a job; directly computes the result of an action
- Exactly one `ResultStage` per job
- Tasks are `ResultTask` instances - run the RDD compute chain and either return data to Driver (`collect`) or write to external storage (`write`)
- Holds reference to the `ActiveJob` and the `func: (TaskContext, Iterator[_]) => _` to apply to each partition
- Partition set - can be a subset of all partitions (eg - `take(10)` only runs tasks for partitions until enough results collected)

---

## ShuffleMapStage

- __`ShuffleMapStage`__ - any stage before the final stage; tasks write shuffle output to local disk for downstream stages
- Tasks are `ShuffleMapTask` instances - run RDD compute chain and write output via `ShuffleWriter`
- Can be __shared across multiple jobs__ -
    - If two jobs need the same shuffle output and it is still available, `DAGScheduler` reuses it
    - Tracked via `shuffleIdToMapStage: Map[Int, ShuffleMapStage]`
- Holds reference to `ShuffleDependency` and `MapOutputTrackerMaster`
- `isAvailable` - true when all map outputs are registered; downstream stages can proceed

> [!NOTE]
> A job with $N$ shuffles produces $N$ `ShuffleMapStage`s + $1$ `ResultStage`. Stages at the same dependency level with no dependency between them can run concurrently - `DAGScheduler` submits them all at once; `TaskScheduler` interleaves their tasks across available slots.

---

## Stage Reuse

- `DAGScheduler` maintains `shuffleIdToMapStage: HashMap[Int, ShuffleMapStage]` keyed by shuffle ID
- Before creating a new `ShuffleMapStage`, checks if one already exists for that shuffle
- If a previous job already computed and all map outputs are still available → stage reused, not rerun
- Critical for iterative algorithms (ML training loops) - base data shuffle not recomputed on each iteration
- Reuse check - `mapOutputTracker.containsShuffle(shuffleId)` AND all map outputs registered
- If even one map output is missing (executor died) → stage must rerun missing tasks

---

## DAGScheduler Event Loop

- `DAGScheduler` is fully event-driven - runs `DAGSchedulerEventProcessLoop` on a dedicated thread
- All external interactions post events to this loop - nothing called directly from other threads
- Single-threaded event processing → internal state consistent without locks

### Key Events

- `JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)` - action triggered; new job to process
- `StageCancelled(stageId)` / `JobCancelled(jobId)` - cancellation from user or timeout
- `CompletionEvent(task, reason, result, accumUpdates, metricPeaks, taskInfo)` - task finished (success or failure); posted by `TaskScheduler`
- `ExecutorAdded(execId, host)` - new executor registered; check if any waiting stages can now be submitted
- `ExecutorLost(execId, reason)` - executor died; invalidate map outputs; resubmit affected stages
- `TaskSetFailed(taskSet, reason, exception)` - all retries for a `TaskSet` exhausted; abort stage
- `ResubmitFailedStages` - retry failed stages after a delay (posted after `FetchFailedException`)
- `JobGroupCancelled(groupId)` - cancel all jobs in a group
- `AllJobsCancelled` - cancel everything

---

## DAGScheduler Fault Tolerance

### Task Failure

- `TaskScheduler` handles individual task retries (up to `spark.task.maxFailures`, default $4$)
- `FetchFailedException` bypasses normal retry path - routed directly to `DAGScheduler` -
    - Signals shuffle data loss, not task-level failure
    - `DAGScheduler` marks lost map outputs in `MapOutputTracker`
    - Posts `ResubmitFailedStages` event after a delay (`spark.scheduler.blacklist.timeout`)
    - Resubmits affected `ShuffleMapStage` to regenerate lost data
    - Then resubmits the blocked downstream stage

### Stage Failure

- `TaskSetFailed` event → `DAGScheduler.handleTaskSetFailed()` → marks stage failed
- Triggers `abortStage` - the stage and all jobs depending on it are aborted
- `spark.stage.maxConsecutiveAttempts` (default $4$) - stage that fails this many times consecutively causes job abort

### Executor Loss

- `ExecutorLost` event posted to event loop
- `DAGScheduler` calls `mapOutputTracker.removeOutputsOnExecutor(execId)` -
    - Nulls out all `MapStatus` entries for that executor
- Checks all running stages - any tasks fetching from lost executor will surface `FetchFailedException`
- Affected `ShuffleMapStage`s resubmitted

### Checkpointing Interaction

- Checkpointed RDD - lineage truncated; `DAGScheduler` sees it as source (no parents)
- Breaks long lineage chains - failure late in $100$-step lineage recomputes only from last checkpoint
- `DAGScheduler` calls `rdd.doCheckpoint()` after each job completes for RDDs marked for checkpointing
- For Structured Streaming - checkpoint location stores committed batch IDs and operator state; lineage recomputation model enables exactly-once recovery

### Data Locality Computation

- `DAGScheduler` computes preferred locations for every task before submitting to `TaskScheduler` -
    - __Cached RDDs__ - queries `BlockManagerMaster` for executor locations of cached partitions → `PROCESS_LOCAL`
    - __HDFS/S3 inputs__ - `FileInputFormat.getSplits` returns HDFS block replica locations → `NODE_LOCAL`
    - __Shuffle inputs__ - no meaningful preferred location; data comes from all map outputs → `ANY`
- Preferred locations passed to `TaskScheduler`; delay scheduling implemented there

### Key DAGScheduler Configs

| Configuration | Default | What It Controls |
| --- | --- | --- |
| `spark.locality.wait` | `3s` | Max wait for preferred locality before fallback |
| `spark.locality.wait.process` | `spark.locality.wait` | Wait for `PROCESS_LOCAL` |
| `spark.locality.wait.node` | `spark.locality.wait` | Wait for `NODE_LOCAL` |
| `spark.locality.wait.rack` | `spark.locality.wait` | Wait for `RACK_LOCAL` |
| `spark.stage.maxConsecutiveAttempts` | `4` | Max consecutive stage retries before job abort |
| `spark.sql.shuffle.partitions` | `200` | Default shuffle output partition count |

---

## TaskScheduler

- __`TaskScheduler`__ - receives `TaskSet`s from `DAGScheduler` and gets them executed on executors
- Sits between `DAGScheduler` and the cluster -
    - `DAGScheduler` thinks in stages and preferred locations
    - `TaskScheduler` thinks in executor slots, resource offers, and task lifecycle
- Operates entirely on the Driver
- `TaskScheduler` is an interface - real implementation is `TaskSchedulerImpl`
- `SchedulerBackend` is a separate interface below it - abstracts the cluster manager

### Core Responsibilities

- Matching tasks to executor slots respecting locality preferences
- Handling task retries and executor exclusion/blacklisting
- Speculative execution for straggler tasks
- Reporting task completion/failure back to `DAGScheduler`

---

## TaskSchedulerImpl

- __`TaskSchedulerImpl`__ - the single concrete implementation of `TaskScheduler`
- Owns -
    - `taskSetsByStageIdAndAttempt: Map[Int, Map[Int, TaskSetManager]]` - all active `TaskSetManager`s
    - `taskIdToTaskSetManager` - maps running task IDs to their `TaskSetManager`
    - `taskIdToExecutorId` - maps running task IDs to executor
    - `executorIdToRunningTaskIds` - maps executor to its running tasks
    - `schedulableBuilder` - either `FIFOSchedulableBuilder` or `FairSchedulableBuilder`
    - `rootPool` - the root scheduling pool; contains all `TaskSetManager`s

### Initialization

- `TaskSchedulerImpl.initialize(backend)` - called with the `SchedulerBackend`; sets up scheduling pool
- `TaskSchedulerImpl.start()` - starts `SchedulerBackend`; starts speculation timer if enabled

### resourceOffers - The Core Scheduling Method

- Called by `SchedulerBackend` when executor slots become available
- `resourceOffers(offers: Seq[WorkerOffer])` - each `WorkerOffer` is `(executorId, host, freeCores, resources)`
- Flow -
    1. Update executor metadata - new executors registered; excluded executors filtered out
    2. Shuffle offers (randomize order to avoid hotspotting same executors)
    3. For each locality level (PROCESS_LOCAL → NODE_LOCAL → RACK_LOCAL → NO_PREF → ANY) -
        - Iterate `TaskSetManager`s in scheduling order (FIFO or Fair)
        - For each `TaskSetManager`, call `resourceOffer(execId, host, localityLevel)` per offer
        - Collect assigned `TaskDescription`s
    4. Return list of `TaskDescription`s - one per slot that got a task
- Offer-based model → `TaskScheduler` is reactive, not proactive

---

## TaskSetManager

- __`TaskSetManager`__ - one per active `TaskSet`; tracks every task in a stage; handles retries, locality, and speculation
- Where most real scheduling logic lives
- Registered in the root scheduling pool as a `Schedulable`

### Pending Task Queues

- `TaskSetManager` maintains separate queues per locality level -
    - `pendingTasksForExecutor: HashMap[String, ArrayBuffer[Int]]` - tasks preferring a specific executor (`PROCESS_LOCAL`)
    - `pendingTasksForHost: HashMap[String, ArrayBuffer[Int]]` - tasks preferring a specific host (`NODE_LOCAL`)
    - `pendingTasksForRack: HashMap[String, ArrayBuffer[Int]]` - tasks preferring a specific rack (`RACK_LOCAL`)
    - `pendingTasksWithNoPrefs: ArrayBuffer[Int]` - tasks with no locality preference
    - `allPendingTasks: ArrayBuffer[Int]` - all remaining tasks regardless of preference
- When resource offer arrives for executor `E` on host `H` in rack `R` -
    - Tries queues in order - `PROCESS_LOCAL` → `NODE_LOCAL` → `RACK_LOCAL` → `NO_PREF` → `ANY`
    - Task removed from all queues once assigned

### Task Retry Tracking

- `taskAttempts: Array[List[TaskInfo]]` - history of all attempts per task
- `numFailures: Array[Int]` - failure count per task
- `tasksSuccessful: Int` - count of successfully completed tasks
- On failure - increments `numFailures[taskIndex]`; if < `spark.task.maxFailures` → re-enqueue; else → `abort()`

---

## Pending Task Queues

- Covered in `TaskSetManager` above
- Key insight - queues are indexed by executor/host/rack ID; a task can appear in multiple queues simultaneously (eg - a task appears in `pendingTasksForExecutor["exec1"]`, `pendingTasksForHost["host1"]`, and `allPendingTasks`)
- When a task is assigned, it is NOT removed from other queues immediately - lazy cleanup on dequeue
- "Deduplication" on assignment - `isTaskBlacklistedOnExecOrNode` check and `taskInfos` check prevent double-assignment

---

## SchedulerBackend

- __`SchedulerBackend`__ - interface between `TaskScheduler` and the cluster manager; abstracts resource acquisition and task launching
- Decouples `TaskScheduler` from cluster-manager-specific APIs

### Interface Methods

- `start()` - connect to cluster manager; begin requesting resources
- `stop()` - disconnect; release all resources
- `reviveOffers()` - signal that new resource offers may be available; triggers `resourceOffers` call
- `killTask(taskId, execId, interruptThread, reason)` - tell executor to kill a running task
- `defaultParallelism()` - returns default parallelism (total executor cores)
- `isReady()` - returns true when enough executors registered to begin scheduling

### CoarseGrainedSchedulerBackend

- Base class for all production backends (Standalone, YARN, K8s)
- Maintains persistent executor connections - executors are long-lived JVMs, not one-per-task
- `DriverEndpoint` - RPC endpoint on Driver that executors communicate with -
    - `RegisterExecutor` - executor startup; stores executor metadata
    - `StatusUpdate` - task completion/failure notification
    - `ReviveOffers` - executor signals it has free slots
- `launchTasks(tasks)` - serializes `TaskDescription`s and sends `LaunchTask` RPC to each executor

---

## StandaloneSchedulerBackend

- __`StandaloneSchedulerBackend`__ - extends `CoarseGrainedSchedulerBackend`; connects to Spark Standalone cluster manager
- Standalone master assigns workers; `StandaloneSchedulerBackend` negotiates with master for executor containers
- `StandaloneAppClient` - communicates with Standalone master via RPC -
    - `RegisterApplication` - registers app with master on startup
    - Master responds with `RegisteredApplication` and begins allocating workers
    - Workers launch executor processes; executors register with Driver's `DriverEndpoint`
- `executorAdded` / `executorRemoved` callbacks - Standalone master notifies backend of executor lifecycle
- Dynamic allocation supported via `ExecutorAllocationManager` + `StandaloneSchedulerBackend.requestExecutors` / `killExecutors`

---

## YARNSchedulerBackend

- __`YARNSchedulerBackend`__ - extends `CoarseGrainedSchedulerBackend`; two sub-implementations -
    - `YarnClientSchedulerBackend` - client deploy mode; Driver runs on submitting machine
    - `YarnClusterSchedulerBackend` - cluster deploy mode; Driver runs inside YARN container
- `YarnAllocator` - manages container requests to YARN ResourceManager -
    - Tracks pending, running, and completed containers
    - Sends `ResourceRequest`s to RM; receives allocated containers
    - Launches executor JVMs in allocated containers via YARN NM
- Locality-aware container requests -
    - `YarnAllocator` passes preferred node/rack hints to YARN RM with container requests
    - RM attempts to satisfy locality; falls back if unavailable after timeout
- `spark.yarn.am.waitTime` - time to wait for initial executors before timing out
- `spark.yarn.executor.memoryOverhead` - off-heap memory overhead per executor container (default $10%$ of executor memory or $384 MB$ minimum)

---

## KubernetesSchedulerBackend

- __`KubernetesSchedulerBackend`__ - extends `CoarseGrainedSchedulerBackend`; manages executor pods on Kubernetes
- `ExecutorPodsAllocator` - creates and deletes executor pods via K8s API -
    - Watches pod state changes via K8s watch API
    - Requests new pods when `TaskScheduler` signals pending tasks
    - Deletes idle pods when dynamic allocation signals scale-down
- `ExecutorPodsLifecycleManager` - reconciles pod state with executor state -
    - Pod `Failed`/`Succeeded` → notify `TaskScheduler` of executor loss
    - Pod stuck in `Pending` too long → delete and re-request
- Executor pod spec - configurable via `spark.kubernetes.executor.podTemplateFile`; supports resource limits, node selectors, tolerations, volumes
- __Client mode__ - Driver runs outside K8s cluster; executors in pods connect back to Driver IP
- __Cluster mode__ - Driver runs in a pod; Spark creates a `SparkDriver` pod; executors connect to Driver pod's internal K8s DNS name
- Dynamic allocation on K8s - uses Shuffle Tracking (no `ExternalShuffleService` by default); see Shuffle Tracking section

---

## Resource Offers Mechanism

- `SchedulerBackend` → `TaskScheduler.resourceOffers(offers)` is the central scheduling trigger
- Every time an executor finishes a task or a new executor registers, `reviveOffers()` is called → triggers `resourceOffers`

### Offer Processing Detail

1. Filter out excluded executors and nodes from offers
2. Update `hostToExecutors` and rack mappings for new executors
3. Shuffle offer list to prevent repeatedly offering same executors (distributes tasks evenly)
4. For each locality level from best to worst -
    - For each `TaskSetManager` in scheduling order (FIFO/Fair) -
        - Loop through offers; call `taskSetManager.resourceOffer(execId, host, localityLevel)`
        - If task assigned - add `TaskDescription` to results; mark offer slot used
        - Continue until no more tasks assigned at this locality level
    - Move to next locality level only after full pass yields no assignments
5. `launchTasks(taskDescriptions)` - serialize and send to executors

### Why Offers Are Shuffled

- Without shuffling, first executor in list always gets tasks first → some executors overloaded, others idle
- Shuffling distributes tasks across executors evenly in expectation
- Combined with delay scheduling → achieves both locality and load balance

---

## Delay Scheduling

- When resource offer arrives and best available task is at worse locality than desired, `TaskScheduler` waits for a better slot
- Implemented via last-launch timestamp per locality level in `TaskSetManager` -
    - If time since last task launched at current level < `spark.locality.wait` → skip offer, wait for better slot
    - If wait expired → fall back to next locality level

### Locality Level Progression

- `PROCESS_LOCAL` → `NODE_LOCAL` → `RACK_LOCAL` → `NO_PREF` → `ANY`
- Each level has its own wait (`spark.locality.wait.process`, `.node`, `.rack`)
- Fallback is permanent within a `resourceOffers` call - once fallen back, does not re-try better locality for that pass

### When Delay Scheduling Helps and Hurts

- __Helps__ - source stage reading HDFS data; task running on node with data avoids network read entirely
- __Hurts__ - shuffle read stage; data comes from network regardless; delay scheduling wastes time waiting for locality that provides no benefit
    - Mitigation - set `spark.locality.wait=0` for shuffle-heavy jobs or use `ANY` directly

---

## Task Serialization & Closure Cleaning

- When `resourceOffers` assigns a task to a slot, `TaskScheduler` creates a `TaskDescription`
- Task object (RDD compute function + closure) serialized using configured serializer (`spark.serializer`)
- Always use Kryo (`org.apache.spark.serializer.KryoSerializer`) in production - Java serializer is slow and produces large output

### Closure Problem

- User code passed to `map`, `filter`, `reduce` etc is a closure - captures variables from enclosing scope
- JVM closures capture the entire enclosing object if any field is referenced - not just the field
- Classic bug -
```python
    class MyProcessor:
        self.lookup_table = huge_dict       # 500 MB
        self.threshold = 10

        def process(self, rdd):
            # This closure captures 'self' → serializes entire MyProcessor including huge_dict
            return rdd.filter(lambda x: x > self.threshold)
```
- Fix - extract needed values to local variables before creating the closure
```python
    threshold = self.threshold              # local variable; only this is captured
    return rdd.filter(lambda x: x > threshold)
```

### Task Size Limit

- If serialized task size exceeds `spark.rpc.message.maxSize` (default $128 MB$) → `TaskTooLargeException`
- Common cause - accidentally capturing a large object (broadcast variable data, large collection) in closure
- Fix - use `sc.broadcast(large_obj)` and reference the broadcast inside the task

---

## ClosureCleaner

- __`ClosureCleaner`__ - Spark utility that analyzes and cleans closures before serialization; removes unnecessary references captured by the closure
- Called automatically before task serialization via `SparkContext.clean(closure)`

### How ClosureCleaner Works

- Uses ASM bytecode analysis to inspect the closure class -
    1. Identifies all outer classes referenced by the closure (via `$outer` field chains in JVM inner classes)
    2. For each outer class, identifies which fields are actually accessed by the closure
    3. Nulls out fields in the outer class that are NOT accessed
    4. This reduces serialized closure size and prevents accidental large object serialization
- Transitive cleaning - if an outer class itself has an outer class, cleans recursively
- `spark.cleaner.referenceTracking.cleanCheckpoints` - whether to clean checkpoint references

### Limitations

- Cannot clean Python closures - Python's `cloudpickle` handles Python closure serialization; operates differently
- Cannot remove fields that are accessed transitively through method calls - only direct field access is analyzed
- Does not help if the large object is directly in the closure scope (not a field of an outer class)

---

## TaskDescription

- __`TaskDescription`__ - the serialized package sent from Driver to executor containing everything needed to run a task
- Created by `TaskSchedulerImpl` when a task is assigned to an executor slot

### Fields

- `taskId: Long` - unique task ID across the application
- `attemptNumber: Int` - attempt number (0 for first attempt; increments on retry)
- `executorId: String` - target executor
- `name: String` - human-readable task name for Spark UI
- `index: Int` - task index within the `TaskSet` (= partition index)
- `partitionId: Int` - partition this task processes
- `addedFiles: Map[String, Long]` - files added via `sc.addFile`; executor fetches if not cached
- `addedJars: Map[String, Long]` - JARs added via `sc.addJar`; executor fetches if not cached
- `addedArchives: Map[String, Long]` - archives (zip/tar) added via `sc.addArchive`
- `properties: Properties` - job properties (scheduler pool, job group, job description)
- `resources: Map[String, ResourceInformation]` - GPU/custom resource assignments for this task
- `serializedTask: ByteBuffer` - serialized `Task` object (the actual RDD compute function + closure)

### Serialization

- `TaskDescription.encode(desc)` - serializes to `ByteBuffer` for RPC transport
- `TaskDescription.decode(buffer)` - deserializes on executor side
- `serializedTask` is the largest field - contains the full closure; size determined by closure cleanliness

---

## Task Lifecycle

- __PENDING__ - waiting in `TaskSetManager` queue for a resource offer
- __RUNNING__ - `LaunchTask` sent to executor; executor acknowledged and running
- __SUCCESS__ - executor sent `StatusUpdate(FINISHED)`; result or `MapStatus` registered
- __FAILED__ - executor sent `StatusUpdate(FAILED)` or executor lost; `TaskSetManager` decides retry or abort
- __KILLED__ - `TaskScheduler` sent `KillTask` (speculation winner found, stage cancelled, executor lost)
- __LOST__ - executor died while task was running; treated as failure; requeued

### Executor-Side Task Execution

- `CoarseGrainedExecutorBackend` receives `LaunchTask(taskDescription)`
- Deserializes `TaskDescription`
- Calls `executor.launchTask(context, taskDescription)` -
    - Creates `TaskRunner` (a `Runnable`)
    - Submits to executor thread pool (`threadPool`)
- `TaskRunner.run()` -
    - Deserializes task from `serializedTask`
    - Creates `TaskContext` (partition ID, attempt ID, stage ID, metrics)
    - Calls `task.run(taskAttemptId, attemptNumber, metricsSystem)` -
        - For `ShuffleMapTask` - calls `rdd.iterator` → `shuffleWriter.write`; returns `MapStatus`
        - For `ResultTask` - calls `rdd.iterator` → applies `func`; returns result
    - Sends `StatusUpdate(taskId, SUCCESS, serializedResult)` back to Driver

---

## Task Result Handling (direct vs indirect)

- When task finishes, executor serializes result and sends `StatusUpdate` to Driver
- Two paths based on result size -

### Direct Result

- Result size ≤ `spark.task.maxDirectResultSize` (default $1 MB$)
- Serialized result embedded directly in `StatusUpdate` RPC message
- Driver receives result inline; no additional fetch needed
- Fast path - single RPC round trip

### Indirect Result

- Result size > `spark.task.maxDirectResultSize` AND ≤ `spark.driver.maxResultSize` (default $1 GB$)
- Executor stores result as a block in its `BlockManager` - `TaskResultBlockId(taskId)`
- `StatusUpdate` contains only the `BlockId` and result size (not the data)
- Driver's `TaskSchedulerImpl` receives status; calls `blockManagerMaster.getLocations(blockId)`
- Driver fetches block from executor's `BlockManager` via `NettyBlockTransferService`
- After fetch, executor removes the result block from `BlockManager`

### Result Too Large

- Result size > `spark.driver.maxResultSize` → task killed; `TaskSetManager` aborts stage
- Error - "Total size of serialized results of tasks is too large"
- Common cause - `collect()` on large DataFrame
- Fix - write to storage (`df.write`) instead of collecting to Driver

### MapStatus as Result

- `ShuffleMapTask` result is always a `MapStatus` - tiny (executor location + compressed partition sizes)
- Always takes direct path - never stored in `BlockManager`
- Registered with `MapOutputTrackerMaster` by `DAGScheduler` upon receipt

---

## Speculative Execution Internals

- Enabled by `spark.speculation=true` (default `false`)
- Solves straggler tasks - one slow task holds up entire stage; often caused by bad hardware, GC pressure, data skew

### Speculation Check

- `TaskSchedulerImpl` runs periodic speculation check via `speculationScheduler` (scheduled thread pool)
- Interval - `spark.speculation.interval` (default $100 ms$)
- For each `TaskSetManager` - calls `checkSpeculatableTasks(minTimeToSpeculation)`

### Straggler Detection

- `checkSpeculatableTasks` in `TaskSetManager` -
    1. Requires at least `spark.speculation.minTaskRuntime` (default $100 ms$) of completed tasks for a reliable median
    2. Requires at least `ceil(spark.speculation.quantile × numTasks)` tasks complete (default $0.75$ → $75%$ done)
    3. Computes median duration of completed tasks
    4. Marks task as speculative if -
        - `taskRuntime > spark.speculation.multiplier × median` (default $1.5×$) AND
        - `taskRuntime > spark.speculation.minTaskRuntime`
    5. Adds task index to `speculatableTasks` set

### Speculative Task Launch

- Speculative task added to pending queues with `ANY` locality (no preference - just run it somewhere fast)
- `resourceOffers` picks it up and launches as a new task attempt on a different executor
- Both original and speculative task run simultaneously - first to finish wins

### Winner and Loser Handling

- First attempt to complete sends `StatusUpdate(SUCCESS)` to Driver
- Driver calls `TaskSetManager.handleSuccessfulTask(taskId, result)` -
    - Marks task as successful
    - If other attempt still running → calls `taskScheduler.killTask(otherTaskId, execId)`
    - Executor receives `KillTask` → interrupts thread → sends `StatusUpdate(KILLED)`
- For `ShuffleMapTask` - winning attempt's `MapStatus` registered; losing attempt's output ignored

### Speculation Pitfalls

- Non-idempotent sinks - duplicate writes if speculative task writes to external system before being killed
- For shuffle write stages - safe; only one `MapStatus` registered; other ignored
- For `ResultTask` writing to files - potential duplicate output files; use `OutputCommitCoordinator` to ensure only one attempt commits

---

## Blacklisting / Node Exclusion

- Tracks executors and nodes with repeated failures; excludes them from future task scheduling
- Renamed from "blacklisting" to "exclusion" in Spark 3.1 for terminology reasons; both config prefixes work

### Exclusion Levels

- __Task-level__ - executor excluded for a specific task -
    - `spark.excludeOnFailure.task.maxTaskAttemptsPerExecutor` (default $1$) - if task fails on this executor once → excluded for this task
- __Stage-level__ - executor excluded for entire stage -
    - `spark.excludeOnFailure.stage.maxFailedTasksPerExecutor` (default $2$) - if $2$ different tasks fail on same executor in same stage → executor excluded for stage
    - `spark.excludeOnFailure.stage.maxFailedExecutorsPerNode` (default $2$) - if $2$ executors on same node excluded → entire node excluded for stage
- __Application-level__ - executor excluded for entire application -
    - `spark.excludeOnFailure.application.maxFailedTasksPerExecutor` (default $2$) - accumulated stage-level exclusions
    - `spark.excludeOnFailure.application.maxFailedExecutorsPerNode` (default $2$) - node-level application exclusion

### Exclusion Timeout

- `spark.excludeOnFailure.timeout` (default $1h$) - exclusions expire after this duration
- After expiry, executor considered healthy again
- Prevents permanent exclusion from transient infrastructure issues

### Executor Killing

- `spark.excludeOnFailure.killExcludedExecutors=true` - kills application-level excluded executors
- Triggers `ExecutorAllocationManager` to request new replacement executors from cluster manager
- Useful for replacing physically unhealthy nodes

---

## FIFO Scheduler

- __FIFO__ - default intra-application scheduling mode (`spark.scheduler.mode=FIFO`)
- Jobs submitted first get all available resources first
- Later jobs wait until earlier jobs release resources

### Implementation

- `FIFOSchedulableBuilder` creates a single flat pool containing all `TaskSetManager`s
- `FIFOSchedulingAlgorithm` - comparator used to order `TaskSetManager`s -
    - Primary sort - job ID ascending (earlier job = higher priority)
    - Secondary sort - stage ID ascending (earlier stage = higher priority within same job)
- `resourceOffers` iterates `TaskSetManager`s in FIFO order; assigns tasks greedily

### When FIFO is Appropriate

- Sequential batch workloads - one job at a time; no concurrency needed
- Single-user environments - no need to share resources between competing users
- Simple pipelines where job order is meaningful

---

## Fair Scheduler

- __Fair Scheduler__ - interleaves resources across jobs in round-robin across pools (`spark.scheduler.mode=FAIR`)
- Resources shared proportionally according to pool weights and minimum shares
- Good for multi-user notebooks, concurrent queries, interactive workloads

### Pool Hierarchy

- Root pool contains named pools (or default pool)
- Each pool contains `TaskSetManager`s for jobs assigned to that pool
- Jobs assigned to pools via `spark.scheduler.pool` property set on `SparkSession` -
```python
    spark.sparkContext.setLocalProperty("spark.scheduler.pool", "pool_A")
    df.count()      # this job goes to pool_A
```

### Fair Scheduling Algorithm

- `FairSchedulingAlgorithm` - comparator ordering schedulables (pools or `TaskSetManager`s) -
    - Computes `runningTasks` / `weight` ratio for each schedulable
    - Schedulable with lower ratio gets priority (it is more "starved" relative to its weight)
    - Ties broken by `minShare` - schedulable below `minShare` gets priority
    - Further ties broken by name (alphabetical)

---

## Fair Scheduler Pools & Weights

- Pool configuration via XML file - `spark.scheduler.allocation.file` (default `fairscheduler.xml` on classpath)

```xml
<allocations>
    <pool name="production">
        <schedulingMode>FAIR</schedulingMode>
        <weight>2</weight>
        <minShare>4</minShare>
    </pool>
    <pool name="development">
        <schedulingMode>FIFO</schedulingMode>
        <weight>1</weight>
        <minShare>0</minShare>
    </pool>
</allocations>
```

### Pool Properties

- `weight` - relative share of resources; `production` pool gets $2×$ resources of `development` pool
- `minShare` - minimum number of tasks guaranteed to run even when cluster is busy; ensures small pools are not starved
- `schedulingMode` - scheduling within the pool; `FAIR` (round-robin across jobs in pool) or `FIFO` (job order within pool)

### Resource Allocation Math

- Available slots = $10$, `production` weight = $2$, `development` weight = $1$
- `production` gets $\frac{2}{3} × 10 ≈ 7$ slots; `development` gets $\frac{1}{3} × 10 ≈ 3$ slots
- `minShare` overrides - if `production` has `minShare=4` and only $2$ tasks pending, remaining slots available to `development`
- Weights only apply when both pools have pending tasks; idle pool's allocation given to active pool

### Default Pool

- Jobs not assigned to a named pool go to the `default` pool
- Default pool has weight $1$, `minShare` $0$, `FIFO` scheduling mode (unless overridden in XML)

---

## Dynamic Allocation Internals

- __Dynamic allocation__ - automatically scales executor count up and down based on workload; eliminates fixed executor count waste
- Enabled via `spark.dynamicAllocation.enabled=true`
- Requires either `ExternalShuffleService` or Shuffle Tracking (K8s)

### Scale-Up Logic

- Triggered when there are pending tasks that cannot be scheduled on current executors
- `ExecutorAllocationManager` detects pending tasks via `TaskScheduler.pendingTasksExist()`
- Initial request - if tasks pending for `spark.dynamicAllocation.schedulerBacklogTimeout` (default $1s$) → request executors
- Subsequent requests - if tasks still pending after `spark.dynamicAllocation.sustainedSchedulerBacklogTimeout` (default $1s$) → request more
- Request size doubles each round - $1$, $2$, $4$, $8$... executors until `spark.dynamicAllocation.maxExecutors`
- Upper bound - `spark.dynamicAllocation.maxExecutors` (default `Int.MaxValue`)

### Scale-Down Logic

- Triggered when executors are idle (no running tasks)
- Executor idle for `spark.dynamicAllocation.executorIdleTimeout` (default $60s$) → request removal
- Executor with cached data idle for `spark.dynamicAllocation.cachedExecutorIdleTimeout` (default `Int.MaxValue`) → separate (higher) threshold to protect cached data
- Lower bound - `spark.dynamicAllocation.minExecutors` (default $0$)
- `spark.dynamicAllocation.executorAllocationRatio` (default $1.0$) - fraction of needed executors to request; set < $1.0$ to under-provision intentionally (avoid over-allocation for bursty workloads)

### Initial Executors

- `spark.dynamicAllocation.initialExecutors` (default = `spark.dynamicAllocation.minExecutors`) - executors requested at application start before any tasks run

---

## ExecutorAllocationManager

- __`ExecutorAllocationManager`__ - the Driver-side component that implements dynamic allocation logic
- Runs a scheduling loop every `spark.dynamicAllocation.schedulerBacklogTimeout`
- Communicates with `SchedulerBackend` via `ExecutorAllocationClient` interface

### Internal State

- `executorsPendingToRemove: Set[String]` - executors requested for removal; waiting for cluster manager acknowledgment
- `removeTimes: Map[String, Long]` - timestamp when each executor became idle; used to compute idle timeout
- `numExecutorsTarget: Int` - current target executor count; adjusted up/down by allocation logic
- `numExecutorsToAdd: Int` - number of executors to request in next batch; doubles each round

### Allocation Loop

- `schedule()` called periodically -
    1. `updateAndSyncNumExecutorsTarget(now)` - recomputes target based on pending tasks and running tasks
    2. `addOrRemoveExecutors()` -
        - If `numExecutorsTarget > current` → call `client.requestExecutors(delta)`
        - If idle executors exist past timeout → call `client.killExecutors(idleExecs)`
    3. `removeTimes` updated for newly idle executors

### Executor Tracking

- Listens to `SparkListenerExecutorAdded` / `SparkListenerExecutorRemoved` events
- `onExecutorAdded` - adds executor to `executorIds`; resets `removeTimes` for executor
- `onExecutorRemoved` - removes from `executorIds`; cleans up tracking state
- `onTaskStart` - removes executor from `removeTimes` (no longer idle)
- `onTaskEnd` - if executor now has no running tasks → adds to `removeTimes` with current timestamp

---

## ExecutorAllocationClient

- __`ExecutorAllocationClient`__ - interface between `ExecutorAllocationManager` and `SchedulerBackend`; abstracts executor request/kill operations across cluster managers

### Interface Methods

- `requestExecutors(numAdditionalExecutors: Int): Boolean` - request more executors; returns whether request acknowledged
- `requestTotalExecutors(numExecutors: Int, localityAwareTasks: Int, hostToLocalTaskCount: Map[String, Int]): Boolean` -
    - Request specific total executor count
    - `localityAwareTasks` - number of tasks with locality preferences; used by YARN for locality-aware container placement
    - `hostToLocalTaskCount` - per-host task counts; YARN uses to request containers on specific nodes
- `killExecutors(executorIds: Seq[String], adjustTargetNumExecutors: Boolean, countFailures: Boolean, force: Boolean): Seq[String]` - request executor removal; returns IDs actually killed
- `killExecutorsOnHost(host: String): Boolean` - kill all executors on a node (used after node exclusion)
- `getExecutorIds(): Seq[String]` - current active executor IDs

### Cluster Manager Implementations

- `StandaloneSchedulerBackend` - sends `RequestExecutors` / `KillExecutors` to Standalone master
- `YarnSchedulerBackend` - adjusts YARN container requests via `YarnAllocator`
- `KubernetesSchedulerBackend` - creates/deletes executor pods via K8s API

### Locality Hints in requestTotalExecutors

- `hostToLocalTaskCount` passed to YARN allocator - YARN uses to request containers on nodes with local data
- Prevents dynamic allocation from requesting containers on random nodes when data locality matters
- `ExecutorAllocationManager` computes this map from `TaskScheduler.getLocalityAwareTasks()`
