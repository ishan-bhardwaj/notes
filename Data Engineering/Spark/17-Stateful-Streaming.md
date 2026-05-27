# Stateful Streaming

## Watermarking Internals

- __Watermark__ - a moving threshold in event time that tells Spark "all data with timestamp ≤ watermark is considered complete; late data beyond this threshold may be dropped"
- Defined via `df.withWatermark("timestamp_col", "delayThreshold")` where `delayThreshold` is a duration string (eg - `"10 minutes"`, `"1 hour"`)
- Formula - `watermark = max(event_time_seen_so_far) - delayThreshold`
- Watermark advances monotonically - never decreases; once state older than watermark, it is eligible for cleanup

### Watermark Computation

- `EventTimeWatermarkExec` - physical operator that tracks event time watermark
- Each task tracks the maximum event time seen in its partition
- After each batch, `MicroBatchExecution` collects the maximum event time across all tasks
- New watermark = `global_max_event_time - delayThreshold`
- New watermark ONLY replaces old watermark if it is HIGHER - monotonically increasing

### Watermark Propagation

- Watermark stored in `CommitLog` metadata (`nextBatchWatermarkMs`) after each committed batch
- On next batch start, watermark loaded from `CommitLog`; injected into physical plan
- All stateful operators receive watermark via `StatefulOperatorStateInfo`
- `StateStoreSaveExec` uses watermark to determine which state keys are expired and can be evicted

### Watermark in Physical Plan

- `withWatermark` inserts `EventTimeWatermark` logical node → `EventTimeWatermarkExec` physical operator
- `EventTimeWatermarkExec` wraps child; after child produces rows, extracts timestamp column; updates `WatermarkTracker` with max seen
- `WatermarkTracker` is shared across operators in same physical plan; holds current `watermarkMs`

---

## Global Watermark vs Per-Partition Watermark

- __Global watermark__ - a single watermark value across ALL partitions and ALL tasks; the minimum of all per-task maximums (actually the global maximum minus delay)
- Spark uses a GLOBAL watermark - not per-partition

### Why Global (Not Per-Partition) Watermark

- Correct watermark must guarantee completeness - "all data with timestamp ≤ watermark has arrived"
- A per-partition watermark would say "in partition 3, all data ≤ T has arrived" - but partition 3 might have received data from only some Kafka partitions
- Global watermark uses `global_max_event_time` across ALL input partitions - only safe because Spark sees all data

### Global Watermark Calculation

- Each task running `EventTimeWatermarkExec` maintains a local max event time
- After batch execution, `MicroBatchExecution` aggregates via `TaskMetrics` accumulators -
    - `EventTimeStats` accumulator collects max event time from all tasks
    - Driver computes `globalMaxEventTime = max across all TaskMetrics`
    - `newWatermark = globalMaxEventTime - delayThreshold`
- Written to `CommitLog` as `nextBatchWatermarkMs`
- All operators in NEXT batch see the updated watermark

### Watermark Lag

- Watermark always lags behind actual max event time by at least `delayThreshold`
- Additionally lags by one batch - watermark from batch $N$ data is applied in batch $N+1$
- Total maximum delay for a window to close = `delayThreshold + 1 batch duration`
- For `ProcessingTime("10 seconds")` trigger and `"10 minutes"` watermark - windows close ~10 minutes + 10 seconds after max event time in that window

---

## Late Data Handling

- __Late data__ - records whose event time is older than the current watermark; by definition, the window or state they belong to may already be finalized and evicted

### What Happens to Late Data

- For windowed aggregations with `Append` output mode -
    - Data older than watermark arrives → its window has already been emitted and state cleaned → record DROPPED
    - No error thrown; silently discarded
    - `numRowsDroppedByWatermark` metric incremented (visible in `StreamingQueryProgress`)

- For `Update` output mode -
    - Late data updates may still be processed if state exists (state not evicted yet)
    - Borderline cases (near watermark) may or may not be processed

### State Eviction via Watermark

- `StateStoreSaveExec` evicts state keys where the associated time window's end is ≤ current watermark
- For `GroupBy + window("timestamp", "5 minutes")` -
    - State key = `(groupKey, windowStart, windowEnd)`
    - Eviction condition - `windowEnd <= currentWatermark`
    - Evicted state emitted as final result (Append mode) then removed from state store

### Tuning Late Data Tolerance

- Larger `delayThreshold` → more late data tolerated → more state retained → higher memory usage
- Smaller `delayThreshold` → less late data tolerated → state evicted faster → lower memory usage
- Choose based on source characteristics - Kafka with one broker slow → need larger threshold; well-synchronized sources → smaller threshold safe

---

## StateStore Interface

- __`StateStore`__ - the abstraction for per-partition, per-operator key-value state storage in Structured Streaming; each stateful operator maintains its state in a `StateStore`
- One `StateStore` instance per partition per operator per batch
- State partitioned by `groupBy` key hash; each Spark partition holds state for a subset of keys

### StateStore Operations

- `get(key: UnsafeRow): UnsafeRow` - get value for key; returns `null` if not present
- `put(key: UnsafeRow, value: UnsafeRow): Unit` - set/update value for key
- `remove(key: UnsafeRow): Unit` - delete key from state
- `iterator(): Iterator[(UnsafeRow, UnsafeRow)]` - iterate all key-value pairs (used for eviction and output)
- `commit(): Long` - commit current batch's state changes; returns new version number
- `abort(): Unit` - discard uncommitted changes
- `metrics: StateStoreMetrics` - row count, memory size, etc.

### StateStore Versioning

- Each committed batch produces a new version of the state store
- Version = batch ID
- `StateStoreProvider` manages the version history
- On recovery - state store loaded from the version corresponding to last committed batch

### StateStore Provider

- `StateStoreProvider` - manages state store instances for a partition; holds all versions
- `HDFSBackedStateStoreProvider` - default; stores state in HDFS/S3
- `RocksDBStateStoreProvider` - embedded RocksDB per executor; local SSD storage

---

## HDFSBackedStateStore

- __`HDFSBackedStateStore`__ - default state store implementation; stores state as delta files on HDFS/S3; state kept in executor memory between batches

### Storage Layout

```
checkpointLocation/state/
└── operatorId/
└── partitionId/
├── 1.delta     ← changes in batch 1 (new/updated/deleted keys)
├── 2.delta     ← changes in batch 2
├── 3.delta
├── ...
└── 5.snapshot  ← full state snapshot at batch 5 (periodic)
```

### Delta Files

- Each committed batch writes a delta file containing ONLY the changes (puts + removes) for that batch
- Delta file is a Hadoop sequence file of `(key, value)` pairs where `value = null` for removes
- Small delta files = fast writes; but reading state requires replaying all deltas from last snapshot

### Snapshot Files

- Periodic full snapshots written every `spark.sql.streaming.stateStore.maintenanceInterval` (default $60s$) or every $N$ batches
- Snapshot = complete state as a sequence file
- Recovery reads last snapshot + all subsequent deltas; faster than replaying all deltas from beginning

### In-Memory State

- `HDFSBackedStateStore` maintains full state as `HashMap[UnsafeRow, UnsafeRow]` in executor JVM heap
- State loaded from HDFS on first access for a partition; held in memory for subsequent batches
- Memory usage = all state for this partition; can cause GC pressure for large state
- `spark.sql.streaming.stateStore.providerClass` - configure state store implementation

### HDFSBackedStateStore Limitations

- All state for a partition must fit in executor JVM heap
- Large state → GC pressure → long GC pauses → batch latency spikes
- State stored in JVM heap → subject to GC overhead
- For large state use cases → use `RocksDBStateStore`

---

## RocksDBStateStore

- __`RocksDBStateStore`__ - state store implementation using embedded RocksDB; stores state on local executor disk (SSD); not in JVM heap; supports state larger than available heap
- Enabled via `spark.sql.streaming.stateStore.providerClass=org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider`

### Why RocksDB

- JVM heap independence - state stored on local disk (NVMe SSD ideal); only active working set in memory
- No GC pressure from state - RocksDB manages its own memory via block cache
- Better for large state - terabytes of state possible (bounded by disk, not heap)
- Write-optimized - LSM tree structure; fast writes; compaction in background

### RocksDB in Spark

- One RocksDB instance per partition per operator on each executor
- State stored as RocksDB key-value pairs with `UnsafeRow` byte serialization for both key and value
- RocksDB instance lives in executor local directory (`spark.local.dir`)
- State checkpointed to HDFS/S3 via RocksDB checkpoint (hard-links to SST files) + delta logs

### RocksDB Checkpointing

- Each batch -
    1. Apply puts/removes to local RocksDB instance
    2. Create RocksDB checkpoint (near-instantaneous hard-link snapshot of SST files)
    3. Upload changed SST files to HDFS/S3 checkpoint location (only changed files, not full state)
    4. Upload checkpoint metadata

### RocksDB Config

- `spark.sql.streaming.stateStore.rocksdb.blockCacheSizeMB` (default $8 MB$) - RocksDB block cache size
- `spark.sql.streaming.stateStore.rocksdb.writeBufferSizeMB` (default $64 MB$) - write buffer (memtable) size
- `spark.sql.streaming.stateStore.rocksdb.maxWriteBufferNumber` (default $2$) - number of write buffers
- `spark.sql.streaming.stateStore.rocksdb.compactOnCommit` (default `false`) - trigger RocksDB compaction on each commit; slower commits but faster reads

---

## State Partitioning

- __State partitioning__ - state for a stateful operator is partitioned across Spark tasks using the same partitioning as the shuffle that precedes the operator
- State partitioning must match shuffle partitioning - all rows with the same group key must go to the same partition where their state lives

### State Partition Count

- State partition count = `spark.sql.shuffle.partitions` at query start time (default $200$)
- FIXED for the lifetime of the query - cannot change without deleting checkpoint
- If `spark.sql.shuffle.partitions` changed after checkpoint exists - Spark uses checkpoint's partition count
- Changing state partition count requires full reprocessing from source

### State Locality

- Each executor holds state for a fixed set of partitions
- Tasks assigned to executors that hold their state partition (when possible)
- If executor dies → state loaded from HDFS checkpoint on replacement executor
- `StateStoreCoordinator` (Driver RPC endpoint) - tracks which executor holds which partition's state
- `StateStoreAwareZipPartitionsRDD` - schedules state tasks preferring executors with local state

### State Rebalancing

- No automatic state rebalancing when executors added/removed
- State always loaded from HDFS on new executor; locality lost on scale events
- For stable state locality - use static executor count for streaming queries

---

## State Schema Evolution

- __State schema evolution__ - changing the schema of state stored in `StateStore` between streaming query restarts

### Schema Compatibility Constraints

- State stored as binary `UnsafeRow` with schema baked in
- Schema changes that are compatible -
    - Adding nullable columns at the END of value schema (old state read; new column defaults to null)
    - Not supported natively; requires careful manual handling
- Schema changes that are INCOMPATIBLE -
    - Changing column types
    - Reordering columns
    - Adding columns in middle of schema
    - Removing columns

### State Schema Migration

- Spark 3.1+ supports state schema evolution for some operators with `--state-schema-migration` flag
- For complex schema changes - must drop checkpoint and reprocess from source
- `spark.sql.streaming.stateStore.stateSchemaCheck` (default `true`) - validates state schema matches checkpoint schema on restart; throws if incompatible

### Versioned State Schemas

- RocksDB state store supports keyed schema versioning (Spark 3.3+)
- Each key stores schema version alongside value
- Evolution rules checked per key on read
- Allows gradual migration - old state read with old schema; new state written with new schema

---

## Timer Internals (flatMapGroupsWithState)

- __Timers__ - a mechanism in `flatMapGroupsWithState` to schedule future callbacks at a specific time (event time or processing time); enables time-based state expiry, timeout logic, and periodic output

### Timer Types

- `GroupStateTimeout.EventTimeTimeout()` - fires when watermark advances past the registered timeout time
- `GroupStateTimeout.ProcessingTimeTimeout()` - fires when processing time passes the registered timeout time
- `GroupStateTimeout.NoTimeout()` - no timer support; `mapGroupsWithState` default

### Timer Storage

- Timers stored in a secondary `StateStore` alongside the group state
- Timer key - `(groupKey, timeoutTimestamp)`
- On each batch - `TimerStateManager` scans all timer entries; fires timers whose `timeoutTimestamp ≤ currentWatermark` (event time) or `≤ currentProcessingTime` (processing time)
- Fired timers call `func(key, Optional.empty(), state)` with no input rows

### GroupState Timer API

- Scala -
```scala
    def updateState(
        key: String,
        inputs: Iterator[Event],
        state: GroupState[MyState]
    ): Iterator[Output] = {
        if (state.hasTimedOut) {
            // Timer fired - no input rows
            val expired = state.get
            state.remove()
            Iterator(Output(key, expired, "TIMEOUT"))
        } else {
            // Normal update
            inputs.foreach(e => state.update(mergeState(state.getOption, e)))
            // Register timeout 10 minutes from now (event time)
            state.setTimeoutTimestamp(state.get.lastSeen + 10.minutes.toMillis)
            Iterator.empty
        }
    }
```

### Timer Firing Mechanics

- `FlatMapGroupsWithStateExec` processes data in two phases per batch -
    1. __Data processing phase__ - for each group key with new input data, call `func`; update state; register/clear timers
    2. __Timer firing phase__ - scan all timer entries; for keys with expired timers AND no input data in this batch, call `func` with empty iterator

---

## flatMapGroupsWithState Internals

- __`flatMapGroupsWithState`__ - the most powerful stateful streaming operator; allows arbitrary per-group state management and output; one function call per group per batch
- Available in Dataset API only (typed); not in DataFrame/SQL
- Supports both event time and processing time timeouts

### GroupState

- `GroupState[S]` - the state object passed to the user function; wraps the state store operations
- `state.exists` - whether state exists for this group
- `state.get` - get current state; throws if not exists
- `state.getOption` - `Option[S]`
- `state.update(newState: S)` - set/update state
- `state.remove()` - delete state for this group
- `state.hasTimedOut` - whether this call is due to timeout (no input rows)
- `state.setTimeoutTimestamp(ms)` - register event time timeout
- `state.setTimeoutDuration(ms)` / `setTimeoutDuration(duration)` - register processing time timeout
- `state.getCurrentWatermarkMs()` - current event time watermark
- `state.getCurrentProcessingTimeMs()` - current processing time

### FlatMapGroupsWithStateExec

- Physical operator wrapping the user state function
- Input - rows for this batch grouped by key
- State - loaded from `StateStore` per group key
- Output - whatever the user function emits
- State serialization - user type `S` serialized via `Encoder[S]` to `UnsafeRow` for `StateStore`

### Output Flexibility

- User function returns `Iterator[Output]` - can emit 0, 1, or many rows per group per batch
- Enables complex patterns -
    - Sessionization - emit one row per session when session closes
    - Deduplication - emit first occurrence; drop subsequent
    - Pattern detection - emit alert row when pattern completed
    - Aggregation with custom logic - any stateful computation

---

## mapGroupsWithState Internals

- __`mapGroupsWithState`__ - simpler version of `flatMapGroupsWithState`; emits exactly one output per group per batch; no timer support (always `NoTimeout`)
- Same internal mechanics as `flatMapGroupsWithState` but output wrapped - must emit exactly one row per group

### Difference from flatMapGroupsWithState

- `mapGroupsWithState` - `func: (key, Iterator[input], GroupState[S]) => output` - returns single output
- `flatMapGroupsWithState` - `func: (key, Iterator[input], GroupState[S]) => Iterator[output]` - returns iterator

### When to Use

- `mapGroupsWithState` - when you need stateful running aggregations and always want to emit a result every batch (eg - running count, latest value)
- `flatMapGroupsWithState` - when you need timers, variable output (0..N rows), or complex state machines

---

## Streaming Aggregations Internals

- __Streaming aggregations__ - `groupBy(...).agg(...)` applied to a streaming DataFrame; state stores partial aggregates; results emitted based on output mode and watermark

### Two-Phase Aggregation in Streaming

- __Partial aggregation (map side)__ - `HashAggregateExec` computes partial aggregates within each partition for new data in this batch
- __State merge (reduce side)__ - `StateStoreRestoreExec` loads existing state from `StateStore`; `StateStoreSaveExec` merges partial aggregates with existing state and saves updated state

### Physical Operators

- `StateStoreRestoreExec` -
    - For each key in new data - fetches existing partial aggregate from `StateStore`
    - Returns `(existingState, newData)` pairs to parent
- `StateStoreSaveExec` -
    - Receives merged aggregates from `HashAggregateExec` (full merge of existing + new)
    - Saves updated state back to `StateStore`
    - Based on output mode and watermark - decides which keys to emit and which to evict

### State Key and Value

- State key = groupBy columns as `UnsafeRow`
- State value = partial aggregate buffer as `UnsafeRow` - internal accumulator state (sum, count, min, max, etc.)
- Complete aggregates (final values) computed on-the-fly from partial buffer at output time

### Output Mode Impact on State

- __`Complete` mode__ -
    - All state retained; never evicted by watermark
    - Every batch emits ALL keys (full result table)
    - Memory grows unboundedly without watermark-based eviction
    - Used for totals, leaderboards where full result needed every batch

- __`Update` mode__ -
    - Only keys with updates in this batch emitted
    - With watermark - expired keys evicted after emission
    - Memory bounded when watermark used

- __`Append` mode__ -
    - REQUIRES watermark
    - Keys NOT emitted until watermark passes their window end
    - Once emitted, key evicted from state
    - Provides exactly-once output per key (key emitted once; then deleted)

---

## Output Modes Internals (Append, Update, Complete)

### Append Mode

- `StateStoreSaveExec` with `Append` output mode -
    - Each batch - scans all state store keys for current operator
    - Emits keys where `windowEnd <= currentWatermark` (for windowed aggregation) or state marked final
    - After emitting, removes the key from `StateStore`
    - Keys NOT yet past watermark → accumulated in state; NOT emitted
- Constraint - can only be used with queries where each row in the result table is finalized exactly once
- NOT valid for aggregations without watermark (would never emit; state grows unboundedly)

### Update Mode

- `StateStoreSaveExec` with `Update` output mode -
    - Each batch - emits ALL keys that received updates in this batch (not just watermark-expired ones)
    - State retained for future updates
    - Watermark still used for eviction - expired keys evicted but final value emitted before eviction
- Constraint - sink must handle updates (overwrite previous value for same key)
- Valid for aggregations with or without watermark

### Complete Mode

- `StateStoreSaveExec` with `Complete` output mode -
    - Each batch - emits ALL keys in state store (not just updated ones)
    - No eviction via watermark - ALL state retained
    - Memory unbounded without explicit state cleanup
- Used when downstream consumer needs complete current snapshot (eg - dashboard refresh)

### Output Mode Compatibility

| Operation | Append | Update | Complete |
| --- | --- | --- | --- |
| No aggregation | ✓ | ✓ | ✗ |
| Aggregation without watermark | ✗ | ✓ | ✓ |
| Aggregation with watermark | ✓ | ✓ | ✓ |
| `flatMapGroupsWithState` | ✓ | ✓ | ✗ |
| Stream-stream join | ✓ | ✗ | ✗ |

---

## Streaming Deduplication Internals

- __Streaming deduplication__ - `dropDuplicates("col1", "col2")` on a streaming DataFrame; emits each unique key combination exactly once; drops subsequent rows with same key

### DeduplicateExec

- Physical operator - `DeduplicateExec(keys, child)`
- State store key = deduplication key columns as `UnsafeRow`
- State store value = empty (only key existence matters)

### Per-Batch Logic

- For each row in new batch data -
    - Check if key exists in `StateStore`
    - If NOT exists → emit row; `put(key, emptyRow)` in state store
    - If exists → drop row (duplicate)

### Memory Management

- State store holds ALL seen keys forever - grows unboundedly without watermark
- With watermark on an event time column -
    - Keys older than watermark evicted from state store
    - Enables memory-bounded deduplication for time-partitioned data
```python
    df.withWatermark("event_time", "1 hour") \
      .dropDuplicates("user_id", "event_type")
    # Keys with event_time > watermark retained
    # Keys with event_time <= watermark evicted
```

### Exactly-Once with Deduplication

- Deduplication + exactly-once source + idempotent sink = end-to-end exactly-once
- Deduplication handles cases where source delivers same message multiple times (Kafka consumer group rebalance, source replay)

---

## Foreach & ForeachBatch Sink Internals

### ForeachSink

- `df.writeStream.foreach(writer: ForeachWriter[T])` - user-defined row-level sink
- `ForeachWriter[T]` interface -
    - `open(partitionId: Long, epochId: Long): Boolean` - called once per partition per batch; return `false` to skip this partition (already processed this epochId)
    - `process(value: T): Unit` - called for each row in partition
    - `close(errorOrNull: Throwable): Unit` - called after all rows processed or on error
- `ForeachDataWriter` - wraps `ForeachWriter`; implements DSv2 `DataWriter`
- `epochId` in `open` enables idempotency - if same `(partitionId, epochId)` already processed, return `false`

### ForeachBatch

- `df.writeStream.foreachBatch(func: (DataFrame, Long) => Unit)` - batch-level sink; most flexible
- Called with the complete output DataFrame for each micro-batch and the batch ID
- Runs on Driver - but `func` can distribute work by calling DataFrame actions on the batch
- Enables -
    - Writing to multiple sinks per batch
    - Using batch DataFrame APIs (arbitrary transformations before write)
    - Custom idempotency logic based on `batchId`

### ForeachBatch Exactly-Once

- Engine guarantees `func` called at most once per batch (may call twice on recovery → idempotency user's responsibility)
- `batchId` enables deduplication -
```python
    def write_idempotent(df, batch_id):
        # Delta Lake handles idempotency via transaction log
        df.write.format("delta").mode("append").save("/delta/events")

        # For non-idempotent sinks - manual check
        if not already_committed(batch_id):
            df.write.mode("append").parquet(f"/out/batch={batch_id}")
            mark_committed(batch_id)
```

### ForeachBatch State Recovery

- If `func` fails, batch is re-run with same `batchId`
- Engine does NOT retry partial failures within `func` - if `func` throws, entire batch retried from start
- Side effects in `func` before the throw may have occurred - must be idempotent

---

## Stream-Static Join Mechanics

- __Stream-static join__ - joining a streaming DataFrame with a batch (static) DataFrame; one side updates continuously; other side is a fixed dataset
- Most common use case - enriching stream data with a dimension table (lookup join)

### Execution Model

- Static side loaded as a regular batch read; broadcast or hash-joined as in batch
- For each micro-batch, the streaming side provides new rows; joined against static side
- Static side NOT re-read every batch by default -
    - For `spark.read.parquet(...)` static side - static side is read ONCE per micro-batch (new query execution)
    - For cached static side - static side read once at start; reused via cache

### Caching Static Side

- If static DataFrame is large and dimension data doesn't change frequently - cache it -
```python
    static_df = spark.read.parquet("hdfs://dimensions/").cache()
    static_df.count()   # force cache materialization

    streaming_df.join(static_df, "key") \
                .writeStream.format("parquet").start()
    # static_df read once from cache; not re-read every batch
```

### Static Side Freshness

- Problem - dimension table may be updated while streaming query runs
- Solution - `spark.sql.streaming.staticJoin.reuseStaticRelation=false` forces re-read every batch (expensive)
- Or restart query periodically to pick up dimension table updates

### State Management

- Stream-static join has NO state in `StateStore` - no state retention needed
- Static side is fully available every batch; no need to buffer rows for future matching
- Stream-static join does NOT require watermark

---

## Stream-Stream Join State Management

- __Stream-stream join__ - joining two streaming DataFrames; both sides are unbounded; Spark must buffer rows from each side until matching rows from the other side arrive

### Why State is Needed

- In batch - both sides fully available; join executes in one pass
- In streaming - row from stream A may arrive in batch 5; matching row from stream B may not arrive until batch 8
- Spark must BUFFER unmatched rows from both sides in `StateStore` until matches arrive or rows expire via watermark

### State Stores for Stream-Stream Join

- Two state stores per operator (one per side) -
    - `left_state_store` - buffered unmatched rows from left stream
    - `right_state_store` - buffered unmatched rows from right stream
- State key = join key columns
- State value = buffered rows with that join key (may be multiple rows per key)

### Symmetric Hash Join

- Both sides processed symmetrically -
    - New left row arrives → probe right state store; emit matches → if no match, buffer in left state store
    - New right row arrives → probe left state store; emit matches → if no match, buffer in right state store
- This is `SymmetricHashJoinExec` - different from batch `SortMergeJoin`

### State Eviction

- Without watermark - state grows unboundedly (all unmatched rows buffered forever)
- With watermark - rows older than watermark evicted from state stores
- State eviction conditions -
    - Left row evicted when `left.timestamp ≤ watermark` AND match window has closed
    - Right row evicted when `right.timestamp ≤ watermark` AND match window has closed

---

## Stream-Stream Join Watermark Interaction

- Stream-stream join REQUIRES watermarks on both sides for state eviction; without them state grows without bound

### Watermark Requirements

- Both streams must have watermarks defined via `withWatermark`
- Join condition must reference the timestamp columns used in watermarks
- Spark uses watermarks to determine when unmatched rows can safely be evicted

### How Watermark Bounds State

- For inner join on event time -
    - Left row with `timestamp = T` can safely be evicted when `currentWatermark ≥ T + matchWindow`
    - A right row with `timestamp ≥ T - matchWindow` could still arrive (within match window)
    - Once `watermark > T + matchWindow`, no right row can arrive that would match left row at time T → safe to evict
- Match window defined by join condition - `|left.time - right.time| <= threshold`

### Example

- Python -
```python
    left = spark.readStream.format("kafka").load() \
                .withWatermark("event_time", "1 hour")

    right = spark.readStream.format("kafka").load() \
                 .withWatermark("event_time", "1 hour")

    # Inner join with time constraint
    joined = left.join(
        right,
        expr("""
            left.key = right.key AND
            left.event_time BETWEEN right.event_time - interval 30 minutes
                                AND right.event_time + interval 30 minutes
        """)
    )
    # State evicted when: min(left_watermark, right_watermark) > timestamp + 30 min
```

### Watermark of Joined Stream

- Output watermark of joined stream = `min(left_watermark, right_watermark) - max_delay`
- Conservative - bounded by the slower watermark
- If left stream has intermittent data, left watermark stalls → right state also accumulates

---

## Interval Join

- __Interval join__ - a stream-stream join where the join condition constrains timestamps to an interval; `|left.time - right.time| <= delta`; enables bounded state via watermark

### Implementation

- Implemented via `SymmetricHashJoinExec` with time range condition
- State eviction uses the interval bounds to determine when rows can be safely discarded
- Eviction condition for left row at time `T` - `currentWatermark > T + intervalHigh` where `intervalHigh` is the upper bound of the time interval

### State Size Estimation

- State for interval join bounded by (approximately) -
    - `rows_per_second × max(intervalHigh, intervalLow) × 2` (both sides)
- With large intervals - state grows proportionally
- Tune interval size based on expected join latency; smaller = less state = less memory

### Inner vs Outer Interval Join

- Inner interval join - unmatched rows evicted without output
- Left outer interval join - unmatched left rows emitted with null right values when left row expires
    - Requires watermark on both sides
    - Output mode must be `Append`

---

## Symmetric Hash Join for Streaming

- __`SymmetricHashJoinExec`__ - the physical operator used for stream-stream joins; processes both streams symmetrically without requiring a sort

### Why Not SortMergeJoin

- `SortMergeJoin` requires knowing the full sorted input before merging - impossible for unbounded streams
- `SymmetricHashJoinExec` processes rows incrementally - one row at a time, probe the other side's state

### Symmetric Hash Join Algorithm

- For each new row from EITHER side -
    1. Probe the OTHER side's `StateStore` with the join key
    2. For each matching row in other side's state - emit a joined output row
    3. Buffer the new row in THIS side's `StateStore` (for future rows from other side)
    4. (After watermark advance) - evict expired rows from both state stores

### StateStore Structure for SymHashJoin

- State key = join key (equi-condition columns)
- State value = list of rows with that key (multiple rows per key possible)
- `UnsafeRowList` - Spark's memory-efficient list for storing multiple rows per state key

### Non-Equi Condition Handling

- Equi conditions (key equality) used for state store lookup
- Additional non-equi conditions (time interval, other predicates) applied as post-filter after state store probe
- State store lookup finds candidates by equi-key; post-filter eliminates non-matching candidates

### Memory Management

- `spark.sql.streaming.joinExec.stateFormatVersion` - controls state serialization format
- `spark.sql.streaming.stateStore.minDeltasForSnapshot` (default $10$) - threshold for triggering state snapshot
- Monitor `stateOperators[].numRowsTotal` in `StreamingQueryProgress` for state growth

> [!NOTE]
> Stream-stream joins with watermark produce state eviction in `Append` mode but the output watermark is min(left, right) - slowest source determines watermark. In production, if one Kafka topic has irregular data (sparse traffic), its watermark stalls, causing the other side's state to accumulate unboundedly. Always monitor `numRowsTotal` in state operator metrics and alert on unexpected growth.

> [!TIP]
> StateStore memory estimation for production sizing -
> - `HDFSBackedStateStore` - state must fit in executor heap; estimate `avg_state_row_bytes × max_keys_per_partition`; set executor memory accordingly
> - `RocksDBStateStore` - state on local disk; set `spark.local.dir` to fast SSD; `blockCacheSizeMB` controls memory for hot state
> - For stream-stream joins - `state_size ≈ input_rate × watermark_duration × avg_row_bytes × 2`; size state store capacity around this