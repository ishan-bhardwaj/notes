# Shuffle Internals

## Shuffle Overview

- __Shuffle__ - the process of redistributing data across partitions between two stages; the most expensive operation in Spark - involves disk I/O, serialization, and network transfer on both map and reduce sides
- Triggered by any wide dependency - `reduceByKey`, `groupByKey`, `join`, `distinct`, `repartition`, `sortByKey`, `cogroup`
- Every shuffle has two sides -
    - __Map side (shuffle write)__ - tasks in the upstream stage write their output partitioned by the target partitioner; data written to local disk
    - __Reduce side (shuffle read)__ - tasks in the downstream stage fetch their partition's data from all map outputs; data pulled over network
- Shuffle is the stage boundary - `DAGScheduler` cuts a new stage at every `ShuffleDependency`

### Shuffle Data Flow

| Stage N (Map Side)                                | Stage N+1 (Reduce Side)                                   |
| ------------------------------------------------- | --------------------------------------------------------- |
| `Task 0` → shuffle write → disk                   | `Task 0` → fetch partition 0 from all map tasks           |
| `Task 1` → shuffle write → disk                   | `Task 1` → fetch partition 1 from all map tasks           |
| `Task 2` → shuffle write → disk                   | `Task 2` → fetch partition 2 from all map tasks           |
| ↓                                                 | ↑                                                         |
| `MapOutputTracker` registers map output locations | `MapOutputTracker` consulted for locations of map outputs |

### Shuffle ID and Registration

- Every `ShuffleDependency` is assigned a unique `shuffleId` by `SparkContext.newShuffleId()`
- `DAGScheduler` registers the shuffle with `MapOutputTrackerMaster` before the map stage runs
- Map tasks report `MapStatus(blockManagerId, partitionSizes)` to `DAGScheduler` on completion
- `DAGScheduler` forwards `MapStatus` to `MapOutputTrackerMaster`
- Reduce tasks query `MapOutputTrackerWorker` (local) → `MapOutputTrackerMaster` (Driver) for locations

### Why Shuffle is Expensive

- __Disk I/O on map side__ - every map task writes ALL output to local disk; no streaming to reduce side
- __Network transfer__ - every reduce task fetches data from every map task; full N×M communication pattern
- __Serialization__ - data serialized on write, deserialized on read (unless Tungsten binary format used)
- __Sorting__ - `SortShuffleManager` sorts by partition ID on map side; sort-merge on reduce side
- __Memory pressure__ - both sides need buffers; reduce side needs sort/aggregation memory

---

## ShuffleManager

- __`ShuffleManager`__ - pluggable interface for shuffle implementation; registered in `SparkEnv`; one instance per executor and Driver
- Configured via `spark.shuffle.manager` - default is `sort` (`SortShuffleManager`); was `hash` before Spark 1.2
- `HashShuffleManager` removed in Spark 2.0 - produced `numMappers × numReducers` files; unscalable at large partition counts

### ShuffleManager Interface

- `registerShuffle[K, V, C](shuffleId, dependency)` -
    - Called on Driver when a `ShuffleDependency` is created
    - Returns a `ShuffleHandle` - a descriptor that determines which write path to use
    - Three handle types in `SortShuffleManager` - `BypassMergeSortShuffleHandle`, `SerializedShuffleHandle`, `BaseShuffleHandle`
- `getWriter[K, V](handle, mapId, context, metrics)` -
    - Called on executor at start of map task
    - Returns a `ShuffleWriter` appropriate for the handle type
- `getReader[K, C](handle, startPartition, endPartition, context, metrics)` -
    - Called on executor at start of reduce task
    - Returns a `ShuffleReader` for fetching and deserializing reduce-side data
- `unregisterShuffle(shuffleId)` - called after all downstream stages complete; triggers cleanup
- `shuffleBlockResolver` - returns `IndexShuffleBlockResolver`; maps shuffle block IDs to file segments

### ShuffleHandle Selection (SortShuffleManager)

- `registerShuffle` logic -
    - If `shouldBypassMergeSort(dependency)` → `BypassMergeSortShuffleHandle`
        - Conditions - no map-side aggregation AND `numPartitions ≤ spark.shuffle.sort.bypassMergeThreshold` (default $200$)
    - Else if `canUseSerializedShuffle(dependency)` → `SerializedShuffleHandle`
        - Conditions - serializer supports relocation AND no map-side aggregation AND `numPartitions ≤ 16777216`
    - Else → `BaseShuffleHandle` (uses `SortShuffleWriter`)

---

## SortShuffleManager

- __`SortShuffleManager`__ - the default and only production shuffle manager since Spark 2.0; replaced `HashShuffleManager`
- Core design - each map task produces exactly ONE data file and ONE index file regardless of the number of output partitions
    - `HashShuffleManager` produced one file per output partition per mapper → `M × R` files total → millions of files at scale
    - `SortShuffleManager` produces `2 × M` files total → scales linearly with mappers only

### Key Design Decisions

- __Sort by partition ID on map side__ - sorting allows a single sequential pass to write all partitions into one file; the index file records byte offsets for each partition
- __Pluggable write paths__ - three writers (`BypassMergeSortShuffleWriter`, `UnsafeShuffleWriter`, `SortShuffleWriter`) selected at shuffle registration time; not at task time
- __`IndexShuffleBlockResolver`__ - translates `ShuffleBlockId(shuffleId, mapId, reduceId)` to a byte range in the data file using the index file

### File Layout

- `shuffle_{shuffleId}{mapId}0.data` - all partitions concatenated, sorted by partition ID
- `shuffle{shuffleId}{mapId}_0.index` - ($numPartitions + 1$) long values; `offset[i]` to `offset[i+1]` is partition `i`

- Index file has `numPartitions + 1` entries - entry `i` is the byte offset of partition `i`'s start; entry `numPartitions` is the total file size
- Reduce task for partition `i` reads bytes `[index[i], index[i+1])` from the data file

---

## SortShuffleWriter

- __`SortShuffleWriter`__ - the default fallback shuffle writer; used when map-side aggregation is required OR when neither `BypassMergeSortShuffleWriter` nor `UnsafeShuffleWriter` conditions are met
- Uses `ExternalSorter` internally - handles both in-memory sorting and spill-to-disk

### Write Process

1. Task calls `writer.write(records)` with the full partition iterator
2. `ExternalSorter.insertAll(records)` - inserts all records into in-memory structure -
    - If aggregation required - inserts into `PartitionedAppendOnlyMap` (hash map with combine)
    - If no aggregation - inserts into `PartitionedPairBuffer` (append-only array)
3. If memory pressure during insertion - spills to disk (one spill file per spill event)
4. `writer.stop(success=true)` triggers `ExternalSorter.writePartitionedMapOutput` -
    - Merges in-memory data with all spill files via k-way merge-sort
    - Writes merged output to final shuffle data file partitioned by partition ID
    - Writes index file with partition byte offsets
5. Returns `MapStatus(blockManagerId, partitionSizes)`

### When SortShuffleWriter is Used

- Map-side aggregation required (eg - `reduceByKey`, `aggregateByKey`) - `ExternalSorter` applies combiner during insertion
- Serializer does not support object relocation (eg - Java serializer with `reduceByKey`)
- `numPartitions > 16777216` - `UnsafeShuffleWriter` partition count limit exceeded

---

## BypassMergeSortShuffleWriter

- __`BypassMergeSortShuffleWriter`__ - the simplest shuffle writer; writes one temporary file per output partition then concatenates them; no sorting of records within partitions
- Used when - no map-side aggregation AND `numPartitions ≤ spark.shuffle.sort.bypassMergeThreshold` (default $200$)

### Write Process

1. Opens one `DiskBlockObjectWriter` per output partition - `numPartitions` open file handles simultaneously
2. For each record, calls `partitioner.getPartition(key)` and writes record to the corresponding writer
3. After all records processed - closes all writers; each produces a temporary file
4. Concatenates all temporary files into one final data file in partition order
5. Writes index file
6. Deletes temporary files
7. Returns `MapStatus`

### Why It Exists

- For small partition counts, the cost of sorting (required by `SortShuffleWriter`) exceeds the cost of maintaining multiple open file handles
- No sort → records within a partition are in insertion order, not sorted by key
- Simple implementation - no `ExternalSorter`, no spill (each partition's writer streams directly to disk)
- Memory usage - one open `BufferedOutputStream` per partition; `numPartitions × bufferSize` memory; default buffer $32 KB$ × $200$ partitions = $6.4 MB$ maximum

### Limitations

- `numPartitions` open file handles simultaneously - can exhaust OS file descriptor limits at high partition counts
    - Default threshold $200$ chosen conservatively for this reason
- No map-side aggregation - cannot be used for `reduceByKey`; all values shipped raw
- No in-memory sort buffer - cannot sort by key within partitions

---

## UnsafeShuffleWriter

- __`UnsafeShuffleWriter`__ - Tungsten-optimized shuffle writer; operates on serialized binary data throughout; avoids deserialization during sort
- Used when - serializer supports relocation AND no map-side aggregation AND `numPartitions ≤ 16777216`
- "Serializer supports relocation" means the serializer can move serialized objects in memory without invalidating them - Kryo and Spark's `UnsafeRowSerializer` support this; Java serializer does not (contains absolute pointers)

### Write Process

1. Each record serialized immediately on insertion using `SerializerInstance`
2. Serialized bytes written into a Tungsten `MemoryBlock` page
3. A `long` pointer `(partitionId << 40) | (pageNumber << 27) | offset` stored in a `LongArray` sort buffer
    - Upper 24 bits - partition ID
    - Lower 40 bits - encoded page+offset pointer to the serialized record in a Tungsten page
4. When memory pressure occurs - `ShuffleExternalSorter.spill()` -
    - Sorts the `LongArray` by the upper 24 bits (partition ID) using radix sort
    - Writes serialized records to a spill file in partition order (raw bytes, no deserialization)
    - Releases Tungsten pages
5. At `stop()` - merge all spill files (and remaining in-memory records) into final data file + index file

### Why UnsafeShuffleWriter is Fastest

- __No deserialization during sort__ - sort operates on the `LongArray` of pointers; actual record bytes never touched during sort
- __Radix sort__ - sorts pointer array by partition ID using radix sort (O(n) vs O(n log n) for comparison sort); extremely fast for partition-ID-only sorting
- __Tungsten pages__ - records stored in off-heap or on-heap `MemoryBlock` pages; contiguous memory layout; cache-friendly
- __Binary spill files__ - spill writes raw serialized bytes; no re-serialization cost

### Partition Count Limit

- Partition ID packed into upper 24 bits of `long` pointer → maximum $2^{24} = 16777216$ partitions
- Exceeding this limit forces fallback to `SortShuffleWriter`

---

## ExternalSorter

- __`ExternalSorter`__ - the core data structure used by `SortShuffleWriter`; a spill-aware sorter and optional combiner for `(partitionId, key) → value` data
- Also used by `sortByKey` and any sort-based aggregation
- A `MemoryConsumer` - registers with `TaskMemoryManager`; can be asked to spill by other consumers

### In-Memory Structures

- __`PartitionedAppendOnlyMap`__ (when aggregation required) -
    - Hash map keyed by `(partitionId, key)`
    - On insertion - calls `mergeValue(existing, newValue)` if key exists; `createCombiner(value)` if new
    - Backed by an open-addressing hash table with linear probing
    - Size-tracked - estimates its own memory usage to trigger spill proactively
- __`PartitionedPairBuffer`__ (when no aggregation) -
    - Simple append-only array of `(partitionId, key, value)` triples
    - No deduplication or combining
    - More memory-efficient than hash map for non-aggregating cases

### Insertion

- `insertAll(records)` - iterates through all records -
    - If aggregating - calls `map.changeValue((partitionId, key), updateFunc)`
    - If not aggregating - calls `buffer.insert(partitionId, key, value)`
    - After every $1000$ records - checks `maybeSpill()` → calls `TaskMemoryManager.acquireExecutionMemory`
    - If memory acquisition fails - calls `spill()`

### Spill Process

- `spill()` -
    1. Sort in-memory structure by `(partitionId, key)` using `TimSort`
        - `partitionId` is primary sort key
        - `key` is secondary sort key (using provided `Ordering[K]` if available; hash order otherwise)
    2. Write sorted records to a `SpillFile` via `DiskBlockObjectWriter` -
        - Serialized using `SerializerInstance`
        - Optionally compressed via `SerializerManager`
    3. Record `SpilledFile(file, blockId, serializerBatchSizes, partitionLengths)` metadata
    4. Clear in-memory structure; release memory back to `TaskMemoryManager`
- Multiple spills → multiple `SpilledFile` objects; one final merge

### Merge Process

- `writePartitionedMapOutput(blockId, writer)` -
    1. Sort remaining in-memory records
    2. Open `SpillReader` for each spill file
    3. K-way merge using `mergeSort(iterators, comparator)` -
        - Priority queue (min-heap) over `(partitionId, key)` from all sources
        - Yields records in globally sorted `(partitionId, key)` order
    4. If aggregation - merge values for same key using `mergeCombiners`
    5. Write merged output to shuffle data file; record partition byte offsets for index file

### TimSort Details

- Java's `Arrays.sort` (TimSort) used for in-memory sort
- TimSort is O(n log n) worst case but O(n) for already-sorted data
- `ExternalSorter` sorts by `(partitionId, key)` - if data arrives pre-sorted by partition, TimSort runs in O(n)

---

## PartitionedAppendOnlyMap

- __`PartitionedAppendOnlyMap`__ - open-addressing hash map used inside `ExternalSorter` when map-side aggregation is required; keys are `(partitionId, K)` pairs
- Extends `AppendOnlyMap[K, V]` - a Spark-specific hash map optimized for the append-and-combine pattern
- Backed by a flat `Array[AnyRef]` with alternating key-value slots - `[key0, value0, key1, value1, ...]`
    - Avoids per-entry object overhead of Java's `HashMap`
    - Linear probing for collision resolution

### AppendOnlyMap Internals

- Initial capacity $64$ entries; grows by $2×$ when $70%$ full
- `update(key, value)` - if key exists, replaces value; if not, inserts
- `changeValue(key, updateFunc)` - if key exists, calls `updateFunc(true, existing)`; if not, calls `updateFunc(false, null)`; this is the combine operation
- `iterator()` - returns all entries; used by `ExternalSorter` before spill or output

### Size Tracking

- `SizeTrackingAppendOnlyMap` wraps `AppendOnlyMap` with memory estimation
- Samples actual size via `SizeEstimator.estimate(this)` every $N$ insertions
- Builds a linear regression model to predict size growth per insertion
- Triggers spill check when estimated size exceeds a threshold
- Estimation is approximate - actual size may differ; safety margin built into spill threshold

### Destructive Sort

- `destructiveSortedIterator(comparator)` - sorts the backing array in-place and returns a sorted iterator
- "Destructive" because the map cannot be used after this call - array is rearranged for sort
- Used by `ExternalSorter.spill()` - after sorting, map contents written to spill file and map cleared

---

## Shuffle Write Path

- End-to-end flow from task start to `MapStatus` reported

### Step 1 - Task Receives ShuffleHandle

- `CoarseGrainedExecutorBackend` receives `LaunchTask` from Driver
- Task is a `ShuffleMapTask` - wraps the RDD partition and `ShuffleDependency`
- `ShuffleMapTask.runTask` calls `manager.getWriter(dep.shuffleHandle, partitionId, context, metrics)`

### Step 2 - Writer Selection

- `SortShuffleManager.getWriter` matches on handle type -
    - `BypassMergeSortShuffleHandle` → `BypassMergeSortShuffleWriter`
    - `SerializedShuffleHandle` → `UnsafeShuffleWriter`
    - `BaseShuffleHandle` → `SortShuffleWriter`

### Step 3 - Compute Partition

- `rdd.iterator(partition, context)` - computes the RDD partition; returns an iterator of records
- Iterator is lazy - records computed on demand as writer consumes them

### Step 4 - Write Records

- `writer.write(rdd.iterator(partition, context))` -
    - All records from the partition passed to the writer
    - Writer routes each record to the appropriate output partition via `partitioner.getPartition(key)`
    - Records buffered in memory (with possible spills to disk)

### Step 5 - Commit and Index

- `writer.stop(success=true)` -
    - Merges spill files if any
    - Writes final data file and index file via `IndexShuffleBlockResolver.writeIndexFileAndCommit`
    - Returns `MapStatus(blockManagerId, uncompressedSizes)`
        - `uncompressedSizes` - array of uncompressed byte size per output partition; used by reduce tasks to decide fetch order

### Step 6 - Report to Driver

- `ShuffleMapTask.runTask` returns `MapStatus` to `TaskRunner`
- `TaskRunner` sends `StatusUpdate(taskId, SUCCESS, serializedMapStatus)` to Driver
- `DAGScheduler` receives status update; registers `MapStatus` with `MapOutputTrackerMaster`
- `MapOutputTrackerMaster` updates its registry; reduce tasks can now query for this map output

---

## Shuffle Read Path

- End-to-end flow from reduce task start to deserialized records available

### Step 1 - Query Map Output Locations

- `ResultTask` or downstream `ShuffleMapTask` calls `manager.getReader(handle, startPartition, endPartition, context, metrics)`
- `BlockStoreShuffleReader` created
- Reader calls `mapOutputTracker.getMapSizesByExecutorId(shuffleId, startPartition, endPartition)` -
    - `MapOutputTrackerWorker` checks local cache; if miss, fetches from Driver's `MapOutputTrackerMasterEndpoint` via RPC
    - Returns `Seq[(BlockManagerId, Seq[(BlockId, Long, Int)])]` - for each executor, the list of blocks it holds and their sizes

### Step 2 - Fetch Blocks

- `ShuffleBlockFetcherIterator` created with the list of blocks to fetch -
    - Splits blocks into local (same executor) and remote
    - Local blocks fetched directly from `BlockManager` without network
    - Remote blocks fetched via `ShuffleClient` (either `NettyBlockTransferService` or `ExternalShuffleClient`)
    - Fetch parallelism controlled by `spark.reducer.maxSizeInFlight` ($48 MB$) and `spark.reducer.maxBlocksInFlightPerAddress`
    - Fetched blocks stored as `ManagedBuffer` - either in memory or as temp files on disk if `spark.shuffle.detectCorrupt` enabled

### Step 3 - Deserialize

- Each fetched block deserialized via `SerializerManager.dataDeserializeStream` -
    - Decompresses if compressed
    - Deserializes using `SerializerInstance`
    - Returns `Iterator[(K, C)]`

### Step 4 - Aggregate and Sort

- If aggregation required - `ExternalAppendOnlyMap` applied -
    - `createCombiner`, `mergeValue`, `mergeCombiners` from `Aggregator` applied
    - Spill-aware - can spill to disk if memory insufficient
- If sort required - `ExternalSorter` applied with only sort (no combine)
- If neither - records passed through directly

### Step 5 - Return Iterator

- `BlockStoreShuffleReader.read()` returns the final `Iterator[(K, C)]`
- Reduce task consumes this iterator to produce its output

### Fetch Retry and Error Handling

- `RetryingBlockFetcher` wraps the fetch with retry logic -
    - `spark.shuffle.io.maxRetries` retries on `IOException`
    - `spark.shuffle.io.retryWait` wait between retries
- `FetchFailedException` thrown if block not found or fetch fails after retries -
    - Reduce task fails with `FetchFailed`
    - `DAGScheduler` receives `FetchFailed`; marks the map stage as failed
    - Re-submits the failed map tasks to regenerate lost shuffle output
    - Then re-submits the failed reduce tasks

---

## IndexShuffleBlockResolver

- __`IndexShuffleBlockResolver`__ - translates `ShuffleBlockId(shuffleId, mapId, reduceId)` into a `ManagedBuffer` containing the bytes for partition `reduceId` from map task `mapId`
- Uses the index file to locate the byte range for a specific partition within the data file

### File Resolution

- `getDataFile(shuffleId, mapId)` - returns the `.data` file path via `DiskBlockManager.getFile`
- `getIndexFile(shuffleId, mapId)` - returns the `.index` file path
- `getBlockData(blockId)` -
    1. Opens index file; reads offsets at positions `reduceId` and `reduceId + 1`
    2. Computes `offset = indexOffsets[reduceId]`, `length = indexOffsets[reduceId+1] - offset`
    3. Returns `FileSegmentManagedBuffer(file, offset, length)` - a lazy reference to the file segment
    4. `FileSegmentManagedBuffer` enables zero-copy transfer via Netty's `FileRegion`

### Atomic Write

- `writeIndexFileAndCommit(shuffleId, mapId, lengths, dataTmp)` -
    - Receives `lengths` array (bytes per partition) and a temp data file
    - Computes cumulative offsets from lengths array
    - Writes index file atomically (temp file then rename)
    - Renames temp data file to final data file
    - Atomicity prevents partial reads if task fails mid-write
- If a map task is retried (speculative execution), the second attempt's output is only committed if the first attempt hasn't already committed (`OutputCommitCoordinator` grants commit permission to exactly one attempt)

---

## ShuffleBlockFetcher

- __`ShuffleBlockFetcherIterator`__ - the reduce-side component that fetches shuffle blocks from remote executors; implements `Iterator[(BlockId, InputStream)]`
- Created inside `BlockStoreShuffleReader.read()` with the list of blocks to fetch

### Fetch Scheduling

- Categorizes all blocks into local and remote -
    - Local blocks - same executor; fetched synchronously from `BlockManager.getLocalBytes`
    - Remote blocks - grouped by `BlockManagerId` (executor); fetched asynchronously via `ShuffleClient`
- Fetch queue management -
    - Maintains `bytesInFlight` counter - total bytes currently being fetched but not yet consumed
    - Before issuing a new fetch request, checks `bytesInFlight < spark.reducer.maxSizeInFlight`
    - Issues fetch requests greedily until limit reached; then blocks until a fetched block is consumed
- `maxBlocksInFlightPerAddress` (default `Int.MaxValue`) - limits concurrent requests to a single executor
    - Tune to prevent overwhelming executors that hold many map outputs (common with skewed data)

### Memory Management for Fetched Blocks

- Small blocks (size ≤ `spark.maxRemoteBlockSizeFetchToMem`, default $200 MB$) - stored in memory as `ManagedBuffer`
- Large blocks - streamed to a temp file on disk to avoid heap pressure; returned as `FileSegmentManagedBuffer`
- This prevents OOM when a single shuffle block is larger than available heap

### Corrupt Block Detection

- `spark.shuffle.detectCorrupt` (default `true`) - enables checksum validation of fetched blocks
- CRC32 checksum computed on write; verified on read
- Corrupt block → `FetchFailedException` → task fails → `DAGScheduler` re-runs the map task

---

## ShuffleReader

- __`ShuffleReader`__ - interface with single method `read(): Iterator[Product2[K, C]]`
- One implementation - `BlockStoreShuffleReader`
- Wraps `ShuffleBlockFetcherIterator` with deserialization, optional aggregation, optional sort

### BlockStoreShuffleReader Pipeline

- `ShuffleBlockFetcherIterator` - fetches raw bytes per block → 
    - `DeserializationStream` - deserializes bytes to (K, V) records → 
    - `InterruptibleIterator` - supports task cancellation → 
    - `ExternalAppendOnlyMap` - optional aggregation (`mergeCombiners`) → 
    - `ExternalSorter`  - optional sort (if `keyOrdering` defined) → 
    - final `Iterator[(K, C)]` - returned to reduce task

- Each step is lazy - records flow through the pipeline on demand
- `InterruptibleIterator` checks `TaskContext.isInterrupted` on each `next()` call - allows task cancellation without waiting for all records to be processed

---

## Shuffle File Consolidation

- __Shuffle file consolidation__ - technique to reduce the number of shuffle files by reusing file handles across multiple map tasks running on the same executor
- Was a feature of `HashShuffleManager` (`spark.shuffle.consolidateFiles=true`) - REMOVED in Spark 2.0 when `HashShuffleManager` was removed
- `SortShuffleManager` makes consolidation unnecessary - already produces only $2 × M$ files total (one data + one index per mapper)
- Understanding this matters for interviews - candidates often cite "consolidation" from old blog posts; it's historical

### Why SortShuffle Doesn't Need Consolidation

- `HashShuffleManager` without consolidation - `M × R` files (one per mapper per reducer)
    - $1000$ mappers × $1000$ reducers = $1,000,000$ files - catastrophic for filesystem metadata
- `HashShuffleManager` with consolidation - `C × R` files (one per executor core per reducer)
    - Better but still O(R) per core
- `SortShuffleManager` - $2 × M$ files always; independent of reducer count
    - The index file eliminates the need for per-reducer files

---

## Shuffle Compression

- __Shuffle compression__ - compressing shuffle data on write to reduce disk I/O and network transfer volume; decompressed on read
- Controlled by `spark.shuffle.compress` (default `true`)
- Applied at two points -
    - __Spill files__ - `spark.shuffle.spill.compress` (default `true`); compresses data written to disk during spills
    - __Shuffle output files__ - `spark.shuffle.compress`; compresses final shuffle data files

### Compression Codec

- Configured via `spark.io.compression.codec` (default `lz4`)
- Available codecs -
    - `lz4` - fast compression/decompression; moderate ratio; default; best for most workloads
    - `lzf` - older; slower than lz4; less commonly used
    - `snappy` - fast; similar to lz4; Google's codec
    - `zstd` - better compression ratio than lz4/snappy; higher CPU cost; good for network-bound jobs
- `spark.io.compression.lz4.blockSize` (default $32 KB$) - LZ4 compression block size; larger = better ratio, more memory
- `spark.io.compression.zstd.level` (default $1$) - zstd compression level; $1$=fastest, $22$=best ratio

### When to Disable Compression

- Data already compressed (Parquet, ORC, gzip files) - compressing again wastes CPU with no size benefit
- CPU-bound jobs where compression overhead exceeds I/O savings
- Check Spark UI - if `Shuffle Write Size` ≈ `Shuffle Write Time` is small relative to task time, compression may not be worth the CPU

---

## Shuffle Spill & Merge

- Covered in detail under `ExternalSorter` and `MemoryConsumer` - summary here for completeness

### Spill Triggering

- `ExternalSorter.maybeSpill()` called after every $1000$ insertions -
    - Calls `TaskMemoryManager.acquireExecutionMemory(amountToRequest)`
    - If granted < requested → triggers `spill()`
- `ShuffleExternalSorter` (used by `UnsafeShuffleWriter`) - spills when Tungsten page allocation fails

### Spill File Format

- `SortShuffleWriter` / `ExternalSorter` spill format -
    - Serialized records written in `(partitionId, key, value)` order sorted by `(partitionId, key)`
    - Each serializer batch delimited; batch sizes recorded in `SpilledFile` metadata
    - Compressed if `spark.shuffle.spill.compress=true`
- `UnsafeShuffleWriter` spill format -
    - Raw serialized bytes of `UnsafeRow` (or whatever type) written in partition order
    - Partition boundaries recorded; no key ordering within partition

### Merge Process

- Final merge called in `writer.stop()` -
    - Opens one `SpillReader` per spill file
    - One in-memory iterator for remaining in-memory data
    - K-way merge using priority queue ordered by `(partitionId, key)`
    - Output written partition-by-partition to final data file
    - `IndexShuffleBlockResolver.writeIndexFileAndCommit` writes index and commits files

### Merge Performance

- Each spill file read sequentially - good I/O pattern
- K-way merge has O(n log k) complexity where k = number of spill files
- Large number of spill files (>$10$) indicates severe memory pressure - increase `spark.executor.memory` or reduce partition size

---

## ExternalShuffleService

- __`ExternalShuffleService`__ - a long-running daemon process on each worker node that serves shuffle files independently of executor JVM lifecycle
- Configured via `spark.shuffle.service.enabled=true`
- Required for safe dynamic allocation (`spark.dynamicAllocation.enabled=true`) - without it, killing idle executors destroys their shuffle output, causing downstream stage failures

### Architecture

- Runs as a separate JVM process on each worker node -
    - YARN - runs inside the NodeManager as an auxiliary service
    - Standalone - runs as a separate daemon via `$SPARK_HOME/sbin/start-shuffle-service.sh`
    - K8s - runs as a sidecar (or external service); see Shuffle Tracking section
- Listens on `spark.shuffle.service.port` (default $7337$)
- Uses same Netty server protocol as executor's `NettyBlockTransferService` - reduce tasks don't need to know whether they're talking to an executor or the external service

### Executor Registration

- When an executor starts and `ExternalShuffleService` is enabled -
    - Executor sends `RegisterExecutor(appId, execId, executorInfo)` to local `ExternalShuffleService`
    - `ExternalShuffleService` records the executor's shuffle file directories
- When an executor writes a shuffle file -
    - File written to local disk as normal (same `DiskBlockManager` paths)
    - `ExternalShuffleService` knows the directory; can serve the file even if executor dies

### Block Serving

- Reduce tasks send `OpenBlocks` to `ExternalShuffleService` instead of executor
- `ExternalShuffleService` looks up file path via registered executor info
- Serves `FileSegmentManagedBuffer` - same zero-copy transfer as executor-served blocks
- If executor died - files still on disk; `ExternalShuffleService` still serves them

### Application Cleanup

- `ExternalShuffleService` retains shuffle data until notified of application completion
- Driver sends `ApplicationRemoved(appId)` to each `ExternalShuffleService` on job completion
- `ExternalShuffleService` then deletes all shuffle files for the application

---

## ExternalShuffleService Internals

### ExternalShuffleBlockResolver

- Maintains `registeredExecutors: Map[(appId, execId), ExecutorShuffleInfo]`
- `ExecutorShuffleInfo` - local dirs array, subDirsPerLocalDir count, shuffle manager class name
- `getBlockData(appId, execId, shuffleId, mapId, reduceId)` -
    - Looks up `ExecutorShuffleInfo` for the executor
    - Reconstructs file path using same logic as `IndexShuffleBlockResolver`
    - Returns `FileSegmentManagedBuffer`

### State Persistence (Spark 3.0+)

- `spark.shuffle.service.db.enabled` (default `true`) - persists registration state to LevelDB
- Survives `ExternalShuffleService` restart (eg - NodeManager restart in YARN)
- Without persistence - executor re-registration required after service restart; dynamic allocation must re-launch executors

### Merge Finalization (Push-Based Shuffle)

- With push-based shuffle enabled, `ExternalShuffleService` also handles merge finalization -
    - Receives pushed blocks from map tasks during map stage
    - Merges pushed blocks into merged shuffle files
    - `FinalizeShuffleMerge` RPC signals merge completion for a shuffle partition
    - Reduce tasks fetch from merged files for better I/O patterns

---

## Shuffle Tracking (K8s Dynamic Allocation)

- __Shuffle Tracking__ - alternative to `ExternalShuffleService` for dynamic allocation on Kubernetes; introduced in Spark 3.0
- `ExternalShuffleService` is difficult to deploy on K8s - requires a DaemonSet or sidecar; stateful; complex lifecycle management
- Shuffle Tracking avoids the need for `ExternalShuffleService` by tracking which executors hold live shuffle data and NOT decommissioning them until downstream stages complete

### How Shuffle Tracking Works

- Enable via `spark.dynamicAllocation.shuffleTracking.enabled=true` (automatically set when `spark.dynamicAllocation.enabled=true` on K8s without `ExternalShuffleService`)
- `ExecutorMonitor` (inside `ExecutorAllocationManager`) tracks shuffle state per executor -
    - When a map task completes - records `(shuffleId, mapId) → executorId` mapping
    - When a reduce task completes consuming a shuffle - decrements reference count
    - When all reduce tasks for a shuffle have completed - shuffle reference count drops to zero
- An executor is eligible for decommissioning only when -
    - It has been idle for `spark.dynamicAllocation.executorIdleTimeout` AND
    - It holds no live shuffle data (reference count = zero for all its shuffles)
- Executor with live shuffle data is kept alive even if idle - prevents the shuffle loss problem

### Shuffle Tracking Timeout

- `spark.dynamicAllocation.shuffleTracking.timeout` (default infinite) - max time to retain an executor purely for shuffle data
- Setting this causes executors to be decommissioned even if they hold shuffle data after the timeout
- Trade-off - memory savings vs risk of shuffle re-computation
- Set to a value longer than the typical reduce stage duration to be safe

### Limitations vs ExternalShuffleService

- Executors cannot be killed until their shuffle data is no longer needed - less aggressive scaling down
- `ExternalShuffleService` allows executors to be killed immediately; data remains on disk
- For workloads with long-running reduce stages, shuffle tracking holds executors longer than optimal

---

## Push-Based Shuffle (Spark 3.2+)

- __Push-based shuffle__ - map tasks push their output to `ExternalShuffleService` during the map stage; blocks merged before reduce stage starts; reduce tasks read merged files instead of many small individual map outputs
- Enabled via `spark.shuffle.push.enabled=true`
- Requires `ExternalShuffleService` and Spark 3.2+

### Motivation

- Traditional pull-based shuffle - reduce task must connect to every map task's executor (or ESS) to fetch its partition; N map tasks × M reduce tasks = N×M connections and N small reads per reduce task
- With $1000$ mappers, each reduce task makes $1000$ small fetch requests - high connection overhead, random I/O pattern
- Push-based shuffle - each `ExternalShuffleService` node receives and merges blocks from all map tasks for local partitions; reduce task makes fewer, larger reads

### Push Process (Map Side)

1. Map task writes shuffle output normally (data file + index file)
2. After writing, map task pushes blocks to `ExternalShuffleService` nodes -
    - For each output partition, determines the target ESS node (based on where reduce task will run)
    - Sends `PushBlock(shuffleId, shuffleMergeId, mapIndex, reduceId, data)` to target ESS
    - Push is best-effort - failures are tolerated; missing blocks fall back to normal pull
3. ESS merges received blocks into a merged data file per partition

### Merge Process (ExternalShuffleService Side)

- ESS maintains `AppShuffleMergeManager` per application -
    - Receives pushed blocks; buffers them
    - When all expected blocks received (or timeout) - finalizes the merged file
    - `FinalizeShuffleMerge` message from Driver triggers final merge for a shuffle partition
- Merged file format - same as regular shuffle data file; index file records offsets

### Reduce Side Changes

- `BlockStoreShuffleReader` first attempts to fetch from merged location (ESS) -
    - Queries `MapOutputTrackerMaster` for merged block locations
    - If merged block available - fetches one large block instead of many small ones
    - If not available (push failed or timed out) - falls back to fetching individual map outputs

### Configuration

- `spark.shuffle.push.enabled` (default `false`) - enable push-based shuffle
- `spark.shuffle.push.server.mergedShuffleFileManagerImpl` - ESS merge manager implementation
- `spark.shuffle.push.maxRetainedMergerLocations` (default $500$) - max ESS nodes tracked for merging
- `spark.shuffle.push.minShuffleSizeToWait` (default $500 MB$) - minimum shuffle size before waiting for push completion; small shuffles skip push
- `spark.shuffle.push.minCompletedPushRatio` (default $1.0$) - fraction of blocks that must be pushed before reduce stage starts; lower value allows reduce stage to start earlier at cost of more fallback reads

### Performance Characteristics

- Benefits -
    - Fewer, larger fetch requests on reduce side - better network utilization, lower connection overhead
    - Sequential reads from merged files - better disk I/O patterns vs random reads of many small files
    - Lower reduce task startup latency - merged files ready before reduce stage starts
- Costs -
    - Additional network I/O during map stage (push traffic)
    - ESS must have sufficient disk and memory for merge buffers
    - Added complexity; fallback logic required for push failures
- Best suited for - large shuffles with many map tasks where reduce-side fetch is the bottleneck
