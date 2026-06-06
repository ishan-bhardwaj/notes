# Structured Streaming

## Structured Streaming Overview & Execution Model

- __Structured Streaming__ - a fault-tolerant, scalable stream processing engine built on Spark SQL; treats a live data stream as an unbounded table that is continuously appended to; queries written as batch DataFrame/SQL queries execute incrementally
- Introduced in Spark 2.0; replaced DStream-based Spark Streaming
- Core abstraction - __streaming DataFrame/Dataset__; same API as batch; Spark handles incrementalization internally

### The Unbounded Table Model

- Incoming data appended to a conceptual unbounded input table
- Query defined over input table; result written to output table (sink)
- Each trigger interval - new rows arrive → query runs on new rows → result table updated → output written to sink
- User writes a batch-style query; Spark converts it to an incremental plan

### Execution Architecture

```
Source (Kafka, files, socket)
        ↓ new offsets each trigger
StreamExecution (Driver)
        ↓ incremental logical plan
Catalyst Optimizer
        ↓ physical plan
Spark SQL execution (batch job per trigger)
        ↓ results
Sink (Kafka, files, memory, foreach)
        ↓ offset committed to checkpoint
```

### Output Modes

- __`Append`__ - only new rows added to result table since last trigger written to sink; default for queries without aggregation; rows never updated once emitted
- __`Complete`__ - entire result table written to sink every trigger; only for aggregations; expensive for large result tables
- __`Update`__ - only rows that changed since last trigger written to sink; for aggregations; not supported by all sinks

### Streaming vs Batch Execution

- Each micro-batch = one Spark batch job on the incremental data
- Same Catalyst optimizer, same physical operators, same code generation
- Additional state management layer for stateful operations (aggregations, joins, deduplication)
- Streaming logical plans contain `StreamingRelation` nodes; batch plans have `LogicalRelation`

---

## StreamExecution

- __`StreamExecution`__ - the base class for all streaming query execution engines; manages the lifecycle of a running streaming query on the Driver
- One `StreamExecution` instance per running `StreamingQuery`
- Lives on the Driver; coordinates sources, sinks, checkpointing, and batch execution

### StreamExecution Responsibilities

- __Trigger management__ - determines when to run the next batch based on `Trigger` type
- __Offset management__ - fetches new offsets from sources; tracks which offsets have been processed and committed
- __Batch execution__ - constructs and executes the incremental Spark SQL query for each batch
- __Checkpoint coordination__ - writes offsets and commits to durable checkpoint storage before and after each batch
- __Fault recovery__ - on restart, reads checkpoint state and resumes from last committed offset
- __Query lifecycle__ - start, stop, awaitTermination, exception handling

### StreamExecution Internal Loop

```
while (streamActive) {
    1. Construct next batch (get new offsets from sources)
    2. Write new offsets to offset log (WAL)
    3. Execute batch (run Spark SQL job)
    4. Commit batch (write to commit log; advance source offsets)
    5. Sleep until next trigger
}
```

### Two Implementations

- `MicroBatchExecution` - processes data in discrete micro-batches; default mode
- `ContinuousExecution` - experimental; long-running tasks with epoch-based commits; millisecond latency

---

## MicroBatchExecution

- __`MicroBatchExecution`__ - the primary streaming execution engine; processes data in discrete micro-batches; each micro-batch is a complete Spark batch job

### Micro-Batch Lifecycle

1. __`constructNextBatch()`__ -
    - Calls `source.getOffset()` (v1) or `source.latestOffset()` (v2) to get new available offsets
    - Computes `startOffset` (last committed) and `endOffset` (new available)
    - If no new data and `Trigger` is not time-based → skip batch (return `false`)
    - Writes `endOffset` to `OffsetSeqLog` (WAL) before executing batch

2. __`runBatch()`__ -
    - Constructs incremental logical plan -
        - Replaces `StreamingRelation` with `StreamingExecutionRelation` bounded to `[startOffset, endOffset]`
        - For stateful ops - injects `StateStoreRestoreExec` / `StateStoreSaveExec`
    - Runs Catalyst optimization pipeline on incremental plan
    - Executes resulting Spark SQL job
    - Sink writes output rows

3. __`commitBatch()`__ -
    - Writes `endOffset` to `CommitLog` (marks batch as successfully completed)
    - Calls `source.commit(endOffset)` to allow source to discard processed data

### Incremental Plan Construction

- For a `filter → groupBy → count` streaming query -
    - Batch plan would aggregate all rows
    - Incremental plan - reads only new rows in `[startOffset, endOffset]`; merges partial aggregates with existing state in `StateStore`
    - `StreamingGlobalLimitExec`, `StateStoreSaveExec`, `StateStoreRestoreExec` - physical operators injected for stateful streaming

### Watermark Integration

- `withWatermark("timestamp", "10 minutes")` - defines event time watermark
- Watermark = max event time seen - delay threshold
- Data older than watermark considered late and dropped
- Watermark advances after each batch; old state cleared from `StateStore` when below watermark
- `EventTimeWatermark` logical node injected into plan; physical `EventTimeWatermarkExec` tracks max event time

---

## ContinuousExecution

- __`ContinuousExecution`__ - experimental execution mode for millisecond latency; long-running tasks that continuously read from source and write to sink; commits happen per epoch not per micro-batch
- Enabled via `Trigger.Continuous(checkpointInterval)`
- Available only for select sources (Kafka, rate source) and sinks

### Continuous Mode Architecture

- Long-running tasks started once per partition (one task per Kafka partition)
- Each task runs a tight loop - read record → process → write to sink → repeat
- No Spark job submitted per record - task runs indefinitely until stopped or epoch boundary
- __Epoch__ - a logical checkpoint interval; after each epoch, offsets committed to checkpoint

### ContinuousExecution vs MicroBatchExecution

| Aspect | MicroBatchExecution | ContinuousExecution |
| --- | --- | --- |
| Latency | $\geq 100 ms$ (one job per batch) | $\sim 1 ms$ |
| Throughput | High (batch processing) | Lower (per-record overhead) |
| Supported ops | Full Spark SQL | Select, filter, map only (no aggregation) |
| Fault tolerance | Batch-level | Epoch-level |
| Maturity | Stable, production | Experimental |

### Epoch Commits in Continuous Mode

- `EpochCoordinator` RPC endpoint on Driver - coordinates epoch boundaries across all tasks
- Tasks periodically report their current offset to `EpochCoordinator`
- `EpochCoordinator` determines when all tasks have passed an epoch boundary → commits that epoch
- Tasks continue processing without pausing for epoch commit

---

## Trigger Types & Trigger Internals

- __`Trigger`__ - controls when micro-batches are executed; different triggers for different latency/throughput trade-offs

### Trigger Types

- __`Trigger.ProcessingTime(interval)`__ -
    - Runs a new batch every `interval` (eg - `"10 seconds"`, `"1 minute"`)
    - If batch takes longer than interval - next batch starts immediately (no backpressure pause)
    - If batch finishes early - waits until interval expires
    - Most common trigger for production

- __`Trigger.Once()`__ -
    - Runs exactly one batch on all available data; then stops the streaming query
    - Processes all data accumulated since last checkpoint
    - Useful for incremental batch processing (run on a schedule; process new data; stop)
    - `Trigger.AvailableNow()` (Spark 3.3+) - similar but more granular; multiple batches until all available data processed

- __`Trigger.Continuous(checkpointInterval)`__ -
    - Continuous processing mode; long-running tasks; epoch-based commits
    - `checkpointInterval` - how often epochs are committed (eg - `"1 second"`)

- __`Trigger.AvailableNow()`__ (Spark 3.3+) -
    - Processes all available data in multiple batches (respects `maxOffsetsPerTrigger`)
    - More efficient than `Trigger.Once()` for large backlogs
    - Stops after all available data processed

### Trigger Internals

- `MicroBatchExecution.triggerExecutor` - wraps the batch loop with trigger timing logic
- `ProcessingTimeExecutor` - sleeps between batches to maintain interval; accounts for batch execution time
- `OnceExecutor` - executes one batch; sets `streamActive = false` afterward
- `ContinuousTriggerExecutor` - no sleep; drives continuous execution loop

### Python -
```python
from pyspark.sql.streaming import Trigger

# Fixed interval
query = df.writeStream \
    .trigger(processingTime="10 seconds") \
    .format("parquet") \
    .start()

# Run once (batch-style)
query = df.writeStream \
    .trigger(once=True) \
    .format("parquet") \
    .start()
query.awaitTermination()

# Available now (Spark 3.3+)
query = df.writeStream \
    .trigger(availableNow=True) \
    .format("parquet") \
    .start()

# Continuous (experimental)
query = df.writeStream \
    .trigger(continuous="1 second") \
    .format("kafka") \
    .start()
```

---

## Offset Management

- __Offset__ - a serializable position in a streaming source that uniquely identifies how much data has been consumed; the core bookkeeping primitive for streaming fault tolerance

### Offset Types

- `Offset` - abstract base class; source-specific implementations -
    - `KafkaSourceOffset` - `Map[TopicPartition, Long]`; offset per Kafka partition
    - `FileStreamSourceOffset` - file index (sequential integer) tracking which files have been processed
    - `LongOffset` - simple long integer; used by rate source, socket source
    - `SerializedOffset` - JSON string wrapper; used when offset stored in log

### Offset Lifecycle per Batch

- `startOffset` - the end offset of the last COMMITTED batch; where to start reading for this batch
- `endOffset` - the latest available offset from source; where to stop reading for this batch
- Both offsets stored in checkpoint before batch execution

### OffsetSeqLog

- `OffsetSeqLog` - append-only log of `OffsetSeq` objects; one entry per batch
- `OffsetSeq` - array of offsets, one per source (query may have multiple sources)
- Written to `$checkpointLocation/offsets/` as numbered files - `0`, `1`, `2`, ...
- Written BEFORE batch execution (write-ahead) - enables recovery of what batch was being processed if failure occurs during execution

### CommitLog

- `CommitLog` - append-only log of committed batch IDs
- Written AFTER successful batch execution and sink write
- Written to `$checkpointLocation/commits/` as numbered files
- On recovery - if batch ID is in both `OffsetSeqLog` AND `CommitLog` → batch was fully committed; advance past it
- If batch ID in `OffsetSeqLog` but NOT `CommitLog` → batch may have partially executed; re-execute from start offsets

---

## Epoch Commits

- __Epoch__ - in continuous processing mode, a logical boundary at which offsets are committed to checkpoint; allows fault recovery without replaying all data since stream start
- In micro-batch mode, "epoch" is effectively synonymous with "batch ID" - each batch is an epoch

### Epoch Commit in Continuous Mode

- `EpochCoordinator` - RPC endpoint on Driver; tracks epoch progress across all tasks
- Each task maintains a local `EpochTracker`
- At each epoch boundary (every `checkpointInterval`) -
    1. All tasks report current partition offsets to `EpochCoordinator`
    2. `EpochCoordinator` waits for all partitions to report
    3. Commits aggregate offsets to `OffsetSeqLog` and `CommitLog`
    4. Calls `sink.commit(epochId)` to allow sink to finalize epoch's output
- Tasks do NOT stop at epoch boundaries - they continue processing; commit is asynchronous

### Epoch and Exactly-Once

- Continuous mode guarantees exactly-once for supported sinks by -
    - Source records position at epoch boundary
    - Sink commits only complete epochs
    - On recovery - re-process from last committed epoch; sink idempotently handles potential re-sends

---

## Source Interface (v1 & v2)

### Source v1

- __`Source`__ trait - DSv1 streaming source interface
- Methods -
    - `schema: StructType` - schema of data produced
    - `getOffset: Option[Offset]` - returns latest available offset; called at start of each batch
    - `getBatch(start: Option[Offset], end: Offset): DataFrame` - returns batch of data in `[start, end]` range
    - `commit(end: Offset): Unit` - called after batch committed; source can discard processed data
    - `stop(): Unit` - called when query stopped; release resources

### Source v2 (MicroBatchStream)

- __`MicroBatchStream`__ - DSv2 interface for micro-batch streaming sources
- Methods -
    - `initialOffset(): Offset` - starting offset for a new query (no checkpoint)
    - `latestOffset(start: Offset, readLimit: ReadLimit): Offset` - latest available offset; `ReadLimit` controls max data per batch (`ReadLimit.maxRows(n)`, `ReadLimit.maxFiles(n)`)
    - `planInputPartitions(start: Offset, end: Offset): Array[InputPartition]` - partitions to read for this batch
    - `createReaderFactory(): PartitionReaderFactory` - factory for reading each partition
    - `commit(end: Offset): Unit` - batch committed; source cleanup
    - `stop(): Unit`
    - `deserializeOffset(json: String): Offset` - reconstruct offset from checkpoint JSON

### SparkDataStream (v2 continuous)

- __`ContinuousStream`__ - DSv2 interface for continuous processing sources
- `planInputPartitions(start: Offset): Array[ContinuousInputPartition]` - long-running partitions
- `mergeOffsets(offsets: PartitionOffset[]): Offset` - combines per-partition offsets into single offset

---

## Sink Interface (v1 & v2)

### Sink v1

- __`Sink`__ trait - DSv1 streaming sink interface
- Single method - `addBatch(batchId: Long, data: DataFrame): Unit` -
    - Called by `MicroBatchExecution` after each batch execution
    - `batchId` enables idempotency - if same `batchId` called twice (retry), sink must handle gracefully
    - `data` - the batch output DataFrame; sink writes it to destination

### ForeachBatchSink

- `df.writeStream.foreachBatch(func)` - user-provided function called with each batch DataFrame and batch ID
- Enables arbitrary batch operations on each micro-batch
- Python -
```python
    def process_batch(df, batch_id):
        # Can use any batch DataFrame operations
        df.write.mode("append").parquet(f"hdfs://output/batch_{batch_id}")
        # Or multiple sinks
        df.write.format("delta").save("hdfs://delta/events")

    query = streaming_df.writeStream \
        .foreachBatch(process_batch) \
        .start()
```

### Sink v2 (StreamingWrite)

- __`WriteBuilder`__ - DSv2 write builder; returns `StreamingWrite` from `buildForStreaming()`
- __`StreamingWrite`__ - `commit(epochId, messages)` / `abort(epochId, messages)` / `createStreamingWriterFactory()`
- `StreamingDataWriterFactory` - creates `DataWriter[InternalRow]` per partition per batch
- `DataWriter` - `write(row)`, `commit()`, `abort()` - fine-grained per-partition write control

### ForeachSink

- `df.writeStream.foreach(ForeachWriter)` - user-defined per-row sink
- `ForeachWriter[T]` - `open(partitionId, epochId): Boolean`, `process(value: T): Unit`, `close(error: Throwable): Unit`
- `open` returns `false` to skip this partition (idempotency - if already committed this epochId)
- Less efficient than `foreachBatch` - called per row; no batch optimization

---

## Streaming Checkpointing

- __Checkpoint__ - durable storage of streaming query state that enables fault recovery and exactly-once processing; stored at `checkpointLocation` on a reliable filesystem (HDFS, S3, GCS)

### Checkpoint Directory Structure

```
checkpointLocation/
├── metadata              ← query metadata (id, runId, description)
├── offsets/              ← OffsetSeqLog - one file per batch
│   ├── 0
│   ├── 1
│   └── 2
├── commits/              ← CommitLog - one file per committed batch
│   ├── 0
│   └── 1
├── sources/              ← source-specific checkpoint data
│   └── 0/               ← source 0 checkpoint
│       └── ...
└── state/                ← StateStore data for stateful operations
└── 0/               ← operator 0 state
├── 0/           ← partition 0
│   ├── 1.delta
│   └── 2.delta
└── 1/           ← partition 1
```

### Checkpoint on Restart

1. `StreamExecution.start()` reads `metadata` file - validates query ID matches
2. Reads `OffsetSeqLog` and `CommitLog` -
    - Last batch in `CommitLog` = last fully committed batch; set `committedOffsets` to its end offsets
    - Last batch in `OffsetSeqLog` but NOT in `CommitLog` = interrupted batch; set `pendingOffsets` to re-execute
3. Source `initialOffset` ignored; checkpoint offsets used instead
4. If interrupted batch exists - re-execute it (source provides same data for same offset range; sink must handle idempotent re-delivery)
5. Continue from there

### Checkpoint Compaction

- Offset and commit log files accumulate over time
- `OffsetSeqLog` compacts every $10$ batches by default - older entries merged
- `spark.sql.streaming.minBatchesToRetain` (default $100$) - minimum batches retained in logs
- StateStore has its own compaction mechanism - delta files merged into snapshots periodically

---

## WAL (Write-Ahead Log)

- __Write-Ahead Log (WAL)__ - in streaming context, the `OffsetSeqLog`; records end offsets BEFORE batch execution; ensures recovery knowledge of what was being processed

### WAL Semantics

- WAL written in `constructNextBatch()` BEFORE `runBatch()` executes
- If failure occurs DURING batch execution -
    - WAL has the offsets for the in-progress batch
    - On restart, `CommitLog` has no entry for this batch
    - StreamExecution re-executes the batch from WAL offsets
    - Source provides the same data (offset-bounded replay)
    - Sink must be idempotent (same `batchId` may be written twice)
- If failure occurs AFTER batch but BEFORE commit log write -
    - Same recovery path - batch re-executed
    - Same `batchId` presented to sink again

### WAL File Format

- Each WAL entry is a JSON-serialized `OffsetSeq` -
```json
    v1
    {"batchWatermarkMs":0,"batchTimestampMs":1700000000000,"conf":{}}
    {"kafka":{"topic1":{"0":1000,"1":2000}}}
```
- First line - version header
- Second line - metadata (watermark, timestamp, config)
- Remaining lines - one JSON offset per source

---

## Offset Log & Commit Log

### OffsetSeqLog

- `HDFSMetadataLog[OffsetSeq]` - append-only log backed by HDFS/S3
- Each entry stored as a numbered file in `checkpointLocation/offsets/`
- `add(batchId, metadata)` - writes new entry; atomic rename (temp file → final file)
- `get(batchId)` - reads specific batch entry
- `getLatest()` - reads most recent entry (highest numbered file)
- `purge(thresholdBatchId)` - removes entries older than threshold

### CommitLog

- `CommitLog` extends `HDFSMetadataLog[CommitMetadata]`
- Entry stored in `checkpointLocation/commits/`
- `CommitMetadata` contains -
    - `nextBatchWatermarkMs` - watermark value at commit time; propagated to next batch
- Written AFTER successful sink write and before `source.commit()`
- Recovery logic -
    - Latest commit ID = last fully processed batch
    - If `OffsetSeqLog` has entries beyond latest commit → those batches need re-execution

### Log Compaction

- Both logs compact old entries to bounded storage -
    - Every `spark.sql.streaming.minBatchesToRetain` (default $100$) batches, old entries eligible for deletion
    - Compaction writes a snapshot file containing the retained range; old individual files deleted
    - Enables long-running queries without unbounded checkpoint storage growth

---

## Exactly-Once Semantics — Source Contract

- __Source exactly-once contract__ - a streaming source must be able to replay any range of data given `[startOffset, endOffset]` deterministically; same data returned for same offset range regardless of how many times `getBatch` is called

### Source Replay Guarantee

- `getBatch(start, end)` must return identical data for the same `(start, end)` regardless of retries
- Source must NOT advance its internal read pointer in `getBatch` - only in `commit(end)`
- `commit(end)` called after batch successfully committed - source may now discard data up to `end`
- If `commit` never called (failure before commit) - source retains data; replay available on next `getBatch`

### Source Implementations

- __Kafka__ - offsets are explicit and replayable; `getBatch(start, end)` reads exactly `[start, end]` offset range from Kafka; old messages retained until Kafka retention policy (not Spark commit)
- __File source__ - file list is immutable; `getBatch` reads specific set of files; files not deleted by Spark; exactly replayable
- __Rate source / socket__ - do NOT support true replay; only usable for testing; not production exactly-once

### Offset Ordering Guarantee

- Source offsets must be monotonically increasing - `endOffset` of batch $N$ = `startOffset` of batch $N+1$
- No gaps - every record in source reachable via some `[start, end]` range
- No reordering - records within an offset range always returned in same order

---

## Exactly-Once Semantics — Sink Contract

- __Sink exactly-once contract__ - a sink must be idempotent with respect to `batchId`; writing the same `batchId` twice must produce the same result as writing it once

### Sink Idempotency

- `Sink.addBatch(batchId, data)` may be called with the same `batchId` twice (if failure between batch execution and commit log write)
- Sink must detect and handle duplicate batch IDs -
    - Transactional sinks - use `batchId` as transaction ID; second call is a no-op if transaction already committed
    - File sinks - write to temp location with `batchId` in name; atomic rename on success; second call detects existing target
    - Kafka sink - uses producer transactions with `batchId` as transaction key

### File Sink Exactly-Once

- `FileStreamSink` - writes output files to a staging directory; maintains a log of committed files
- `FileStreamSinkLog` - list of files committed per batch; stored in `checkpointLocation/sinks/`
- On duplicate `batchId` - checks `FileStreamSinkLog`; if already present, skips write
- Atomic commit - files moved from staging to final location; `FileStreamSinkLog` updated

### Kafka Sink Exactly-Once

- `KafkaStreamingWrite` uses Kafka's idempotent producer + transactions -
    - `transactionalId` derived from `batchId` and partition ID
    - Each batch written in a Kafka transaction; committed atomically
    - Duplicate `batchId` → same `transactionalId` → Kafka deduplicates via sequence numbers

### ForeachBatch Exactly-Once

- User is responsible for idempotency in `foreachBatch`
- `batchId` parameter enables idempotency -
```python
    def process_batch(df, batch_id):
        # Check if batch already processed
        if not is_processed(batch_id):
            df.write.mode("append").parquet(output_path)
            mark_processed(batch_id)   # external idempotency marker
```

---

## Streaming Query Progress & Metrics

- __`StreamingQueryProgress`__ - the metrics snapshot for one micro-batch; accessible via `StreamingQuery.lastProgress` or `StreamingQuery.recentProgress`

### Key Metrics

- `id` - unique query UUID; stable across restarts
- `runId` - unique UUID for this query RUN; changes on each restart
- `batchId` - current batch number (0-based)
- `timestamp` - batch start time (ISO 8601)
- `batchDuration` - total batch wall-clock time in ms
- `numInputRows` - total rows read across all sources in this batch
- `inputRowsPerSecond` - input rate (rows / trigger interval)
- `processedRowsPerSecond` - processing throughput (rows / batch duration)

### Source Metrics

- `sources[]` - per-source metrics -
    - `description` - source description (eg - `KafkaV2[Subscribe[events]]`)
    - `startOffset` / `endOffset` - offset range for this batch
    - `latestOffset` - latest available offset at batch start
    - `numInputRows` - rows read from this source
    - `inputRowsPerSecond` / `processedRowsPerSecond`

### Sink Metrics

- `sink` - sink description
- No row count reported for sink (sink writes are opaque to engine)

### State Metrics

- `stateOperators[]` - per stateful operator metrics -
    - `operatorName` - eg `stateStoreSave`, `deduplicate`, `symmetricHashJoin`
    - `numRowsTotal` - total rows currently in state store
    - `numRowsUpdated` - rows updated in this batch
    - `numRowsDroppedByWatermark` - rows evicted due to watermark advancement
    - `memoryUsedBytes` - state store memory consumption
    - `customMetrics` - operator-specific metrics (eg - RocksDB metrics)

### Accessing Progress

- Python -
```python
    query = df.writeStream.format("console").start()

    # Last batch progress
    progress = query.lastProgress
    print(progress["batchId"])
    print(progress["numInputRows"])
    print(progress["processedRowsPerSecond"])

    # Recent history (default last 100 batches)
    for p in query.recentProgress:
        print(p["batchId"], p["batchDuration"])

    # Listen for progress events
    class MyListener(StreamingQueryListener):
        def onQueryStarted(self, event): pass
        def onQueryProgress(self, event):
            print(event.progress.prettyJson)
        def onQueryTerminated(self, event): pass

    spark.streams.addListener(MyListener())
```

### StreamingQueryListener

- `StreamingQueryListener` - callback interface for streaming query events
- `onQueryStarted(event: QueryStartedEvent)` - query started
- `onQueryProgress(event: QueryProgressEvent)` - batch completed; contains `StreamingQueryProgress`
- `onQueryTerminated(event: QueryTerminatedEvent)` - query stopped (with or without error)
- Register via `spark.streams.addListener(listener)`; unregister via `spark.streams.removeListener(listener)`
- Useful for monitoring, alerting, lag tracking in production systems

---

## StreamingQueryManager

- __`StreamingQueryManager`__ - manages all active streaming queries in a `SparkSession`; accessible via `spark.streams`

### Core Operations

- `spark.streams.active` - returns array of all active `StreamingQuery` objects in this session
- `spark.streams.get(id)` - returns specific `StreamingQuery` by UUID
- `spark.streams.awaitAnyTermination()` - blocks until any streaming query terminates (cleanly or with exception)
- `spark.streams.awaitAnyTermination(timeout)` - same with timeout in milliseconds; returns `true` if terminated, `false` if timed out
- `spark.streams.resetTerminated()` - resets termination state; allows `awaitAnyTermination` to block again after a query terminated
- `spark.streams.addListener(listener)` / `removeListener(listener)` - manage `StreamingQueryListener`s

### StreamingQuery Interface

- `query.id` - stable UUID across restarts
- `query.runId` - UUID for this run; changes on restart
- `query.name` - user-specified name (from `.queryName("myQuery")`)
- `query.isActive` - whether query is currently running
- `query.status` - `StreamingQueryStatus` with current state description
- `query.lastProgress` / `query.recentProgress` - metrics
- `query.exception` - `StreamingQueryException` if query terminated with error
- `query.awaitTermination()` - block until this query terminates
- `query.awaitTermination(timeout)` - block with timeout
- `query.stop()` - gracefully stop the query; waits for current batch to complete

### Multi-Query Management Pattern

- Python -
```python
    # Start multiple queries
    q1 = df1.writeStream.format("parquet").option("path", "/out1") \
            .option("checkpointLocation", "/ckpt1").start()
    q2 = df2.writeStream.format("kafka") \
            .option("kafka.bootstrap.servers", "broker:9092") \
            .option("topic", "output") \
            .option("checkpointLocation", "/ckpt2").start()

    # Monitor all queries
    try:
        spark.streams.awaitAnyTermination()
    except Exception as e:
        # One query failed - check which one
        for q in spark.streams.active:
            if q.exception:
                print(f"Query {q.name} failed: {q.exception}")
        raise
    finally:
        # Stop all remaining queries
        for q in spark.streams.active:
            q.stop()
```

### Query Restart and Checkpoint Compatibility

- Same `checkpointLocation` + same query definition = resume from checkpoint
- Schema changes, source/sink changes, or operator changes may be incompatible with existing checkpoint
- `spark.sql.streaming.schemaInference=true` - allow schema changes (use with caution)
- Breaking changes require deleting checkpoint and reprocessing from beginning (or earliest source offset)

> [!NOTE]
> The relationship between `OffsetSeqLog` and `CommitLog` is the core of Structured Streaming's fault tolerance. An entry in `OffsetSeqLog` with no corresponding `CommitLog` entry means a batch was started but not committed - this batch will be re-executed on restart. Sources must replay the same data; sinks must handle the same `batchId` idempotently. Breaking either contract breaks exactly-once semantics.

> [!TIP]
> Production streaming query checklist -
> - Set `checkpointLocation` on reliable storage (HDFS, S3 with strong consistency) - never local filesystem
> - Use `ProcessingTime` trigger with interval tuned to your latency requirements
> - Monitor `processedRowsPerSecond < inputRowsPerSecond` - indicates query falling behind; scale up executors
> - Set `spark.sql.streaming.minBatchesToRetain` to retain enough history for debugging
> - Use `foreachBatch` with idempotency checks for non-transactional sinks
> - Test recovery by killing the query mid-batch and verifying exactly-once output