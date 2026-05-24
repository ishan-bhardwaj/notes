# DAGScheduler

- Translates the logical RDD DAG (or physical plan from Catalyst) into a _physical execution plan of Stages_
- `DAGScheduler` receives the final RDD (the last RDD in the lineage chain) when an action is called
- Walks the lineage backwards recursively to discover the full DAG
- Operates entirely on the Driver - no executor involvement at this layer

## Stage Types

- __`ShuffleMapStage`__ -
    - Any stage _before the final stage_
    - Its tasks write shuffle output to local disk for downstream stages to consume
    - Can be _shared across multiple jobs_ - if two jobs need the same shuffle output and it's still available, `DAGScheduler` reuses it

- __`ResultStage`__ -
    - The _final stage_ in a job - directly computes the result of an action
    - There is exactly one `ResultStage` per job
    - Either return data to the Driver (`collect`) or write to external storage (`write`)

> [!TIP]
> A job with $N$ shuffles produces $N$ ShuffleMapStages + $1$ ResultStage

## DAGScheduler Algorithm

1. Action is called on an RDD → `SparkContext.runJob()` → handed to `DAGScheduler.runJob()`
2. `DAGScheduler` calls `getOrCreateResultStage(finalRDD)` - starts building stages backwards
3. For the final RDD, it creates a `ResultStage`
4. It walks the RDD's dependencies recursively -
    - If dependency is _narrow_ → same stage, keep walking back
    - If dependency is _wide_ (`ShuffleDependency`) → create a new `ShuffleMapStage` for the parent; record this as a stage dependency
5. This recursion continues until it reaches RDDs with no parents (source RDDs - file scans, `parallelize`, etc.)
6. Result - a DAG of stages where edges represent shuffle dependencies

## Stage Reuse

- `DAGScheduler` maintains a `HashMap[Int, ShuffleMapStage]` keyed by shuffle ID
- Before creating a new `ShuffleMapStage`, it checks if one already exists for that shuffle
- If a previous job already computed and cached that shuffle output, it is reused - the stage is not rerun
- This is how iterative algorithms (ML training loops) avoid recomputing the same shuffle repeatedly if the base data hasn't changed

## Data Locality

- `DAGScheduler` computes _preferred locations_ for every task before submitting to `TaskScheduler`
- Goal - run the task on the node where its input data already lives, avoiding network reads
- __Locality levels__ - 
    - `PROCESS_LOCAL` - data is in the same executor's memory (cached partition)
    - `NODE_LOCAL` - data is on the same node (HDFS block, cached on another executor on the same node)
    - `RACK_LOCAL` - data is on the same rack (same switch), different node
    - `ANY` - data must be fetched over the network from a different rack

- Computing preferred locations -
    - __Cached RDDs__ - 
        - `BlockManagerMaster` on the Driver knows which executors have which cached partitions
        - `DAGScheduler` queries it to get the executor/node location
    - __HDFS/S3 inputs__ - 
        - `FileInputFormat` (or Spark's own file source) returns the HDFS block locations for each split
        - `DAGScheduler` uses these as preferred locations
    - __Shuffle inputs__ - 
        - For the reduce side of a shuffle, there is no meaningful preferred location (data comes from all map outputs)
        - `DAGScheduler` sets locality to `ANY` for shuffle read tasks

### Delay Scheduling

- `DAGScheduler` passes preferred locations to `TaskScheduler`
- `TaskScheduler` implements _delay scheduling_ - 
    - Waits a configurable time (`spark.locality.wait`, default $3s$) for a slot to open on the preferred node before falling back to a worse locality level
- `DAGScheduler` itself doesn't implement delay scheduling - it just provides the location hints

## Stage Submission and Ordering

- `DAGScheduler` only submits a stage when _all its parent stages are complete_
- It maintains a set of 
    - `waitingStages` - parents not yet done
    - `runningStages` - currently executing
    - `failedStages` - failed, awaiting resubmission
- When a `ShuffleMapStage` completes, `DAGScheduler` checks if any waiting stages are now unblocked and submits them
- Stages at the same level with no dependency between them can run _concurrently_ - 
    - `DAGScheduler` submits them all at once
    - `TaskScheduler` interleaves their tasks across available executor slots

## Missing Shuffle Output Handling

- When a `ShuffleMapStage` completes, its map output locations are registered with `MapOutputTracker` (a Driver-side component)
- Reduce tasks query `MapOutputTracker` to find where to fetch each partition's shuffle data
- If an executor dies and its shuffle output is lost -
    - The reduce task that tries to fetch it reports a `FetchFailedException`
    - `DAGScheduler` catches this, marks the map output as lost in `MapOutputTracker`
    - Resubmits the affected `ShuffleMapStage` (only the lost partitions need recomputation if the `ExternalShuffleService` was running; otherwise the whole stage reruns)
    - After the `ShuffleMapStage` recomputes, the blocked `ResultStage` or downstream `ShuffleMapStage` is resubmitted

## Event-Driven Architecture

- DAGScheduler is _fully event-driven_ internally - 
    - It runs an event loop (`DAGSchedulerEventProcessLoop`) on a dedicated thread
- All external interactions post events to this loop - nothing is called directly
- Key events -
    - `JobSubmitted` - action triggered, new job to process
    - `StageCancelled` / `JobCancelled` - cancellation from user or timeout
    - `CompletionEvent` - a task finished (success or failure), posted by `TaskScheduler`
    - `ExecutorLost` - an executor died; need to check for lost shuffle outputs
    - `TaskSetFailed` - all retries for a `TaskSet` exhausted; stage has failed
    - `ResubmitFailedStages` - retry failed stages after a delay

- This design means `DAGScheduler` never blocks - 
    - The event loop processes one event at a time, keeping internal state consistent without locks

## Failure Handling

### Task Failure

- `TaskScheduler` handles individual task retries (up to `spark.task.maxFailures`)
- If a task fails with `FetchFailedException` - 
    - `DAGScheduler` is notified directly (not just `TaskScheduler`) 
    - Because this is a shuffle data loss problem, not just a task problem
- DAGScheduler treats `FetchFailedException` as a signal to resubmit the parent `ShuffleMapStage`

### Stage Failure

- If `TaskScheduler` reports that a `TaskSet` has failed (all retries exhausted) → `DAGScheduler.handleTaskSetFailed()` → marks the stage failed
- Triggers an `AbortStage` - the stage and all jobs depending on it are aborted
- The job fails and the user gets an exception from the action call

### Executor Loss

- `ExecutorLost` event is posted to `DAGScheduler`'s event loop
- `DAGScheduler` checks `MapOutputTracker` - which shuffle map outputs were on that executor?
- Marks those map outputs as lost
- Any running stage that was consuming those outputs gets its fetch failures surfaced
- Affected `ShuffleMapStages` are resubmitted

### Stage Retry Limit

- `spark.stage.maxConsecutiveAttempts` (default $4$) - 
    - A stage that fails this many times in a row causes the job to abort entirely

## MapOutputTracker

- Driver-side component that `DAGScheduler` works closely with
- Tracks the _locations of all shuffle map outputs_ - for each shuffle, for each map partition, which executor/host has the output data
- When a `ShuffleMapStage` completes, each map task reports its output locations to `MapOutputTracker` via the Driver
- When reduce tasks start, they query `MapOutputTracker` to find where to fetch each partition
- On executor loss, `DAGScheduler` calls `MapOutputTracker.removeOutputsOnExecutor()` to invalidate lost map outputs - 
    - This is what triggers stage resubmission

## Checkpointing Interaction

- If an RDD is checkpointed (`rdd.checkpoint()`), its lineage is _truncated_ - 
    - `DAGScheduler` sees the checkpointed RDD as a source (no parents) rather than walking further back
- This breaks long lineage chains - 
    - Without checkpointing, a failure late in a $100-step$ lineage forces `DAGScheduler` to recompute from scratch
- `DAGScheduler` calls `rdd.doCheckpoint()` after each job completes for any RDDs marked for checkpointing whose jobs have now finished
- For Structured Streaming - checkpoint location stores committed batch IDs and operator state -
    - `DAGScheduler`'s stage/lineage recomputation model is what makes _"exactly once"_ recovery possible

## RDD Cache Interaction

- `DAGScheduler` checks `BlockManagerMaster` for cached RDD partitions before computing preferred locations
- If a partition is cached on an executor - 
    - That partition's task gets `PROCESS_LOCAL` as preferred location
    - `TaskScheduler` will try hard to schedule it there
- If an executor with cached partitions dies - 
    - `DAGScheduler` doesn't automatically recompute the cache
    - It simply recomputes the partition from lineage when a task needs it
- `rdd.persist()` marks the RDD for caching
    - The actual caching happens the _first time_ a task computes that partition and writes it to the `BlockManager`

## Key Configs

| Configuration                        | Default               | What It Controls                                                                                       |
| ------------------------------------ | --------------------- | ------------------------------------------------------------------------------------------------------ |
| `spark.locality.wait`                | `3s`                  | Maximum time to wait for a preferred locality level before falling back to a less-local task placement |
| `spark.locality.wait.process`        | `spark.locality.wait` | Wait time specifically for `PROCESS_LOCAL` scheduling                                                  |
| `spark.locality.wait.node`           | `spark.locality.wait` | Wait time specifically for `NODE_LOCAL` scheduling                                                     |
| `spark.locality.wait.rack`           | `spark.locality.wait` | Wait time specifically for `RACK_LOCAL` scheduling                                                     |
| `spark.stage.maxConsecutiveAttempts` | `4`                   | Maximum number of consecutive stage retries before aborting the job                                    |
| `spark.sql.shuffle.partitions`       | `200`                 | Default number of partitions used for shuffle operations in Spark SQL/DataFrame workloads              |

