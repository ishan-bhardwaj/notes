# Spark Memory Management

## Unified Memory Manager

- Default memory manager since Spark 1.6
- Single unified pool shared between execution memory and storage memory
- Replaced `StaticMemoryManager` which used fixed non-borrowable regions
- One instance per executor JVM, and one on driver
- __Heap Layout__ -
    - JVM heap divided into -
        - Reserved memory - fixed $300MB$ for Spark/JVM internals
        - User memory - unmanaged user objects, UDF state, Python objects
        - Unified memory - managed by Spark for execution + storage
- Key configs - 
    - `spark.memory.fraction` (default $0.6$) - fraction of usable heap for Spark-managed memory
    - `spark.memory.storageFraction` (default $0.5$) - fraction of unified pool initially given to storage
    - `spark.memory.offHeap.enabled` (default `false`) - enable off-heap memory
    - `spark.memory.offHeap.size` (default $0$) - total off-heap memory per executor in bytes
- $4GB$ Executor heap example -
    - Reserved = $300MB$
    - Usable = $4096 - 300 = 3796MB$
    - Unified = $3796 * 0.6  = 2278MB$ (`spark.memory.fraction=0.6`)
        - Initial storage = $1139MB$ ($0.5$ * `spark.memory.fraction`)
        - Initial execution = $1139MB$
    - User = $3796 * 0.4  = 1518MB$

### Borrowing Model
    
- Execution can evict storage blocks
- Storage can borrow only idle execution memory
- Execution memory is prioritized over caching

## StaticMemoryManager (Legacy)

- Old pre-Spark-1.6 memory manager
- Used fixed non-overlapping regions
- Enabled via - `spark.memory.useLegacyMode=true`
- Problems -
    - Shuffle-heavy jobs spilled despite free cache memory
    - Cache-heavy jobs wasted execution memory
    - Required manual tuning of multiple fractions
- How `UnifiedMemoryManager` fixed this -
    - - Single pool + dynamic borrowing
    - Both sides can use the full pool when the other side is idle
    - Only one primary knob (`spark.memory.fraction`) instead of multiple interacting fractions

## Execution Memory

- Memory used during computation -
    - Shuffle buffers
    - Sort buffers
    - Aggregation maps
    - Hash join build side
    - Window buffers
- Managed by `ExecutionMemoryPool`
- __Per-Task Fairness__ -
    - Active tasks share execution memory
    - If `N` = active tasks on executor then each task gets roughly -
        - $Max ≈ 1/N$
        - $Guaranteed minimum ≈ 1/(2N)$
- Allocation flow -
    ```
    Task
    -> TaskMemoryManager
    -> UnifiedMemoryManager
    -> ExecutionMemoryPool
    ```

- If memory is unavailable - 
    ```
    Request memory
    -> Borrow from storage if possible
    -> Evict cached blocks
    -> Spill if still insufficient
    ```

## Storage Memory

- Memory used for persisted data -
    - Cached RDD partitions
    - Broadcast blocks
    - Unroll memory
- Managed by `StorageMemoryPool`

- __Unroll Memory__ -
    - Before storing a deserialized RDD partition in memory, Spark must "unroll" it - iterate through the iterator and collect elements into an array
    - Unroll memory is acquired from the storage pool
    - Failure may -
        - Spill to disk
        - Drop cache attempt
    - `spark.storage.unrollMemoryThreshold` -
        - Initial unroll allocation
        - Default $1MB$

- __Eviction__ -
    - LRU eviction policy -
    - `MEMORY_ONLY` - block dropped, recomputed later
    - `MEMORY_AND_DISK` - block spilled to disk

## Memory Borrowing & Eviction

- Execution can -
    - Evict cached blocks
    - Reclaim storage memory immediately
- Storage - 
    - Can use idle execution memory
    - Cannot evict active execution memory
- __Storage Safety Boundary__ -
    - Defined by `spark.memory.storageFraction`
    - Storage below this boundary is protected
    - Storage above this boundary is borrowable/evictable

> [!NOTE]
> `persist()` is opportunistic -
>   - Cached blocks may disappear anytime under execution pressure
>   - Never assume cached data will remain memory-resident.

## TaskMemoryManager

- Per-task memory manager
- One instance per task attempt
- Coordinates memory allocation/spill for task consumers
- Responsibilities -
    - Execution memory allocation
    - Spill coordination
    - Page management
    - Task-level accounting
    - Cleanup on task completion

### Spill Coordination

- If memory request fails -
    ```
    TaskMemoryManager
    -> ask largest consumers to spill
    -> free memory
    -> retry allocation
    ```

- If spilling is insufficient - `SparkOutOfMemoryError`

### Tungsten Page Model

- Memory is allocated in pages (`long[]` arrays)
- Each page has -
    - Page number - upper $13 bits$ of $64-bit$ pointer
    - Offset inside the page - lower $15 bits$
- `spark.buffer.pageSize` -
    - Tungsten page size
    - Default $64MB$

## MemoryConsumer

- Abstract base class for execution-memory users
- Implemented by - 
    - `ExternalSorter`
    - `ExternalAppendOnlyMap`
    - `UnsafeExternalSorter`
    - `BytesToBytesMap`
- Must implement - `spill(size, trigger)` - called when Spark needs memory back
- Typical spill flow -
    1. Sort / serialize in-memory data
    2. Open a spill file via `SpillWriter` and write spill file to disk
    3. Call `freeMemory(bytesFreed)` to release memory back to `TaskMemoryManager`
    4. Record spill metadata (file path, offsets) for later merge

## MemoryPool

- Abstract base class for a tracked memory region
- Two implementations -
    - `ExecutionMemoryPool` - tracks execution memory
    - `StorageMemoryPool` - tracks storage memory

### ExecutionMemoryPool

- Tracks -
    - Total pool size
    - Total memory used
    - Memory used per active task
- If a task is below its minimum fair share, it may block and wait for memory
- Uses `wait()` / `notifyAll()` to coordinate tasks waiting for memory

### StorageMemoryPool

- Tracks storage memory used by cached blocks
- Handles -
    - Cached RDD blocks
    - Broadcast blocks
    - Unroll memory
    - Block eviction

### Dynamic Pool Resizing

- Unified memory dynamically transfers space between storage and execution pools
    - `storagePool.decrementPoolSize(delta)`
    - `executionPool.incrementPoolSize(delta)`
    - Or the reverse
- These pool-size changes are atomic and protected by synchronized locks
- This prevents race conditions when multiple executor task threads request memory concurrently

## Off-Heap Memory

- Memory allocated outside JVM heap using `sun.misc.Unsafe`
- Managed by Spark but not scanned by JVM GC
- Enabled with -
    ```python
    spark.memory.offHeap.enabled=true
    spark.memory.offHeap.size=<bytes>
    ```

- Tungsten preferentially uses off-heap when available
- Why Off-Heap Matters -
    - Reduces GC pressure from large execution buffers.
    - Useful for shuffle-heavy, sort-heavy, and aggregation-heavy workloads
    - Helps when executor heaps are large and GC pauses become expensive
    - Works well with Tungsten binary memory layout

> [!NOTE]
> Total executor container memory is closer to -
>   ```
>   spark.executor.memory
>   + spark.memory.offHeap.size
>   + spark.executor.memoryOverhead
>   ```
>
> If the container limit does not include all of these, the executor can be killed by YARN/Kubernetes even if JVM heap looks fine

## Tungsten

- Tungsten is Spark’s low-level execution engine improvement
- Does not optimize plain object-based `RDD[MyClass]` workloads
- Three main ideas -
    - Binary row format instead of JVM object graphs - stores data as compact binary rows (`UnsafeRow`)
    - Cache-aware memory layout - algorithms designed around CPU cache lines
    - Runtime code generation - Spark generates bytecode at runtime for expression evaluation and operator pipelines

### Tungsten Binary Row Format

- Spark SQL stores rows as compact binary data
- Row is a contiguous sequence of bytes laid out in a fixed schema
- Three sections per row -
    - __Null bitmap__ - 
        - One bit per field
        - Indicates whether each field is null
        - Padded to $8-byte$ alignment
    - __Fixed-length values area__ - 
        - $8 bytes$ per field
        - Stores fixed-length types (`int`, `long`, `double`, `float`, `boolean`, `date`, `timestamp`) directly
        - Stores $offset + length$ for variable-length types
    - __Variable-length data area__ - 
        - Stores the actual bytes for `strings`, `byte` `array`, `map`
        - Referenced by $offset + length$ pointers in the fixed section
- Why this is fast -
    - Field access is offset-based and `O(1)`
    - No pointer chasing
    - Fewer JVM objects
    - Less GC pressure
    - Efficient CPU cache usage
    - Shuffle/sort can copy binary bytes directly
- Practical Mental Model -
    - A filter like `id > 10` becomes -
        - Read $8 bytes$ at fixed offset
        - Compare
    - Instead of -
        - Deserialize object
        - Call getter
        - Unbox value
        - Compare

## UnsafeRow

- Spark's concrete implementation of `InternalRow` using Tungsten binary format
- Used throughout the physical execution layer in DataFrame/SQL operations
- Holds a reference to a memory region (on-heap `Object` + offset, or off-heap address) and interprets it as a row
- Core fields -
    - `baseObject` - backing object for on-heap memory, or null for off-heap
    - `baseOffset` - start offset/address of the row, or absolute address for off-heap
    - `sizeInBytes` - total byte size of this row
    - `numFields` - number of fields in the schema
- Fast access -
    - Direct offset reads against binary memory
    - Eg -
        ```
        row.getLong(0)
        row.getDouble(1)
        row.isNullAt(2)
        ```

> [!NOTE]
> `UnsafeRow` is mutable and often reused
>
> If you need to keep a row beyond the current iteration, copy it - `val safe = row.copy()`
>
> Otherwise the same backing memory may be overwritten by the next record

> [!TIP]
> Physical operators pass around `UnsafeRow` - most work happens without converting rows into normal JVM objects

> [!TIP]
> `UnsafeRow` is the reason `Dataset[Row]` / `DataFrame` are faster than `RDD[Row]` even though both work with rows
>   - `RDD[Row]` uses `GenericRow` - a JVM object wrapping an `Array[Any]`
>   - `DataFrame` uses `UnsafeRow` - binary bytes with direct field access with less GC pressure than object-based RDD code

## Spill Mechanics

- Writing in-memory execution state to local disk when memory is insufficient
- It prevents task OOM at the cost of disk I/O and later merge cost
- Spill files are written under `spark.local.dir`
- When spill happens -
    ```
    MemoryConsumer requests memory
    -> TaskMemoryManager asks UnifiedMemoryManager
    -> memory unavailable
    -> existing consumers are asked to spill
    -> memory is freed
    -> task continues
    ```

### Spillable Structures

- `ExternalSorter` - 
    - Used for sort and shuffle sort paths
    - Spill process -
        - Sorts in-memory records `(partitionId, key)` using TimSort
        - Writes sorted run to spill file in batches
        - Clears memory
        - Later performs k-way merge using a priority queue (min-heap on `(partitionId, key)`) across spill files and remaining memory
        - Write merged output to final shuffle file
- `ExternalAppendOnlyMap` - 
    - Used for aggregations and cogroup-style operations
    - Spill process -
        - Sort in-memory map by key hash
        - Write sorted key-value pairs to spill file
        - Clears memory
        - K-way merge by key (using `Ordering[K]`)
        - Write merged output to final shuffle file
- `UnsafeExternalSorter` -
    - Tungsten-aware sorter for binary rows - operates on `UnsafeRow` binary data
    - Sorts pointer array rather than full JVM objects
    - Writes raw binary row bytes to disk
    - Spill process -
        - Sort the pointer array by `(partitionId, key)` - comparison reads directly from binary rows via pointers
        - Write raw bytes of each `UnsafeRow` to spill file (copy from Tungsten pages to disk)
        - Free Tungsten pages and release memory

### Spill File Management

- Spill files are temporary
- Created in `spark.local.dir` and deleted after successful merge/output
- Many spills from one task usually indicate partition too large or memory too low
- Key configs -
    - `spark.shuffle.spill.compress` -
        - Compress spill files
        - Reduces disk I/O, costs CPU
    - `spark.shuffle.spill.diskWriteBufferSize` -
        - Buffer size for spill file writes
    - `spark.shuffle.file.buffer` -
        - Map-side shuffle write buffer
- Spark UI Spill Metrics -
    - `Spill (memory)` - logical size of data spilled from memory
    - `Spill (disk)` - actual bytes written to disk
    - If memory spill is much larger than disk spill, compression is helping

### Avoiding / Reducing Spills

- Increase executor memory
- Increase execution memory share if appropriate
- Increase number of partitions to reduce per-task data
- Prefer `reduceByKey` over `groupByKey`
- Avoid skewed partitions.
- Use map-side combine where possible
- Use `MEMORY_AND_DISK` when cache eviction would cause expensive recomputation
- Put `spark.local.dir` on fast disks

> [!TIP]
> Spill is not always bad - 
>   - Small controlled spill - acceptable
>   - Repeated large spill - performance bottleneck
>   - OOM avoidance spill - better than task failure

> [!NOTE]
> Increasing partition count can improve performance even with more tasks -
> ```
> More partitions
> -> Smaller per-task working set
> -> Fewer spills
> -> Less disk merge cost
> ```