# Spark Memory Management

## Unified Memory Manager

- __`UnifiedMemoryManager`__ - the default memory manager since Spark 1.6; manages a single unified memory pool shared between execution and storage with dynamic borrowing between them
- Replaced `StaticMemoryManager` which had fixed non-borrowable regions - the core problem was execution starving storage or vice versa with no recourse
- One `UnifiedMemoryManager` instance per executor JVM - also one on the Driver (for driver-side operations)
- Lives inside `SparkEnv` - initialized during `SparkContext` creation

### Memory Regions

- Total JVM heap is divided into three regions -
    - __Reserved memory__ - fixed $300 MB$ hardcoded in Spark source; holds Spark internal objects, framework overhead; not configurable; exists to prevent Spark from consuming all heap and starving the JVM itself
    - __User memory__ - `(heap - reserved) * (1 - spark.memory.fraction)`; default fraction is $0.6$ so user memory = $40%$ of usable heap; holds user data structures, UDF objects, RDD operator state that Spark doesn't manage - if you exceed this, you get OOM from user code not from Spark's memory manager
    - __Unified memory pool__ - `(heap - reserved) * spark.memory.fraction`; default $60%$ of usable heap; shared between execution and storage

- Python -
```python
    # With 4GB executor heap:
    # Reserved   = 300 MB (fixed)
    # Usable     = 4096 - 300 = 3796 MB
    # User       = 3796 * 0.4 = 1518 MB
    # Unified    = 3796 * 0.6 = 2278 MB
    # Storage    = 2278 * 0.5 = 1139 MB (initial soft boundary)
    # Execution  = 2278 * 0.5 = 1139 MB (initial soft boundary)
```

### Unified Pool Split

- Within the unified pool, `spark.memory.storageFraction` (default $0.5$) defines the initial soft boundary between storage and execution
- This is NOT a hard boundary - either side can grow into the other's space (borrowing)
- Storage gets `unifiedPoolSize * storageFraction` initially
- Execution gets the remainder initially
- The entire unified pool is available to execution if storage is empty, and vice versa

### Key Configuration

- `spark.memory.fraction` (default $0.6$) - fraction of usable heap allocated to unified pool
- `spark.memory.storageFraction` (default $0.5$) - fraction of unified pool initially given to storage
- `spark.memory.offHeap.enabled` (default `false`) - enable off-heap memory
- `spark.memory.offHeap.size` (default $0$) - total off-heap memory per executor in bytes

> [!NOTE]
> Increasing `spark.memory.fraction` gives more memory to Spark but less to user code (UDFs, Python workers, third-party libraries). The right value depends on workload - shuffle-heavy jobs benefit from more execution memory; cache-heavy jobs benefit from more storage memory. With unified memory, you rarely need to tune both separately.

---

## StaticMemoryManager (Legacy)

- __`StaticMemoryManager`__ - the original memory manager; fixed, non-overlapping regions for execution and storage; replaced by `UnifiedMemoryManager` in Spark 1.6
- Still available via `spark.memory.useLegacyMode=true` - present for backward compatibility but should never be used in new workloads
- Understanding it matters for interviews because it explains WHY `UnifiedMemoryManager` was designed the way it was

### Static Memory Layout

- Heap divided into fixed regions -
    - __Execution memory__ - `heap * spark.shuffle.memoryFraction` (default $0.2$) - used for shuffle, sort, aggregation buffers
    - __Storage memory__ - `heap * spark.storage.memoryFraction * spark.storage.safetyFraction` (default $0.6 * 0.9 = 0.54$) - used for cached RDDs, broadcast variables
    - __Other__ - remaining heap (~26%) - user code, framework overhead, task metadata
- These regions are completely separate - execution cannot borrow from storage and vice versa

### Problems with StaticMemoryManager

- __Execution starvation__ - if a job does heavy shuffling with small cached data, $20%$ execution memory is often insufficient; spills to disk even though storage region is mostly empty
- __Storage waste__ - if a job does no caching, $54%$ of heap sits unused while execution spills
- __Manual tuning burden__ - operators had to tune `spark.shuffle.memoryFraction` and `spark.storage.memoryFraction` per workload; wrong values caused OOM or wasted memory
- __Safety fractions on top of fractions__ - `spark.storage.safetyFraction` existed to avoid OOM from imprecise memory accounting; added another tuning knob with non-obvious interaction

### Why UnifiedMemoryManager Fixed This

- Single pool + dynamic borrowing eliminates the static waste problem
- Both sides can use the full pool when the other side is idle
- Only one primary knob (`spark.memory.fraction`) instead of multiple interacting fractions

---

## Execution Memory

- __Execution memory__ - the portion of unified memory used for computation during task execution
    - Shuffle write buffers
    - Sort buffers (`ExternalSorter`)
    - Aggregation hash maps (`ExternalAppendOnlyMap`, `Aggregator`)
    - Hash join build side (`HashedRelation`)
    - Window function buffers
- Managed by `UnifiedMemoryManager` via `ExecutionMemoryPool`
- __Per-task allocation__ - execution memory is further divided among active tasks to prevent one task from monopolizing all execution memory and starving others

### Per-Task Memory Allocation

- `UnifiedMemoryManager` tracks how many tasks are active on the executor
- Each task is entitled to at most `1/N` of execution memory where `N` = number of active tasks
- Each task is guaranteed at least `1/(2*N)` of execution memory - prevents starvation
- When a task requests memory and the pool is full -
    - If the requesting task is below its fair share - wait for other tasks to release memory
    - If the requesting task is above its fair share - spill
- This dynamic per-task allocation means memory entitlement changes as tasks start and finish on the executor

### Memory Acquisition Flow

- Task calls `TaskMemoryManager.acquireExecutionMemory(numBytes, consumer)` -
    - `TaskMemoryManager` delegates to `UnifiedMemoryManager.acquireExecutionMemory(numBytes, taskAttemptId, memoryMode)`
    - `UnifiedMemoryManager` checks available execution memory
    - If insufficient - attempts to borrow from storage pool (evicting cached blocks if needed)
    - If still insufficient - returns less than requested; task must decide whether to spill
- Execution memory is released when the task completes or explicitly calls `TaskMemoryManager.releaseExecutionMemory`

---

## Storage Memory

- __Storage memory__ - the portion of unified memory used for persisted data
    - Cached RDD partitions (`MEMORY_ONLY`, `MEMORY_AND_DISK`, `MEMORY_ONLY_SER`, etc.)
    - Broadcast variable data (stored as blocks in `BlockManager`)
    - Unroll memory - temporary memory used while deserializing a partition before storing it
- Managed by `UnifiedMemoryManager` via `StorageMemoryPool`
- Tracked at the block level via `BlockManager` - each cached block has a `BlockId` and a size estimate

### Unroll Memory

- Before storing a deserialized RDD partition in memory, Spark must "unroll" it - iterate through the iterator and collect elements into an array
- This requires temporary memory (unroll memory) before the final storage size is known
- Unroll memory is acquired from the storage pool
- If unroll fails (insufficient memory) - the partition is stored on disk instead (if `MEMORY_AND_DISK`) or dropped
- Unroll memory converts to storage memory once the partition is fully materialized
- `spark.storage.unrollMemoryThreshold` (default $1 MB$) - initial memory requested per partition for unrolling; grows as needed

### Storage Memory Eviction

- When storage memory is full and more is needed (either for new cached blocks or borrowed by execution) -
    - LRU eviction - least recently used blocks evicted first
    - Eviction target - blocks from RDDs with `MEMORY_AND_DISK` → spilled to disk; blocks from `MEMORY_ONLY` → dropped entirely
    - `BlockManagerMaster` notified of eviction so it can update the global block registry
    - If evicted block is `MEMORY_ONLY` - it is dropped; recomputed from lineage on next access
    - If evicted block is `MEMORY_AND_DISK` - written to disk; served from disk on next access

---

## Memory Borrowing & Eviction

- __Memory borrowing__ - the mechanism by which execution and storage can dynamically use each other's allocated space when one side has idle memory
- This is the core innovation of `UnifiedMemoryManager` over `StaticMemoryManager`

### Borrowing Rules - Asymmetric by Design

- __Execution borrowing from storage__ -
    - Execution CAN evict cached storage blocks to claim their memory
    - Eviction is forced - storage has no veto; blocks are evicted even if they were recently used
    - This is asymmetric by design - in-progress computation (execution) is more urgent than cached data (storage); losing cached data just causes recomputation; losing execution memory causes task failure
- __Storage borrowing from execution__ -
    - Storage CAN use idle execution memory when execution is not using it
    - BUT storage CANNOT evict in-progress execution memory - execution memory is protected while a task is running
    - When execution needs the memory back, storage must wait or spill its own data

### The Storage Safety Boundary

- `spark.memory.storageFraction` defines a __protected__ region within the unified pool for storage
- Storage blocks in this region CANNOT be evicted by execution
- Storage blocks ABOVE this region (borrowed execution space) CAN be evicted by execution
- This prevents execution from completely evicting all cached data - at least `storageFraction` of the pool is always available for storage

### Borrowing Flow - Execution Needs More Memory

1. Execution task requests memory via `TaskMemoryManager`
2. `UnifiedMemoryManager` checks execution pool - insufficient
3. Checks storage pool for memory above the storage safety boundary
4. If available - evicts LRU storage blocks from the borrowable region
5. Transfers freed memory to execution pool
6. Returns memory to requesting task
7. If still insufficient after eviction - task must spill

### Borrowing Flow - Storage Needs More Memory

1. `BlockManager` requests storage memory to cache a new partition
2. `UnifiedMemoryManager` checks storage pool - insufficient
3. Checks execution pool for idle memory (memory allocated to pool but not currently claimed by any task)
4. If available - transfers idle execution memory to storage pool
5. Caches the partition
6. When execution later needs that memory - storage must release it (evicting the cached block if necessary)

> [!NOTE]
> The asymmetry is critical - execution can forcibly reclaim memory from storage at any time; storage can only borrow from execution when execution is genuinely idle. This means caching is opportunistic - a cached RDD partition can be evicted at any time if execution needs memory. Design your jobs to tolerate cache eviction; never assume a `persist()`ed RDD will always be in memory.

### Eviction and Recomputation Cost

- When an evicted block is `MEMORY_ONLY` and gets dropped - next access triggers full lineage recomputation from source
- Long lineage chains make cache eviction extremely expensive - this is why `checkpoint()` is used to truncate lineage
- When an evicted block is `MEMORY_AND_DISK` - next access reads from disk; slower than memory but no recomputation
- For iterative algorithms - use `MEMORY_AND_DISK` to protect against eviction-induced recomputation

---

## TaskMemoryManager

- __`TaskMemoryManager`__ - per-task memory manager; the interface through which individual tasks request, track, and release memory
- One `TaskMemoryManager` instance per task attempt - created when the task starts, destroyed when it completes
- Sits between individual `MemoryConsumer`s (shuffle writers, sorters, aggregators) and the executor-level `MemoryManager`
- Tracks all `MemoryConsumer`s registered by the task - knows total memory granted to the task and to each consumer

### Responsibilities

- __Allocation__ - accepts memory requests from `MemoryConsumer`s; delegates to `UnifiedMemoryManager`; returns granted bytes (may be less than requested)
- __Spill coordination__ - when a new allocation fails, asks existing consumers to spill until enough memory is freed; calls `consumer.spill(size, trigger)` on registered consumers
- __Page management__ - for Tungsten on-heap mode, manages `MemoryBlock` pages (large contiguous memory chunks); tracks page table for pointer decoding
- __Accounting__ - tracks total memory used by the task; used by `UnifiedMemoryManager` for per-task fairness enforcement
- __Cleanup__ - on task completion, releases all memory back to `UnifiedMemoryManager`; calls `freePage` on all allocated pages

### Page Table and Addressing

- In Tungsten on-heap mode, `TaskMemoryManager` allocates memory in pages - large `long[]` arrays
- Each page has a page number (upper 13 bits of a 64-bit pointer) and an offset within the page (lower 51 bits)
- `encodePageNumberAndOffset(page, offset)` - encodes a `MemoryBlock` and offset into a single `long` pointer
- `decodePageNumber(pointer)` / `decodeOffset(pointer)` - decode the pointer back to page + offset
- Maximum pages per task - $8192$ (13-bit page number)
- Maximum page size - $2^{51}$ bytes (~$2$ PB theoretical; bounded by `spark.buffer.pageSize`, default $64 MB$)
- `spark.buffer.pageSize` - controls individual page size; larger pages = fewer allocations but more fragmentation waste

### Spill Triggering

- When `TaskMemoryManager.acquireExecutionMemory` cannot get enough memory from `UnifiedMemoryManager` -
    - Iterates registered `MemoryConsumer`s sorted by memory usage (largest first)
    - Calls `consumer.spill(requiredBytes, requestingConsumer)` on each until enough memory is freed
    - The requesting consumer itself can be asked to spill if it's the largest consumer
    - If spilling all consumers still isn't enough - throws `SparkOutOfMemoryError`

- Scala -
```scala
    // Internal usage pattern - not public API
    val taskMemoryManager = new TaskMemoryManager(memoryManager, taskAttemptId)

    // Consumer registers itself
    val sorter = new ExternalSorter(...)   // internally calls taskMemoryManager.registerConsumer

    // Acquire memory
    val granted = taskMemoryManager.acquireExecutionMemory(requiredBytes, consumer)
    if (granted < requiredBytes) {
        consumer.spill(requiredBytes - granted, consumer)
    }

    // Release on task completion - called automatically
    taskMemoryManager.cleanUpAllAllocatedMemory()
```

---

## MemoryConsumer

- __`MemoryConsumer`__ - abstract base class for any Spark component that acquires and holds execution memory
- Concrete implementations - `ExternalSorter`, `ExternalAppendOnlyMap`, `UnsafeExternalSorter`, `BytesToBytesMap`, `ShuffleExternalSorter`
- Each consumer registers with its task's `TaskMemoryManager` on construction
- Must implement `spill(size, trigger)` - called by `TaskMemoryManager` when memory pressure requires the consumer to free memory

### MemoryConsumer Interface

- `used` - current bytes of execution memory held by this consumer
- `spill(size, trigger)` -
    - Called when `TaskMemoryManager` needs the consumer to free at least `size` bytes
    - `trigger` is the `MemoryConsumer` that triggered the spill request (may be `this` or another consumer)
    - Must write in-memory data to disk and release the corresponding memory
    - Returns actual bytes freed
    - Not all consumers can spill - those that can't throw `IOException`
- `allocateArray(size)` / `freeArray(array)` - allocate/free `LongArray` backed by a `MemoryBlock`
- `allocatePage(required)` / `freePage(page)` - allocate/free a `MemoryBlock` page via `TaskMemoryManager`

### Spill Implementation Pattern

- A well-behaved `MemoryConsumer.spill` implementation -
    1. Sorts or serializes in-memory data
    2. Opens a spill file via `SpillWriter`
    3. Writes data to disk
    4. Calls `freeMemory(bytesFreed)` to release memory back to `TaskMemoryManager`
    5. Clears in-memory data structures
    6. Records spill metadata (file path, offsets) for later merge
- On final iteration, merges all spill files with any remaining in-memory data using external merge-sort

### Consumers and Their Spill Behavior

- __`ExternalSorter`__ - used by `SortShuffleWriter` and `sortByKey`; spills sorted runs to disk; merges on output
- __`ExternalAppendOnlyMap`__ - used by `CoGroupedRDD`, `aggregateByKey`; spills sorted key-value pairs; merge-sorts on output
- __`UnsafeExternalSorter`__ - Tungsten-aware version of `ExternalSorter`; operates on `UnsafeRow` binary data; spills serialized binary records
- __`BytesToBytesMap`__ - Tungsten hash map used by `HashedRelation` for broadcast hash joins; does NOT support spill - throws if memory exhausted (broadcast join falls back to sort-merge join in this case)
- __`ShuffleExternalSorter`__ - used by `UnsafeShuffleWriter`; operates on serialized binary records; spills to shuffle spill files

---

## MemoryPool

- __`MemoryPool`__ - abstract base class representing a pool of memory with a tracked size and used amount
- Two concrete implementations in `UnifiedMemoryManager` -
    - __`ExecutionMemoryPool`__ - tracks execution memory; handles per-task allocation with fairness enforcement
    - __`StorageMemoryPool`__ - tracks storage memory; handles block caching and LRU eviction

### ExecutionMemoryPool

- Tracks `_memoryUsed` (total execution memory currently granted across all tasks) and `_poolSize` (current pool capacity)
- Per-task tracking via `memoryForTask: HashMap[Long, Long]` - maps `taskAttemptId` to bytes granted
- `acquireMemory(numBytes, taskAttemptId)` -
    - Computes per-task fair share based on active task count
    - If task is below minimum entitlement - blocks (waits on a lock) until memory is available
    - Implements a retry loop with `wait()` / `notifyAll()` for tasks blocked on memory
- `releaseMemory(numBytes, taskAttemptId)` - decrements task's allocation; calls `notifyAll()` to wake waiting tasks

### StorageMemoryPool

- Tracks `_memoryUsed` and `_poolSize`
- `acquireMemory(blockId, numBytes)` - attempts to allocate storage memory; triggers eviction if needed
- `acquireMemoryForUnroll(blockId, numBytes)` - separate path for unroll memory acquisition
- `freeMemory(numBytes)` - releases memory back to pool
- `releaseMemoryForThisBlock(blockId)` - releases memory for a specific block (called on eviction)

### Pool Size Adjustment

- `UnifiedMemoryManager` adjusts pool sizes dynamically when borrowing occurs -
    - `storagePool.decrementPoolSize(delta)` + `executionPool.incrementPoolSize(delta)` - transfer storage space to execution
    - `executionPool.decrementPoolSize(delta)` + `storagePool.incrementPoolSize(delta)` - transfer execution space to storage
- These are atomic operations under a synchronized lock to prevent race conditions between concurrent tasks

---

## Off-Heap Memory

- __Off-heap memory__ - memory allocated outside the JVM heap using `sun.misc.Unsafe`; not subject to JVM garbage collection
- Enabled via `spark.memory.offHeap.enabled=true` and `spark.memory.offHeap.size` (per executor, in bytes)
- When enabled, `UnifiedMemoryManager` manages two unified pools - one on-heap, one off-heap
- Tungsten preferentially uses off-heap when available

### Why Off-Heap

- __GC elimination__ - on-heap memory triggers GC; large long-lived objects (shuffle buffers, sort buffers) cause major GC pauses; off-heap memory is invisible to GC
- __Predictable memory__ - off-heap size is exact; JVM heap size is approximate due to object overhead, GC headroom, fragmentation; off-heap gives precise control
- __Larger effective memory__ - JVM heap has practical limits (~32 GB for compressed OOPs; beyond that pointer size doubles); off-heap can use hundreds of GB
- __Unsafe direct access__ - `sun.misc.Unsafe` allows direct memory reads/writes at C-like speeds; no bounds checking, no object header overhead

### Off-Heap Memory Layout in Spark

- Off-heap unified pool split same as on-heap - `storageFraction` for storage, remainder for execution
- Execution off-heap - used by Tungsten operators (`UnsafeExternalSorter`, `BytesToBytesMap`) when `spark.memory.offHeap.enabled=true`
- Storage off-heap - used for cached blocks when storage level is `OFF_HEAP`

### sun.misc.Unsafe Operations

- `allocateMemory(bytes)` - allocates off-heap memory; returns raw address as `long`
- `freeMemory(address)` - frees off-heap allocation; MUST be called manually - no GC
- `copyMemory(srcAddr, dstAddr, bytes)` - bulk memory copy; much faster than Java array copies
- `getLong(address)` / `putLong(address, value)` - direct read/write at memory address
- Memory leaks - off-heap memory is NOT freed on JVM crash; must be freed explicitly; Spark registers shutdown hooks and task completion listeners to free off-heap allocations

### Off-Heap Configuration

- Python -
```python
    spark = SparkSession.builder \
        .config("spark.memory.offHeap.enabled", "true") \
        .config("spark.memory.offHeap.size", str(2 * 1024 * 1024 * 1024)) \  # 2 GB
        .getOrCreate()
```

> [!NOTE]
> Off-heap memory is NOT counted in `spark.executor.memory` - it is additional memory on top of the JVM heap. Total executor memory = `spark.executor.memory` (heap) + `spark.memory.offHeap.size` (off-heap) + `spark.executor.memoryOverhead` (native overhead). Container memory limits must account for all three.

> [!TIP]
> Off-heap is most valuable for -
>   - Executors with large heaps (> 16 GB) where GC pauses are significant
>   - Shuffle-heavy jobs where execution memory buffers are large and long-lived
>   - Jobs with many concurrent tasks where GC pressure from multiple sort buffers compounds

---

## Tungsten Overview

- __Tungsten__ - Spark's internal execution engine overhaul introduced in Spark 1.4; operates below the DataFrame/RDD API level; invisible to user code but responsible for most of Spark's performance gains since 1.4
- Three pillars -
    - __Binary memory format__ - stores data as compact binary rows (`UnsafeRow`) instead of JVM objects; eliminates object overhead, reduces GC pressure, enables direct memory operations
    - __Cache-aware computation__ - algorithms designed around CPU cache lines; sorts and hash maps operate on binary data laid out for sequential memory access
    - __Code generation__ - Spark generates bytecode at runtime for expression evaluation and operator pipelines; eliminates virtual dispatch, interpreter overhead, and boxing/unboxing

### What Tungsten Replaced

- Before Tungsten, Spark operated on JVM objects -
    - A simple `(String, Int)` tuple required multiple objects with headers, references, padding
    - Object graph traversal for operations caused random memory access patterns → cache misses
    - GC had to scan all live objects including data objects → long GC pauses proportional to data size
    - Serialization required converting object graph to bytes → CPU overhead on every shuffle

### Tungsten Memory Modes

- __On-heap__ - `sun.misc.Unsafe` operates on `long[]` arrays allocated on the JVM heap; GC-visible but avoids object overhead within the array; default mode
- __Off-heap__ - `sun.misc.Unsafe` operates on raw memory addresses; completely GC-invisible; requires `spark.memory.offHeap.enabled=true`
- Both modes use the same binary row format (`UnsafeRow`) - the only difference is the memory backing

### Scope of Tungsten

- Affects all DataFrame/Dataset operations and SQL queries - anything that goes through Catalyst's physical plan
- Does NOT affect plain RDD operations - `RDD[MyObject]` still uses JVM objects
- This is a primary reason to prefer DataFrame/Dataset over RDD for performance-critical code

---

## Tungsten Binary Row Format

- __Binary row format__ - Tungsten's compact in-memory representation for a row of data; designed to be operated on directly without deserialization
- A row is a contiguous sequence of bytes laid out in a fixed schema
- Three sections per row -
    - __Null bitmap__ - one bit per field; indicates whether each field is null; padded to 8-byte alignment
    - __Fixed-length values section__ - 8 bytes per field; stores fixed-length types (int, long, double, float, boolean, date, timestamp) directly; stores offset+length for variable-length types (string, binary, array, map)
    - __Variable-length data section__ - stores the actual bytes for strings, byte arrays, arrays, maps; referenced by offset+length pointers in the fixed section

### Row Layout Example

- Schema - `(id: Int, name: String, score: Double)` -
    - Bytes $0-7$   : Null bitmap ($1 byte used, 7 padding$) - $bit 0=id null, bit 1=name null, bit 2=score null$
    - Bytes $8-15$  : id value (stored as long, $upper 4 bytes zero$)
    - Bytes $16-23$ : name offset+length ($upper 4 bytes = offset into variable section, lower 4 = length$)
    - Bytes $24-31$ : score value ($8-byte IEEE 754 double$)
    - Bytes $32+$   : variable-length section - "Alice" bytes ($5 bytes + 3 padding to 8-byte align$)

### Why This Layout

- __Fixed-size field section__ - field access is O(1) by index; no scanning; `row.getLong(fieldIndex)` = read 8 bytes at `baseOffset + 8 + fieldIndex * 8`
- __8-byte alignment__ - all fixed fields 8-byte aligned; enables `sun.misc.Unsafe` to read/write in single instructions without alignment faults
- __Contiguous memory__ - entire row in one contiguous block; no pointer chasing; CPU prefetcher can load entire row into cache in one or few cache line fetches
- __No deserialization__ - operations like `filter`, `project`, `sort` operate directly on binary bytes; a filter on `id > 100` reads 8 bytes at a fixed offset - no object creation

### Comparison with JVM Object Representation

| Aspect | JVM Object | Tungsten Binary |
| --- | --- | --- |
| Object header | 16 bytes per object | None |
| Int storage | 16 bytes (Integer object) | 8 bytes (in fixed section) |
| String storage | 40+ bytes (object + char array) | 4+N bytes (length + UTF-8 bytes) |
| Field access | Virtual dispatch + pointer chase | Direct offset calculation |
| GC visibility | Every field scanned | Only the backing array/address |
| Null handling | `null` reference | Bit in null bitmap |
| Serialization | Java serialization overhead | Already binary - copy directly |

---

## UnsafeRow

- __`UnsafeRow`__ - Spark's concrete implementation of `InternalRow` using Tungsten binary format; the row object used throughout the physical execution layer in DataFrame/SQL operations
- Not a data container itself - it's a pointer + schema wrapper; it holds a reference to a memory region (on-heap `Object` + offset, or off-heap address) and interprets it as a row
- Mutable - `UnsafeRow` can be written to (used by `UnsafeRowWriter`) and read from
- Reused across records in tight loops - the same `UnsafeRow` object is repositioned to point at each successive record; avoids per-record object allocation

### UnsafeRow Fields

- `baseObject` - the backing `Object` (a `byte[]` or `long[]` for on-heap; `null` for off-heap)
- `baseOffset` - byte offset within `baseObject` where this row starts (or absolute address for off-heap)
- `sizeInBytes` - total byte size of this row
- `numFields` - number of fields in the schema

### Field Access Methods

- `getLong(ordinal)` - reads 8 bytes at `baseOffset + 8 + ordinal * 8`; O(1)
- `getInt(ordinal)` - reads lower 4 bytes of the 8-byte fixed slot
- `getDouble(ordinal)` - reads 8 bytes and reinterprets as IEEE 754 double
- `getUTF8String(ordinal)` - reads offset+length from fixed slot, returns `UTF8String` pointing into variable section
- `isNullAt(ordinal)` - checks bit `ordinal` in null bitmap
- `setLong(ordinal, value)` / `setInt` / `setDouble` etc - writes to fixed slot
- `setNullAt(ordinal)` - sets bit in null bitmap, zeroes fixed slot

### UnsafeRowWriter

- __`UnsafeRowWriter`__ - helper for constructing `UnsafeRow` instances
- Writes fields in order; handles null bitmap, fixed section, variable-length section
- Automatically grows the backing buffer if variable-length data exceeds initial estimate

- Scala -
```scala
    import org.apache.spark.sql.catalyst.expressions.codegen.{UnsafeRowWriter, BufferHolder}

    val writer = new UnsafeRowWriter(3)     // 3 fields
    writer.reset()
    writer.write(0, 42L)                    // id = 42
    writer.write(1, UTF8String.fromString("Alice"))
    writer.write(2, 95.5)                   // score
    val row: UnsafeRow = writer.getRow
```

### UnsafeRow in the Execution Pipeline

- Physical operators receive and emit `UnsafeRow` -
    - `FileScan` reads bytes from Parquet/ORC and writes directly into `UnsafeRow` without creating JVM objects
    - `Filter` checks a field via `row.getLong(0) > 100` - one memory read, no deserialization
    - `Project` copies selected fields into a new `UnsafeRow`
    - `SortMergeJoin` compares rows by reading fixed-offset fields - no object creation during comparison
    - `ShuffleWriter` serializes an `UnsafeRow` by copying its raw bytes - serialization IS the copy
- Code generation (`WholeStageCodegen`) generates code that operates directly on `UnsafeRow` fields - the generated code has no virtual dispatch, no boxing

### Copying UnsafeRow

- `UnsafeRow.copy()` - creates a new `UnsafeRow` with a fresh backing byte array; used when the row must outlive the current iteration (eg - storing in a hash map, returning from an operator)
- Without copy, the backing memory may be overwritten by the next record - a common source of subtle bugs when manually working with `UnsafeRow`

> [!NOTE]
> `UnsafeRow` is the reason `Dataset[Row]` (DataFrame) is faster than `RDD[Row]` even though both work with rows. `RDD[Row]` uses `GenericRow` - a JVM object wrapping an `Array[Any]`. `DataFrame` uses `UnsafeRow` - binary bytes with direct field access. The difference is GC pressure, object overhead, and cache efficiency.

---

## Spill Mechanics

- __Spilling__ - the process of writing in-memory data to disk when memory is insufficient to hold it all; allows operations that require more memory than available to complete at the cost of disk I/O
- Triggered when a `MemoryConsumer` cannot acquire enough memory from `TaskMemoryManager`
- Every major memory-consuming operator supports spilling - `ExternalSorter`, `ExternalAppendOnlyMap`, `UnsafeExternalSorter`, `ShuffleExternalSorter`
- Spill files written to `spark.local.dir` - should be on fast local disk (SSD preferred)

### Spill Decision Flow

1. `MemoryConsumer` calls `TaskMemoryManager.acquireExecutionMemory(needed)`
2. `TaskMemoryManager` cannot satisfy the request from the execution pool
3. `TaskMemoryManager` asks other consumers on the same task to spill via `consumer.spill(size, trigger)`
4. If still insufficient, the requesting consumer itself is asked to spill
5. Consumer serializes its in-memory data, writes to a spill file, releases memory
6. Spill file metadata (path, record count, byte offsets) recorded for later merge

### ExternalSorter Spill

- __`ExternalSorter`__ - used by `SortShuffleWriter` and `sortByKey`; handles spilling for sort operations
- In-memory structure - `PartitionedAppendOnlyMap` (if aggregating) or `PartitionedPairBuffer` (if not)
- Spill process -
    1. Sort in-memory records by `(partitionId, key)` using TimSort
    2. Open a `DiskBlockObjectWriter` to a spill file in `spark.local.dir`
    3. Write sorted records to file in batches; flush and close
    4. Record `SpilledFile` metadata - path, partition boundaries, element counts per partition
    5. Clear in-memory structure; release memory
- Merge process (at output time) -
    1. Sort remaining in-memory records
    2. Open all spill files
    3. K-way merge using a priority queue (min-heap on `(partitionId, key)`)
    4. Write merged output to final shuffle file

### ExternalAppendOnlyMap Spill

- Used by `CoGroupedRDD`, `aggregateByKey`, and any operator using a hash-based combine
- In-memory structure - `SizeTrackingAppendOnlyMap` (hash map with size estimation)
- Spill process -
    1. Sort in-memory map by key hash (not key value - for speed)
    2. Write sorted key-value pairs to spill file
    3. Clear map, release memory
- Merge process -
    1. Open all spill files
    2. K-way merge by key (using `Ordering[K]`)
    3. For each key, apply `mergeCombiners` across all values from all spill files and in-memory map
    4. Emit merged result

### UnsafeExternalSorter Spill

- Tungsten-aware version of `ExternalSorter`; operates on `UnsafeRow` binary data
- In-memory structure - `LongArray` of pointers into `MemoryBlock` pages (Tungsten memory)
- Spill process -
    1. Sort the pointer array by `(partitionId, key)` - comparison reads directly from binary rows via pointers
    2. Write raw bytes of each `UnsafeRow` to spill file (copy from Tungsten pages to disk)
    3. Free Tungsten pages; release memory
- Key difference from `ExternalSorter` - operates on binary data throughout; no serialization/deserialization during sort; rows are already in wire format

### Spill File Management

- Spill files are temporary - created in `spark.local.dir`, deleted after merge
- Each spill creates one file; a single task can create many spill files if memory is repeatedly exhausted
- `spark.shuffle.spill.compress` (default `true`) - compress spill files; reduces disk I/O at cost of CPU
- `spark.shuffle.spill.diskWriteBufferSize` (default $1 MB$) - write buffer for spill file I/O

### Spill Metrics in Spark UI

- `Spill (memory)` - total bytes spilled from memory (size of in-memory data before spill)
- `Spill (disk)` - total bytes written to disk (may be smaller than memory spill due to compression)
- Large gap between memory spill and disk spill → compression is effective
- High spill numbers → insufficient execution memory; tune `spark.memory.fraction`, increase executor memory, or reduce partition size

### Spill Avoidance Strategies

- __Increase execution memory__ - increase `spark.executor.memory`, increase `spark.memory.fraction`, or reduce storage fraction
- __Reduce partition size__ - more, smaller partitions means each task handles less data and needs less memory
- __Use `reduceByKey` over `groupByKey`__ - map-side pre-aggregation reduces data before it hits the in-memory hash map
- __Use `mapPartitions` with manual batching__ - control exactly how much data is in memory at once
- __Enable off-heap__ - off-heap memory doesn't GC; more memory available for execution without GC pressure increasing
- __Increase `spark.shuffle.file.buffer`__ - larger write buffer reduces flush frequency during spill

> [!TIP]
> Spill is not always bad - it's better to spill than to OOM. A job that spills occasionally and completes is better than a job that OOMs. Design for spill tolerance by using `MEMORY_AND_DISK` storage levels and operators that support spill (`ExternalSorter`, `ExternalAppendOnlyMap`) rather than trying to eliminate all spills at the cost of correctness.

> [!NOTE]
> The spill-merge cycle is why increasing `numPartitions` can actually speed up a job even if it creates more tasks. Fewer records per partition = less memory per task = less spilling = faster per-task execution. The scheduler overhead of more tasks is often much cheaper than the disk I/O cost of spilling.