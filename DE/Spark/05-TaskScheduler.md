# TaskScheduler

- Receives _TaskSets_ from `DAGScheduler` and gets them executed on executors
- Sits between `DAGScheduler` and the cluster - 
    - `DAGScheduler` thinks in terms of stages and preferred locations
    - `TaskScheduler` thinks in terms of executor slots, resource offers, and task lifecycle
- Operates entirely on the Driver
- Primary concerns -
    - Matching tasks to executor slots (respecting locality preferences)
    - Handling task retries and executor blacklisting
    - Speculative execution for straggler tasks
    - Reporting task completion/failure back to `DAGScheduler`

- `TaskScheduler` is an interface - the real implementation is `TaskSchedulerImpl`
- `SchedulerBackend` is a separate interface below it - abstracts the cluster manager (`StandaloneSchedulerBackend`, `YARNSchedulerBackend`, `KubernetesSchedulerBackend` etc.)

## Components TaskScheduler Owns

- `TaskSetManager` - 
    - One per active `TaskSet`
    - Tracks every task in a stage, handles retries, locality, speculation
    - This is where most of the real scheduling logic lives
- `SchedulerBackend` - 
    - The cluster-facing layer
    - Receives resource offers from the cluster manager and relays them up to `TaskScheduler`
- _Blacklist / Exclusion tracker_ - 
    - Tracks which executors/nodes have had repeated failures and excludes them from future scheduling

## Resource Offers - How Tasks Get Assigned

- `TaskScheduler` does _not_ push tasks to executors proactively
- Instead, `SchedulerBackend` receives resource offers from the cluster manager (a free executor slot has appeared) and calls `TaskScheduler.resourceOffers(offers)`
- `resourceOffers()` is the core scheduling method - called every time executor slots free up
- Flow -
    1. `SchedulerBackend` gets notified that executor slots are available
    2. It calls `TaskScheduler.resourceOffers(workerOffers)` - each offer is `(executorId, host, freeCores)`
    3. `TaskScheduler` shuffles the offers (to avoid hotspotting the same executors)
    4. For each active `TaskSetManager`, it tries to assign tasks to offers respecting locality
    5. Returns a list of `TaskDescription` objects - one per slot that got a task assigned
    6. `SchedulerBackend` serializes and launches those tasks on the corresponding executors

- This offer-based model means `TaskScheduler` is _reactive_, not proactive - 
    - It only schedules when the cluster tells it slots are free

## TaskSetManager

- Every stage submission creates one `TaskSetManager` wrapping the stage's `TaskSet`
- `TaskSetManager` is responsible for -
    - Tracking which tasks are pending, running, finished, or failed
    - Maintaining the _locality-aware pending task queues_
    - Deciding which task to assign to a given resource offer (locality logic lives here)
    - Retrying failed tasks
    - Launching speculative tasks for stragglers
    - Reporting stage completion or failure back to `TaskScheduler` → `DAGScheduler`

### Pending Task Queues

- `TaskSetManager` maintains _separate queues per locality level_ -
    - `pendingTasksForExecutor` - tasks that prefer a specific executor (`PROCESS_LOCAL`)
    - `pendingTasksForHost` - tasks that prefer a specific host (`NODE_LOCAL`)
    - `pendingTasksForRack` - tasks that prefer a specific rack (`RACK_LOCAL`)
    - `pendingTasksWithNoPrefs` - tasks with no locality preference (no-pref / `ANY`)
    - `allPendingTasks` - all remaining tasks regardless of preference
- When a resource offer arrives for executor `E` on host `H` in rack `R`
    - `TaskSetManager` tries queues in preference order - `PROCESS_LOCAL` first, then `NODE_LOCAL`, then `RACK_LOCAL`, then `ANY`
    - A task is removed from all queues once it is assigned

## Delay Scheduling

- When a resource offer arrives and the best available task for that slot is at a worse locality level than desired, `TaskScheduler` does not immediately fall back
- It _waits_ for a better-locality slot to open up - this is delay scheduling
- Implemented via a _last-launch timestamp_ per locality level in `TaskSetManager` -
    - If time since last task launch at the current level < `spark.locality.wait` → skip this offer and wait for a better one
    - If wait has expired → fall back to the next locality level
- Locality wait configs -
    - `spark.locality.wait` (default $3s$) - base wait, applies to all levels unless overridden
    - `spark.locality.wait.process` - wait for `PROCESS_LOCAL`
    - `spark.locality.wait.node` - wait for `NODE_LOCAL`
    - `spark.locality.wait.rack` - wait for `RACK_LOCAL`
- Tradeoff - higher locality wait = better data locality = less network I/O, but tasks sit idle longer waiting for the right slot
- For shuffle-read stages (reduce side), all data comes from the network anyway - locality wait is irrelevant -
    - Tasks should use `ANY` and `spark.locality.wait` should effectively be $0$ here

## Task Lifecycle

- __PENDING__ - waiting in `TaskSetManager` queue for a resource offer
- __RUNNING__ - assigned to an executor, currently executing
- __SUCCESS__ - executor reported successful completion; result or shuffle location registered
- __FAILED__ - executor reported failure or heartbeat timed out; `TaskSetManager` decides retry or abort
- __KILLED__ - `TaskScheduler` explicitly killed the task (speculation winner found, stage cancelled, executor lost)

## Task Serialization

- When `resourceOffers()` assigns a task to a slot, `TaskScheduler` creates a `TaskDescription`
- `TaskDescription` contains -
    - Serialized task object (the RDD compute function + closure)
    - Task ID, attempt number, partition ID
    - Serialized `Properties` (job group, description, fair scheduler pool)
    - List of jar/file dependencies to fetch
- The _closure_ serialized with the task includes everything the lambda captures - 
    - This is a common source of accidental large task serialization (capturing a large object in scope)
- Closure serialization uses the configured serializer (`spark.serializer`) - always use Kryo in production
- If a task's serialized size exceeds `spark.rpc.message.maxSize` (default $128MB$) - 
    - It will fail with a "task too large" error
    - Usually means you accidentally captured a large broadcast variable in the closure instead of using `sc.broadcast()`

## Task Result Handling

- When a task finishes, the executor sends the result back to the Driver via the `SchedulerBackend`
- Two paths depending on result size -
    - __Direct result__ - 
        - Result fits in `spark.task.maxDirectResultSize` (default $1MB$)
        - Sent inline in the status update message
    - __Indirect result__ - 
        - Result is stored in the executor's `BlockManager` and the Driver fetches it separately
        - Used for results up to `spark.driver.maxResultSize` (default $1GB$)
    - If result exceeds `spark.driver.maxResultSize` → task is killed and job fails with _"result size exceeds"_ error - 
        - Common when `collect()` is called on a large DataFrame
- `TaskScheduler` calls `TaskSetManager.handleSuccessfulTask()` → 
    - Updates task state, checks if all tasks in the stage are done
    - If yes, notifies `DAGScheduler` that the stage is complete

## Failure Handling

### Individual Task Failure

- Executor reports failure (exception, OOM, lost executor) → 
    - `SchedulerBackend` notifies `TaskScheduler` → `TaskSetManager.handleFailedTask()`
- `TaskSetManager` increments the failure count for that task
- If failure count < `spark.task.maxFailures` (default $4$) → 
    - task goes back to `PENDING` queue
    - Will be retried on a different executor
- Retried tasks get a new _attempt ID_ - important for idempotency - 
    - Sinks that are not idempotent may see duplicate writes from speculative or retried tasks

### `FetchFailedException` - Special Case

- If a task fails while fetching shuffle data (the map output it needs is gone) → `FetchFailedException`
- `TaskScheduler` catches this and routes it directly to `DAGScheduler` (not just a normal retry) - because the problem is not the task itself but the missing shuffle data upstream
- `DAGScheduler` then resubmits the parent `ShuffleMapStage`

### Executor Loss

- `SchedulerBackend` detects executor loss (missed heartbeat, cluster manager notification)
- Notifies `TaskScheduler` → `TaskScheduler.executorLost()`
- All running tasks on that executor are marked failed and requeued
- TaskSetManagers for stages that were writing shuffle output on that executor are notified
- `DAGScheduler` is notified via `TaskScheduler.dagScheduler.executorLost()`

### Task Failure vs Stage Failure

- Each task has an independent retry counter
- Each executor also has a failure counter tracked by the exclusion tracker
- If a single task fails on `spark.task.maxFailures` different executors → 
    - `TaskSetManager` calls `abort()` → 
    - stage fails → 
    - `DAGScheduler` gets `TaskSetFailed` event → 
    - job aborts
- This prevents infinite retry loops when the failure is caused by bad data in a specific partition (not a transient infrastructure failure)

## Speculative Execution

- Enabled by `spark.speculation=true` (default `false`)
- Problem it solves - _straggler tasks_ i.e. one slow task holds up an entire stage even though $99$ other tasks finished; often caused by bad hardware, GC pressure, or data skew
- __Working__ -
    1. `TaskScheduler` runs a periodic speculation check (every `spark.speculation.interval`, default $100ms$)
    2. For each running `TaskSetManager`, it checks if any running task is taking significantly longer than the median of completed tasks in that stage
    3. A task is marked for speculation if its runtime > `spark.speculation.multiplier` (default $1.5$) × median task runtime AND the stage has at least `spark.speculation.minTaskRuntime` (default $100ms$) of completed tasks to compute a reliable median
    4. The speculative task is launched on a different executor as a new task attempt
    5. Whichever attempt finishes first - the other is killed
- Speculative execution is a _duplicate execution_ - both the original and speculative task run simultaneously -
    - Non-idempotent sinks (writing to a DB, appending to a file) can get duplicate writes
    - For shuffle write stages, the speculative winner's output is registered in `MapOutputTracker`
      The loser's output is ignored
- `spark.speculation.quantile` (default $0.75$) - 
    - Fraction of tasks in a stage that must complete before speculation kicks in
    - Prevents speculating too early when the median is unreliable

## Executor Blacklisting / Exclusion

- Tracks executors and nodes that repeatedly cause task failures
- Per-task - 
    - If a task fails on the same executor multiple times → `spark.excludeOnFailure.task.maxTaskAttemptsPerExecutor` (default $1$) - executor is excluded for that task
- Per-stage - 
    - If too many tasks fail on the same executor → `spark.excludeOnFailure.stage.maxFailedTasksPerExecutor` (default $2$) - executor excluded for entire stage
- Per-application - 
    - If an executor accumulates too many stage-level exclusions → `spark.excludeOnFailure.application.maxFailedTasksPerExecutor` (default $2$) - executor excluded for the entire application
- Node-level exclusion - if enough executors on the same node are excluded, the whole node is excluded
- Exclusions are time-bounded - `spark.excludeOnFailure.timeout` (default $1h$)
    - After expiry the executor is considered clean again
- With YARN/K8s, application-level excluded executors can be killed and replaced via `spark.excludeOnFailure.killExcludedExecutors=true`

## Fair Scheduling vs FIFO

- `TaskScheduler` supports two _intra-application_ scheduling modes (across multiple jobs within the same `SparkSession`)
- `spark.scheduler.mode` -
    - `FIFO` (default) - 
        - Jobs submitted first get all available resources first, later jobs wait
        - Simple, good for sequential batch workloads
    - `FAIR` - 
        - Resources are interleaved across jobs in round-robin across pools
        - Uses _pools_ - each job is assigned to a pool (default pool or named pool via `spark.scheduler.pool`)
        - Pools have weights and `minShare` (minimum guaranteed slots)
        - Good for multi-user notebooks or concurrent queries where you don't want one big job to starve others
- This is purely intra-application scheduling - has nothing to do with how the cluster manager allocates resources between different Spark applications

## Dynamic Allocation Interaction

- When dynamic allocation is enabled, `TaskScheduler` tells the `SchedulerBackend` when to request more executors or release idle ones
- `TaskScheduler.executorAvailable()` - 
    - When new executor are registered, `TaskScheduler` immediately tries to assign pending tasks to it
    - Idle detection - `TaskScheduler` tracks when each executor last ran a task - 
        - If idle for `spark.dynamicAllocation.executorIdleTimeout` (default $60s$) → 
        - `SchedulerBackend` requests the cluster manager to release it
- Pending tasks accumulate → `SchedulerBackend` requests new executors from cluster manager
- This feedback loop is driven by `TaskScheduler`'s visibility into pending task queues

## Key Configs

| Config                           | Default | What it controls                                                         |
| -------------------------------- | ------- | ------------------------------------------------------------------------ |
| `spark.task.maxFailures`         | `4`     | Retry limit per task before stage abort                                  |
| `spark.locality.wait`            | `3s`    | Base delay scheduling wait across all locality levels                    |
| `spark.speculation`              | `false` | Enables speculative execution for slow/straggler tasks                   |
| `spark.speculation.interval`     | `100ms` | Frequency for checking straggler tasks                                   |
| `spark.speculation.multiplier`   | `1.5`   | Runtime threshold vs median task runtime to trigger speculation          |
| `spark.speculation.quantile`     | `0.75`  | Fraction of completed tasks required before speculation starts           |
| `spark.scheduler.mode`           | `FIFO`  | Intra-application scheduling mode (`FIFO` or `FAIR`)                     |
| `spark.task.maxDirectResultSize` | `1MB`   | Max task result size sent directly to driver before using `BlockManager` |
| `spark.driver.maxResultSize`     | `1GB`   | Max total serialized result size allowed on driver before job failure    |
| `spark.excludeOnFailure.enabled` | `false` | Enables executor/node exclusion after repeated failures                  |
| `spark.rpc.message.maxSize`      | `128MB` | Maximum serialized RPC message size (e.g. task binary, shuffle metadata) |
