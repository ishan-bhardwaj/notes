# Observability & Debugging

## Spark Metrics System

- __Spark Metrics System__ - a pluggable metrics collection framework built on Dropwizard Metrics (formerly Coda Hale Metrics); collects runtime metrics from every Spark component and exports them to external monitoring systems
- Configured via `$SPARK_HOME/conf/metrics.properties`; one configuration file per Spark process
- Architecture - metrics flow from `MetricsSource` (producers) → `MetricsRegistry` → `MetricsSink` (exporters)

### Metrics Namespacing

- Every metric named as `{appId}.{executorId}.{source}.{metricName}` -
    - `appId` - `local-1700000000` or `application_1700000000_0001`
    - `executorId` - `driver`, `0`, `1`, `2`, ...
    - `source` - `executor`, `jvm`, `CodeGenerator`, `DAGScheduler`, etc.
    - `metricName` - `shuffleReadBytes`, `heapUsed`, `activeTasks`, etc.
- Full metric name example - `application_1700000000_0001.1.executor.shuffleReadBytes`

### Metric Types (Dropwizard)

- `Counter` - monotonically increasing integer; eg - `totalShuffleRead`
- `Gauge` - current instantaneous value; eg - `heapUsed`, `activeTasks`
- `Histogram` - statistical distribution of values; tracks min/max/mean/percentiles
- `Meter` - rate of events over time (1min/5min/15min EWMA); eg - `taskCompletionRate`
- `Timer` - histogram + meter; tracks duration and rate of timed events

### Configuration

```properties
# metrics.properties
*.sink.graphite.class=org.apache.spark.metrics.sink.GraphiteSink
*.sink.graphite.host=graphite.example.com
*.sink.graphite.port=2003
*.sink.graphite.period=10
*.sink.graphite.unit=seconds
*.sink.graphite.prefix=spark

# Prometheus pushgateway
*.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
*.sink.prometheusServlet.path=/metrics/prometheus

# Enable JVM metrics
*.source.jvm.class=org.apache.spark.metrics.source.JvmSource
```

---

## MetricsSource & Sinks (Graphite, Prometheus)

### MetricsSource

- __`MetricsSource`__ - interface for components that produce metrics; register metrics with `MetricsSystem`
- Implementations -
    - `ExecutorSource` - task counts, shuffle read/write bytes, JVM memory
    - `DAGSchedulerSource` - job/stage counts, failed stages
    - `BlockManagerSource` - memory used/free, disk used, cached blocks
    - `JvmSource` - heap/non-heap memory, GC counts/times, thread counts
    - `CodeGenerator` - generated classes, compilation time
    - `StreamingQueryManagerSource` - active streaming queries
    - `AccumulatorSource` - user-defined accumulator values

### Built-in Sinks

- __`GraphiteSink`__ - sends metrics to Graphite via plaintext TCP protocol; best for time-series dashboards
- __`PrometheusSink` / `PrometheusServlet`__ - exposes metrics via HTTP endpoint; Prometheus scrapes; most common in modern deployments
- __`Slf4jSink`__ - logs metrics via SLF4J; useful for debugging; pollutes logs
- __`CSVSink`__ - writes metrics to CSV files at configured intervals; useful for offline analysis
- __`ConsoleSink`__ - prints to stdout; testing only
- __`StatsdSink`__ - sends to StatsD agent; common in AWS/GCP monitoring stacks

### Prometheus Integration

```properties
# metrics.properties - expose Prometheus endpoint on each executor and driver
*.sink.prometheusServlet.class=org.apache.spark.metrics.sink.PrometheusServlet
*.sink.prometheusServlet.path=/metrics/prometheus
driver.sink.prometheusServlet.path=/metrics/driver/prometheus
```

```python
# Spark config for Prometheus pushgateway (alternative)
spark = SparkSession.builder \
    .config("spark.ui.prometheus.enabled", "true") \
    .getOrCreate()

# Access metrics endpoint
# http://driver-host:4040/metrics/prometheus
# http://executor-host:executor-port/metrics/prometheus
```

### Custom MetricsSource

- Scala -
```scala
    import com.codahale.metrics.{Gauge, MetricRegistry}
    import org.apache.spark.metrics.source.Source

    class MyAppMetrics extends Source {
        override val sourceName: String = "MyApp"
        override val metricRegistry: MetricRegistry = new MetricRegistry

        // Register custom gauge
        val processedRecords = metricRegistry.counter("processedRecords")
        val processingLatency = metricRegistry.histogram("processingLatencyMs")

        def recordProcessed(latencyMs: Long): Unit = {
            processedRecords.inc()
            processingLatency.update(latencyMs)
        }
    }

    // Register with SparkContext
    val myMetrics = new MyAppMetrics
    SparkEnv.get.metricsSystem.registerSource(myMetrics)
```

---

## Task Metrics Internals

- __`TaskMetrics`__ - the primary data structure tracking per-task execution statistics; collected by every task; aggregated by `DAGScheduler` and exposed in Spark UI

### TaskMetrics Fields

- __Input metrics__ (`InputMetrics`) -
    - `bytesRead: Long` - bytes read from input source (HDFS, S3, etc.)
    - `recordsRead: Long` - records read from input source

- __Output metrics__ (`OutputMetrics`) -
    - `bytesWritten: Long` - bytes written to output sink
    - `recordsWritten: Long` - records written to output sink

- __Shuffle read metrics__ (`ShuffleReadMetrics`) -
    - `remoteBlocksFetched: Long` - shuffle blocks fetched from remote executors
    - `localBlocksFetched: Long` - shuffle blocks fetched from local executor
    - `fetchWaitTime: Long` - milliseconds spent waiting for remote shuffle data
    - `remoteBytesRead: Long` - bytes fetched from remote executors
    - `localBytesRead: Long` - bytes fetched from local executor
    - `recordsRead: Long` - shuffle records read

- __Shuffle write metrics__ (`ShuffleWriteMetrics`) -
    - `bytesWritten: Long` - bytes written to shuffle files
    - `recordsWritten: Long` - records written to shuffle files
    - `writeTime: Long` - nanoseconds spent writing shuffle data

- __Spill metrics__ -
    - `memoryBytesSpilled: Long` - bytes spilled from memory (before compression)
    - `diskBytesSpilled: Long` - bytes written to disk during spill (after compression)

- __Executor metrics__ -
    - `executorRunTime: Long` - milliseconds of actual task execution (excludes scheduling overhead)
    - `executorDeserializeTime: Long` - milliseconds to deserialize task closure
    - `resultSerializationTime: Long` - milliseconds to serialize task result
    - `jvmGCTime: Long` - milliseconds spent in JVM GC during task execution
    - `peakExecutionMemory: Long` - peak execution memory bytes used by task

### TaskMetrics Collection Flow

1. `TaskRunner.run()` creates fresh `TaskMetrics` at task start
2. Task execution - physical operators increment metrics via `TaskContext.taskMetrics()`
3. On task completion - `TaskRunner` serializes `TaskMetrics` into `TaskResult`
4. `TaskSchedulerImpl` deserializes result; calls `DAGScheduler.handleTaskCompletion`
5. `DAGScheduler` calls `accumulatorUpdates.foreach(acc => acc.merge(update))` - merges accumulator-backed metrics
6. Aggregated metrics stored in `StageInfo.taskMetrics` and exposed via Spark UI

---

## Executor Metrics

- __Executor metrics__ - JVM-level and OS-level metrics collected per executor; reported via heartbeat; distinct from per-task `TaskMetrics`

### ExecutorMetrics Fields

- `JVMHeapMemory` - current JVM heap used bytes
- `JVMOffHeapMemory` - current JVM off-heap used bytes (direct buffers, Tungsten)
- `OnHeapExecutionMemory` - execution memory used on heap
- `OffHeapExecutionMemory` - execution memory used off heap
- `OnHeapStorageMemory` - storage memory used on heap
- `OffHeapStorageMemory` - storage memory used off heap
- `OnHeapUnifiedMemory` - total unified memory on heap
- `OffHeapUnifiedMemory` - total unified memory off heap
- `DirectPoolMemory` - JVM direct buffer memory
- `MappedPoolMemory` - memory-mapped file bytes
- `MinorGCCount` / `MajorGCCount` - GC event counts
- `MinorGCTime` / `MajorGCTime` - cumulative GC time in ms
- `ProcessTreeJVMVMemory` - virtual memory of JVM process tree
- `ProcessTreeJVMRSSMemory` - resident set size of JVM process tree

### Heartbeat Reporting

- Executors send `Heartbeat(executorId, accumUpdates, blockManagerId, executorMetrics)` to Driver every `spark.executor.heartbeatInterval` (default $10s$)
- `HeartbeatReceiver` on Driver updates executor metrics; stores in `ExecutorMetricsPoller`
- Peak executor metrics computed per stage - max value seen for each metric across all heartbeats during stage execution
- Peak metrics visible in Spark UI → Stages → Peak Values column

---

## Event Log Internals

- __Event Log__ - a JSON-serialized log of all Spark events during application execution; written to `spark.eventLog.dir`; consumed by Spark History Server for post-mortem analysis

### Enabling Event Logging

```python
spark = SparkSession.builder \
    .config("spark.eventLog.enabled", "true") \
    .config("spark.eventLog.dir", "hdfs://namenode/spark-logs") \
    .config("spark.eventLog.compress", "true") \
    .config("spark.eventLog.compression.codec", "zstd") \
    .getOrCreate()
```

### Event Types

- `SparkListenerApplicationStart` / `SparkListenerApplicationEnd`
- `SparkListenerJobStart` / `SparkListenerJobEnd`
- `SparkListenerStageSubmitted` / `SparkListenerStageCompleted`
- `SparkListenerTaskStart` / `SparkListenerTaskEnd` - includes `TaskMetrics`
- `SparkListenerExecutorAdded` / `SparkListenerExecutorRemoved`
- `SparkListenerBlockManagerAdded` / `SparkListenerBlockManagerRemoved`
- `SparkListenerEnvironmentUpdate` - configuration at startup
- `SparkListenerBlockUpdated` - cached block changes
- `SparkListenerSQLExecutionStart` / `SparkListenerSQLExecutionEnd` - SQL query events with plan

### Event Log Format

```json
{"Event":"SparkListenerTaskEnd","Stage ID":0,"Stage Attempt ID":0,
 "Task Type":"ResultTask","Task End Reason":{"Reason":"Success"},
 "Task Info":{"Task ID":42,"Index":0,"Attempt":0,...},
 "Task Executor Metrics":{"JVMHeapMemory":1234567890,...},
 "Task Metrics":{"Executor Run Time":1234,"JVM GC Time":56,
   "Shuffle Read Metrics":{"Remote Bytes Read":1048576,...}}}
```

### Event Log Rolling

- `spark.eventLog.rolling.enabled` (default `false`) - roll event log files to prevent single huge file
- `spark.eventLog.rolling.maxFileSize` (default $128 MB$) - roll when log exceeds this size
- Essential for long-running applications (Structured Streaming queries running for weeks)

---

## Spark History Server

- __Spark History Server__ - a standalone web service that reads event logs from HDFS/S3 and serves the Spark UI for completed (or still-running) applications
- Runs independently of Spark applications; started via `$SPARK_HOME/sbin/start-history-server.sh`

### Configuration

```properties
# spark-defaults.conf for History Server
spark.history.ui.port=18080
spark.history.fs.logDirectory=hdfs://namenode/spark-logs
spark.history.retainedApplications=100       # in-memory app cache size
spark.history.fs.cleaner.enabled=true        # auto-clean old logs
spark.history.fs.cleaner.maxAge=7d           # keep 7 days of logs
spark.history.fs.update.interval=10s         # poll for new events
```

### Incomplete Application Handling

- For running applications - History Server shows partial event log (updated periodically)
- `spark.eventLog.logStageExecutorMetrics=true` - log executor metrics to event log (needed for full History Server display)
- `spark.eventLog.logBlockUpdates.enabled=false` (default) - set `true` to log every block update (very verbose; avoid for long jobs)

### History Server REST API

```bash
# List applications
curl http://history-server:18080/api/v1/applications

# Get stages for an app
curl http://history-server:18080/api/v1/applications/{appId}/stages

# Get tasks for a stage
curl http://history-server:18080/api/v1/applications/{appId}/stages/{stageId}/{attemptId}/taskList

# Get SQL execution plan
curl http://history-server:18080/api/v1/applications/{appId}/sql
```

---

## Spark UI — Jobs & Stages Tab

- __Jobs tab__ - top-level view; one row per Spark job (triggered by one action); shows duration, stages, tasks succeeded/failed/skipped
- __Stages tab__ - all stages across all jobs; each stage shows task count, duration, shuffle read/write bytes, spill

### Reading the Jobs Tab

- __Duration__ - wall-clock time from job submission to completion; includes scheduling overhead
- __Stages__ - `X/Y` format where X = completed stages, Y = total stages; failed stages shown in red
- __Tasks__ - `X/Y` completed/total; failed tasks shown separately
- __Skipped stages__ - stages whose output is already cached or computed by a previous job; shown as skipped; no re-computation

### Reading the Stages Tab

- __Input__ - bytes read from source (HDFS, S3, Kafka, etc.)
- __Output__ - bytes written to sink
- __Shuffle Read__ - bytes fetched from shuffle files (reduce side)
- __Shuffle Write__ - bytes written to shuffle files (map side)
- __Spill (Memory)__ - bytes spilled from memory before compression
- __Spill (Disk)__ - bytes written to disk after compression
- __GC Time__ - sum of JVM GC time across all tasks
- __Duration__ - wall-clock time for the stage

### Stage DAG Visualization

- Each stage shows its DAG of operators (collapsed by default)
- "Show Additional Metrics" - reveals per-task timing breakdown: scheduler delay, task deserialization time, result serialization time, getting result time

### Key Warning Signs in Jobs Tab

- Stage duration $10×$ longer than sibling stages → skew or GC issue
- Many failed task attempts → executor instability or OOM
- Skipped stages not expected → check if stale cache is being reused incorrectly
- `Shuffle Write >> Shuffle Read` → many-to-few shuffle; potential skew in reduce side

---

## Spark UI — Tasks Tab

- __Tasks tab__ - per-stage drill-down; one row per task; the most granular view of execution; essential for diagnosing skew, GC, and stragglers

### Key Task Columns

- __Duration__ - wall-clock time for the task; outliers indicate skew or GC
- __GC Time__ - JVM GC time during task; `> 10%` of duration indicates memory pressure
- __Shuffle Read Size__ - bytes this task fetched from shuffle; large outlier = skew
- __Shuffle Write Size__ - bytes this task wrote to shuffle; large outlier = map-side skew
- __Spill (Memory/Disk)__ - spill indicates task exceeded execution memory
- __Input Size / Records__ - for source scans; large outlier = uneven input splitting
- __Errors__ - exception message for failed tasks; first error visible inline

### Tasks Tab Sorting

- Sort by `Duration DESC` - find slowest tasks; compare to median
- Sort by `Shuffle Read Size DESC` - find tasks reading most shuffle data (reduce-side skew)
- Sort by `GC Time DESC` - find tasks with most GC pressure
- Sort by `Spill (Memory) DESC` - find tasks exceeding memory budget

### Task Status Distribution

- `Summary Metrics` section shows P25/median/P75/max for all metrics across tasks
- `max / median > 5×` for Duration or Shuffle Read → significant skew present
- All tasks identical duration with many running simultaneously → likely I/O-bound (waiting for S3, etc.)

---

## Spark UI — SQL Tab & Physical Plan

- __SQL tab__ - shows all SQL/DataFrame queries; each query has a timeline, physical plan, and per-operator metrics

### Physical Plan Visualization

- DAG of physical operators rendered as a graph
- Each operator node shows -
    - Operator name (`BroadcastHashJoin`, `SortMergeJoin`, `FileScan parquet`, etc.)
    - Key metadata (`numOutputRows`, `outputRowsCount`, `peakMemory`)
    - Exchange nodes shown as shuffle boundaries (colored differently)

### SQL Tab Metrics per Operator

- `number of output rows` - rows produced by operator; compare to input rows to see filter effectiveness
- `peak memory` - peak memory used by operator (for aggregation, sort, join)
- `spill size` - bytes spilled to disk for this operator
- `time in aggregation build` / `time in sort` - operator-specific timing

### Reading the SQL Plan

```
(2) HashAggregate (keys=[category], functions=[sum(amount)])     ← codegen stage 2
+- Exchange hashpartitioning(category, 200)                       ← shuffle boundary
+- *(1) HashAggregate (keys=[category], functions=[partial_sum(amount)])  ← codegen stage 1
+- *(1) Filter (amount > 0)
+- *(1) FileScan parquet [category, amount]
PushedFilters: [IsNotNull(amount), GreaterThan(amount,0)]
PartitionFilters: [date = '2024-01-01']
```

- `*(N)` prefix - codegen stage N; all operators with same number fused into one generated class
- No `*` prefix - interpreted (Volcano model); check why (UDF? too many fields?)
- `PushedFilters` in `FileScan` - filters pushed to Parquet; good
- `PartitionFilters` in `FileScan` - partition pruning applied; directories skipped

---

## Spark UI — Storage Tab

- __Storage tab__ - shows all persisted RDDs and DataFrames; memory and disk usage per cached dataset

### Reading Storage Tab

- __Storage Level__ - `Memory Deserialized`, `Memory Serialized`, `Disk`, `Off-Heap`, etc.
- __Cached Partitions__ - how many partitions are currently cached
- __Fraction Cached__ - `cachedPartitions / totalPartitions`; less than $100\%$ means some partitions were evicted and will be recomputed on next access
- __Size in Memory__ - bytes consumed in executor memory stores
- __Size on Disk__ - bytes consumed on executor local disks

### Storage Tab Warnings

- `Fraction Cached < 100%` with `MEMORY_ONLY` level → cache thrashing; switch to `MEMORY_AND_DISK`
- Large `Size in Memory` consuming most of executor heap → less execution memory available → spill expected
- Many cached datasets from old jobs → call `unpersist()` on unused datasets

---

## Spark UI — Environment Tab

- __Environment tab__ - shows all configuration properties, JVM properties, classpath entries, system properties; essential for verifying configuration is as expected

### Key Things to Check

- `spark.serializer` - verify Kryo is configured if expected
- `spark.sql.adaptive.enabled` - verify AQE is on
- `spark.executor.memory` / `spark.executor.cores` - verify sizing
- `spark.sql.shuffle.partitions` - verify partition count
- Spark version and Scala version - verify compatibility
- Classpath entries - verify correct JAR versions; detect version conflicts

### Configuration Priority Debugging

```python
# Programmatically check effective config
for k, v in spark.sparkContext.getConf().getAll():
    if "memory" in k or "shuffle" in k or "adaptive" in k:
        print(f"{k} = {v}")
```

---

## Spark UI — Executors Tab

- __Executors tab__ - one row per executor + Driver; shows resource usage, task counts, and GC metrics per executor

### Key Executor Columns

- __RDD Blocks__ - cached RDD blocks stored on this executor
- __Storage Memory__ - used / total storage memory
- __Disk Used__ - local disk used for shuffle and spill
- __Cores__ - cores allocated to this executor
- __Active Tasks__ - tasks currently running
- __Failed Tasks__ - cumulative failed task count; high count → executor instability
- __GC Time__ - cumulative GC time; percentage of total task time
- __Input / Shuffle Read / Shuffle Write__ - cumulative data volumes

### Executor Health Diagnosis

- One executor with `Failed Tasks >> others` → hardware issue; investigate that node
- One executor with `GC Time >> others` → that executor has more data-heavy tasks; possible skew
- Dead executors shown with red background; tasks rescheduled
- Large `Disk Used` → heavy spill on that executor; may cause OOM if disk fills

---

## Debugging OOM Errors

- __OOM (OutOfMemoryError)__ - the most common Spark failure; different OOM types have different root causes and fixes

### OOM Error Types

- __Executor heap OOM__ - `java.lang.OutOfMemoryError: Java heap space`
    - Root causes -
        - Partition too large for executor heap; task processing more data than memory allows
        - Large `collect()` or `collectAsMap()` pulling too much data to one executor
        - Unbounded state accumulation (streaming without watermark)
        - Cached RDDs consuming all storage memory; no room for execution
    - Fixes -
        - Increase `spark.executor.memory`
        - Reduce partition size (increase `spark.sql.shuffle.partitions`)
        - Use `MEMORY_AND_DISK` storage level instead of `MEMORY_ONLY`
        - Switch to `RocksDBStateStore` for streaming
        - Eliminate unnecessary `collect()` calls

- __Executor GC overhead OOM__ - `java.lang.OutOfMemoryError: GC overhead limit exceeded`
    - JVM spending > $98\%$ of time in GC recovering < $2\%$ of heap
    - Fix - increase heap; reduce per-task data; use serialized storage levels; switch to ZGC

- __Driver OOM__ - `java.lang.OutOfMemoryError` on Driver
    - `collect()` / `take()` pulling too much data; exceeds `spark.driver.maxResultSize` or Driver heap
    - `broadcast()` of object too large for Driver heap
    - Too many tasks/stages → `DAGScheduler` metadata accumulates
    - Fix -
        - Replace `collect()` with `write()` to sink
        - Increase `spark.driver.memory`
        - Increase `spark.driver.maxResultSize`

- __Container OOM (YARN/K8s kills executor)__ - no Java OOM; container killed by OS
    - Total process memory (JVM heap + off-heap + native) exceeds container limit
    - Fix - increase `spark.executor.memoryOverhead` (default $10\%$ of heap or $384 MB$)
    - For Python workers - increase `spark.executor.memoryOverhead` significantly ($30-50\%$ of heap)

### OOM Debugging Workflow

```python
# Step 1 - identify which executor / task OOM'd
# Check Spark UI → Executors tab → look for dead executors
# Check Spark UI → Stages tab → look for failed tasks with OutOfMemoryError

# Step 2 - check if spill preceded OOM
# Spark UI → Stages tab → Spill (Memory/Disk) columns
# Spill present → execution memory insufficient; increase executor memory or reduce partition size

# Step 3 - check GC metrics
# Spark UI → Stages tab → GC Time column
# GC > 20% of task time → severe GC pressure

# Step 4 - check partition size
df.rdd.mapPartitions(lambda it: [sum(1 for _ in it)]).collect()
# Extreme outlier partition sizes → skew causing one task to OOM

# Step 5 - enable verbose GC logging and analyze
# -XX:+PrintGCDetails -XX:+PrintGCDateStamps → heap occupancy at GC time
```

---

## Debugging Skew from UI

- Skew is the most common cause of stage stalls; $99\%$ tasks complete while $1$ task runs for hours

### Identifying Skew in Spark UI

1. __Jobs tab__ - job duration much longer than sum of expected stage durations
2. __Stages tab__ - one stage taking much longer than others; stage not finishing despite most tasks done
3. __Tasks tab__ (drill into slow stage) -
    - Sort by `Duration DESC` - one or few tasks with $10-100×$ longer duration than median
    - Sort by `Shuffle Read Size DESC` - one task reading dramatically more data (reduce-side skew)
    - Sort by `Input Size DESC` - one task reading dramatically more input (map-side skew / uneven files)

### Skew Patterns

- __Reduce-side skew__ - `Shuffle Read Size` outlier; one partition has most data; hot key in join/groupBy
    - Fix - AQE skew join, manual salting, broadcast join

- __Map-side skew__ - `Input Size` outlier; one task reading huge file while others read small files
    - Fix - use splittable file format, `repartition` at read, reduce `maxPartitionBytes`

- __Explode skew__ - one array row has millions of elements; exploded into millions of rows on one partition
    - Fix - pre-filter, pre-aggregate, or limit array sizes before explode

### Skew Quantification

```python
# Measure partition skew ratio
from pyspark.sql.functions import spark_partition_id, count, stddev, mean

skew_df = df.groupBy(spark_partition_id().alias("pid")).count()
stats = skew_df.agg(
    mean("count").alias("mean"),
    stddev("count").alias("stddev"),
    (skew_df["count"].max() / skew_df["count"].median()).alias("max_to_median")
).collect()[0]

print(f"Mean: {stats.mean:.0f}, StdDev: {stats.stddev:.0f}, Max/Median: {stats.max_to_median:.1f}x")
# max_to_median > 5x → significant skew; tune AQE or apply salting
```

---

## Debugging Shuffle Spill

- Shuffle spill indicates tasks exceeding execution memory; causes significant slowdown due to disk I/O

### Identifying Spill in Spark UI

- __Stages tab__ - `Spill (Memory)` and `Spill (Disk)` columns; non-zero = spill occurred
    - `Spill (Memory)` = bytes spilled from memory (pre-compression size; indicates actual data volume)
    - `Spill (Disk)` = bytes written to disk (post-compression; lower than memory spill if good compression)
    - Gap between memory and disk spill = compression ratio; large gap = good compression

- __Tasks tab__ (drill into spilling stage) -
    - Sort by `Spill (Memory) DESC` - which tasks are spilling most
    - Compare spilling tasks' partition size vs non-spilling tasks; large outliers → skew-induced spill

### Spill Root Causes

- Partition too large for available execution memory
- High concurrency per executor - many tasks competing for execution memory pool
- Skewed partitions - one large partition spills while others don't
- Aggregation with high cardinality - hash map grows beyond execution memory

### Spill Debugging Workflow

```python
# Step 1 - check if spill is skew-induced
# Tasks tab → sort by Shuffle Read Size; spilling tasks also have large read sizes
# → Fix: skew handling (AQE, salting)

# Step 2 - check execution memory per task
# Execution memory per task = (unifiedPool × (1 - storageFraction)) / activeTasks
# If activeTasks is high, each task gets less execution memory

# Step 3 - calculate memory per task
executor_memory_gb = 16  # spark.executor.memory in GB
memory_fraction = 0.6    # spark.memory.fraction
storage_fraction = 0.5   # spark.memory.storageFraction
cores_per_executor = 5   # spark.executor.cores

reserved_mb = 300
usable_gb = executor_memory_gb - reserved_mb / 1024
unified_gb = usable_gb * memory_fraction
execution_gb = unified_gb * (1 - storage_fraction)
memory_per_task_gb = execution_gb / cores_per_executor
print(f"Execution memory per task: {memory_per_task_gb:.2f} GB")

# Step 4 - check partition size vs memory per task
# If partition > memory_per_task / 2, spill is likely
# Fix: increase spark.executor.memory or increase spark.sql.shuffle.partitions
```

---

## Debugging Slow Stages

- Slow stages have many possible root causes; systematic elimination is the approach

### Slow Stage Diagnosis Decision Tree

```
Is one task much slower than others (Tasks tab)?
├── YES → Skew (see Debugging Skew)
└── NO → All tasks slow equally
All tasks slow equally:
├── High GC Time (> 20% of task duration)?
│   ├── YES → GC tuning (increase memory, switch GC, reduce per-task data)
│   └── NO → Continue
├── High Shuffle Read Fetch Wait Time (Tasks tab)?
│   ├── YES → Network bound or remote executor overloaded
│   │         Fix: increase spark.reducer.maxSizeInFlight, check network bandwidth
│   └── NO → Continue
├── High Spill (Memory)?
│   ├── YES → Execution memory insufficient (see Debugging Shuffle Spill)
│   └── NO → Continue
├── High Executor Deserialize Time?
│   ├── YES → Large task closure; large broadcast variable re-read
│   └── NO → Continue
└── All metrics look normal but stage is slow?
→ Input data reading bottleneck (S3 rate limits, slow HDFS, many small files)
→ Check Input bytes vs Duration; compute MB/s read rate; compare to expected
```

### Scheduler Delay

- `Scheduler Delay` in Tasks tab = time from task submission to task start
- High scheduler delay → Driver overloaded; too many concurrent tasks; large task serialization
- Fix - reduce active task count; reduce task serialization size (smaller closures)

### Task Deserialization Time

- High `Task Deserialization Time` → large task closure being deserialized on executor
- Common cause - large broadcast variable or accidentally large closure
- Fix - use `sc.broadcast()` for large objects; check closure size

---

## Debugging FetchFailedException

- __`FetchFailedException`__ - thrown when a reduce task cannot fetch shuffle data from a map task's executor; the most cascading failure type in Spark

### What Triggers FetchFailedException

- Map-side executor died → shuffle files lost (if no External Shuffle Service)
- Network timeout fetching shuffle blocks from remote executor
- Executor OOM during shuffle serve causing it to die
- BlockManager on remote executor restarted (lost shuffle block registration)

### FetchFailedException Cascade

```
Map stage completes → reduce tasks start fetching →
Executor X dies → reduce tasks fetching from X get FetchFailedException →
DAGScheduler marks all map outputs from X as lost →
DAGScheduler resubmits map stage (only lost partitions if External Shuffle Service enabled) →
Map stage re-runs → reduce stage re-runs →
If X dies again → same cascade repeats until stage.maxConsecutiveAttempts exceeded
```

### Debugging FetchFailedException

```python
# Step 1 - check if executor died
# Spark UI → Executors tab → look for dead executors (red)
# Check why executor died: OOM? Node failure? YARN preemption?

# Step 2 - check if External Shuffle Service is enabled
print(spark.conf.get("spark.shuffle.service.enabled"))  # should be "true" in production

# Step 3 - check if OOM caused the executor failure
# Driver logs: "ERROR YarnSchedulerBackend$YarnDriverEndpoint: Lost executor X on host Y"
# Executor logs (YARN): check container exit code
# Exit code 137 = OOM kill by OS (Linux OOM killer)
# Exit code 143 = SIGTERM (YARN eviction/preemption)

# Step 4 - if repeated FetchFailedException on same stage
# Check spark.stage.maxConsecutiveAttempts (default 4)
# Stage failing 4 times → job abort with FetchFailed

# Step 5 - enable External Shuffle Service to prevent full stage rerun
spark.conf.set("spark.shuffle.service.enabled", "true")
spark.conf.set("spark.dynamicAllocation.enabled", "true")
```

### FetchFailedException Fix Strategies

- __Enable External Shuffle Service__ - shuffle files survive executor death; only lost tasks rerun not full stage
- __Prevent executor OOM__ - if executor OOM causes death; increase `spark.executor.memory` or reduce partition size
- __Increase network timeouts__ - for flaky networks; increase `spark.shuffle.io.maxRetries` and `spark.shuffle.io.retryWait`
- __Use spot/preemptible instances carefully__ - YARN preemption kills executors; External Shuffle Service critical for spot instances

---

## Debugging Task Serialization Errors

- __Task serialization errors__ - failures when Spark tries to serialize a task closure for dispatch to executor; occur before the task even starts running

### `NotSerializableException`

- __`org.apache.spark.SparkException: Task not serializable`__ with `java.io.NotSerializableException`
- Root cause - closure captures a non-serializable object
- Common patterns -
```python
    # BAD 1 - capturing database connection
    conn = database.connect()
    rdd.map(lambda x: conn.query(x))   # conn not serializable

    # BAD 2 - capturing 'self' which has non-serializable fields
    class MyTransformer:
        def __init__(self):
            self.model = ComplexModel()   # not serializable

        def transform(self, rdd):
            return rdd.map(lambda x: self.model.predict(x))  # captures self

    # FIX 1 - create inside task
    def query_in_task(x):
        conn = database.connect()   # created per-partition; not serialized
        return conn.query(x)
    rdd.mapPartitions(lambda it: map(query_in_task, it))

    # FIX 2 - broadcast serializable form
    model_bytes = serialize_model(model)   # serialize to bytes
    bc_model = sc.broadcast(model_bytes)
    rdd.map(lambda x: deserialize_model(bc_model.value).predict(x))
```

- Scala -
```scala
    // FIX - @transient lazy val for non-serializable fields
    class MyTransformer extends Serializable {
        @transient lazy val model: ComplexModel = loadModel()
        def transform(rdd: RDD[String]): RDD[String] =
            rdd.map(x => model.predict(x))  // model recreated on executor
    }
```

### Closure Too Large

- `SparkException: Serialized task ... was X bytes, which exceeds max allowed: ...`
- `spark.rpc.message.maxSize` (default $128 MB$) exceeded
- Cause - accidentally capturing large data structure (DataFrame schema, large collection, ML model)

```python
# Diagnose - check what's in the closure
import cloudpickle, sys

large_dict = {i: str(i) for i in range(1000000)}

func = lambda x: large_dict.get(x, "unknown")
print(f"Closure size: {sys.getsizeof(cloudpickle.dumps(func))} bytes")

# Fix - broadcast large objects
bc = sc.broadcast(large_dict)
func = lambda x: bc.value.get(x, "unknown")
print(f"Fixed closure size: {sys.getsizeof(cloudpickle.dumps(func))} bytes")
```

### KryoException: Buffer Overflow

- `com.esotericsoftware.kryo.KryoException: Buffer overflow`
- Object being serialized (for shuffle or broadcast) exceeds `spark.kryoserializer.buffer.max`
- Fix -
```python
    spark.conf.set("spark.kryoserializer.buffer.max", "512m")
    # Or reduce object size (use more partitions; don't aggregate too much per task)
```

### ClassNotFoundException on Executor

- Executor cannot find class referenced in task closure
- Cause - JAR containing the class not distributed to executors
- Fix -
```python
    # Add JAR at session creation
    spark = SparkSession.builder \
        .config("spark.jars", "/path/to/my-library.jar") \
        .getOrCreate()

    # Or add at runtime
    spark.sparkContext.addJar("/path/to/my-library.jar")
```

### InvalidClassException (serialVersionUID mismatch)

- `java.io.InvalidClassException: com.example.MyClass; local class incompatible: stream classdesc serialVersionUID = X, local class serialVersionUID = Y`
- Driver serialized class with version X; executor has class with version Y (different JAR)
- Fix - ensure identical JAR versions on Driver and all Executors; check `Environment tab → Classpath Entries`

> [!NOTE]
> The most impactful observability setup for production Spark -
> 1. Enable event logging to HDFS/S3 with rolling enabled (essential for post-mortem)
> 2. Export metrics to Prometheus/Grafana (real-time monitoring)
> 3. Alert on: executor death rate > 0, stage failure rate > 0, GC time > 20%, spill > 0 in critical jobs
> 4. Retain History Server with enough history for your SLA debugging window (7+ days)

> [!TIP]
> Systematic debugging order for any Spark performance problem -
> ```
> 1. Jobs tab → identify slow job
> 2. Stages tab → identify slow stage; check shuffle/spill/GC columns
> 3. Tasks tab (sorted by Duration DESC) → is it one task or all?
>    - One task → skew (check shuffle read size)
>    - All tasks → GC, spill, network, or I/O bound
> 4. SQL tab → check physical plan for unexpected operations
>    - Unexpected shuffle? → missing co-partitioning or wrong join hint
>    - No partition pruning? → filter type mismatch or predicate not pushed
>    - SMJ instead of BHJ? → stale stats; use broadcast hint
> 5. Executors tab → any dead executors? high GC on specific executors?
> 6. Environment tab → verify config is as expected
> ```
