# Storage & Block Management

## BlockManager

- `BlockManager` - 
    - Storage engine on every Driver and Executor
    - One instance per JVM process - on each executor and on driver
    - Lives inside `SparkEnv`
    - Manages all Spark blocks -
        - Cached RDD partitions
        - Shuffle data
        - Broadcast variables
        - Spill files
    - Every stored object is a block identified by `BlockId`

### Managed Block Types

- RDD Blocks (`RDDBlockId`) - cached partitions
- Shuffle Blocks (`ShuffleBlockId`) - shuffle outputs
- Broadcast Blocks (`BroadcastBlockId`) - broadcast variables
- Temporary Shuffle Blocks (`TempShuffleBlockId`) - temporary shuffle data
- Stream Blocks (`StreamBlockId`) - Structured Streaming blocks
- Core Components -
    - `MemoryStore` - in-memory storage (JVM heap or off-heap)
    - `DiskStore` - local disk storage
    - `DiskBlockManager` - file placement and paths
    - `BlockManagerMaster` - cluster-wide block metadata
    - `NettyBlockTransferService` - block transfer over network
    - `MemoryManager` - storage memory allocation
    - `SerializerManager` - serialization and compression

### Block Lifecycle

- Store path -
    - Memory first
    - Disk fallback if storage level allows
    - Optional replication to remote executors
- Lookup path -
    - `MemoryStore` first
    - `DiskStore`
    - Remote executor fetch via `NettyBlockTransferService`
- Every add/remove operation reported to `BlockManagerMaster`

> [!NOTE]
> `BlockManager` is the storage layer, and `BlockManagerMaster` is only metadata
>
> Actual block bytes never pass through the Driver

### BlockManagerMaster

- Driver-side metadata coordinator
- Runs as `BlockManagerMasterEndpoint`
- Maintains cluster-wide block registry
- Tracks -
    - Executor registrations
    - Block locations
    - Memory usage
    - Replication metadata
- __Global Block Registry__ -
    - `blockManagerInfo: Map[BlockManagerId, BlockManagerInfo]` -
        - Maps each executor's `BlockManagerId` to its info (max memory, used memory, block list)
    - `blockLocations: Map[BlockId, Set[BlockManagerId]]` -
        - Maps each `BlockId` to the set of executors that hold it
        - Used by `DAGScheduler` to compute task preferred locations

### BlockManagerSlave

- Executor-side RPC endpoint
- Receives commands from Driver

## MemoryStore

- Fastest storage tier
- Stores blocks in JVM heap or off-heap memory
- Backed by LRU-ordered `LinkedHashMap`
- Storage Types -
    - `DeserializedMemoryEntry`
        - Used by `MEMORY_ONLY`
        - Stores JVM objects
    - `SerializedMemoryEntry`
        - Used by `MEMORY_ONLY_SER`
        - Stores serialized bytes

### Storage Behavior

- Requires unrolling before storing deserialized partitions
- Unroll memory comes from storage pool
- Unroll memory converted into storage memory after successful materialization

### Retrieval

- Read updates LRU order
- Frequently accessed blocks move to tail
- Least recently used blocks stay near head

### Size Estimation

- Deserialized blocks use `SizeEstimator`
- Serialized blocks use exact byte count

### LRU Eviction

- Triggered by storage pressure or execution borrowing
- Least recently used blocks removed first
- Skips blocks that are pinned (currently being read) or belong to the protected storage fraction
- Eviction Behavior -
    - `MEMORY_ONLY`
        - Dropped
        - Recomputed later from lineage
    - `MEMORY_AND_DISK`
        - Written to disk before eviction

### ChunkedByteBuffer

- Stores serialized blocks
- Avoids huge contiguous arrays
- Default chunk size ≈ $64MB$
- Can be converted to Netty buffers for network transfers without copying

## DiskStore

- Persistent local storage layer
- Stores blocks as files
- Properties -
    - One file per block - file path determined by `DiskBlockManager`
    - Handles serialization and compression via `SerializerManager`
    - Used by cache eviction, shuffle, broadcasts

### Write Path

- Temporary file written first
- Atomic rename at completion from temp name to final name
- Prevents partial reads

### Read Path

- Memory-mapped file access - returns a `ChunkedByteBuffer`
- Uses OS page cache - repeated reads of the same file are fast

### File Naming

- File names are deterministic from `BlockId` -
    - `RDDBlockId(1, 5)` → `"rdd_1_5"`
    - `ShuffleBlockId(2, 3, 4)` → `"shuffle_2_3_4"`
    - `BroadcastBlockId(7)` → `"broadcast_7"`
- Deterministic naming means Spark can reconstruct file paths without metadata lookups

### Performance Notes

- Compression controlled by
    ```properties
    spark.shuffle.compress
    spark.rdd.compress
    ```

- No `fsync()`
    - Relies on OS write-back cache
    - Data may be lost on hard crash
    - But acceptable because Spark can recompute data via lineage recomputation

## Block Lifecycle

- __Cached RDD__ -
    - Task computes partition
    - Stored via `BlockManager`
    - Metadata reported to Driver

- __Shuffle Output__ -
    - `ShuffleWriter` writes partition data to local disk via `DiskBlockManager`
    - Metadata stored in `MapStatus` and sent to `DAGScheduler`

- __Broadcast Variable__ -
    - Stored on Driver
    - Executors fetch lazily via `TorrentBroadcast` (BitTorrent-style p2p)

### Access

- Local memory lookup
- Local disk lookup
- Remote fetch
- Recomputation if missing

### Eviction

- Triggered by memory pressure
- LRU based
- `BlockManagerMaster` notified of eviction

### Deletion

- Explicit `unpersist()`
- Shuffle cleanup
- Broadcast cleanup (`broadcast.unpersist()` / `broadcast.destroy()`)

> [!TIP]
> `unpersist()` only removes from executors; `destroy()` removes from Driver's `BlockManager` also

## Block Replication

- Enabled via replication storage levels - `MEMORY_ONLY_2` and `DISK_ONLY_2`
- Replication is synchronous - blocks until all replicas are confirmed stored before returning

### Replication Process

- Store locally
- Select replica executors
- Transfer block over Netty
- Replica reports success

> [!NOTE]
>  Replication maintained only during write; Lost replicas are not automatically rebuilt later

## DiskBlockManager

- Maps block IDs to physical files
- Used by `DiskStore`, shuffle writers and spill writers

### Directory Layout

- Uses `spark.local.dir`
- Directory structure -
    ```
    spark.local.dir/
    └── blockmgr-{uuid}/          ← one per BlockManager instance
    ├── 00/                       ← 64 subdirectories (00 - 3f hex)
    ├── 01/
    ├── ...
    └── 3f/
    ```

- Creates 64 hashed subdirectories - avoid filesystem degradation from huge directories
- Subdirectory for a block - `hash(blockId) % 64` → hex directory name

### Multiple Disks

- If `spark.local.dir` has multiple paths (different disks) -
    - `DiskBlockManager` creates one `blockmgr-{uuid}` directory per path
    - Block assignment distributed across directories via hash
    - Allows disk striping
    - Improves shuffle throughput

> [!NOTE]
> Put `spark.local.dir` on local SSDs; never use network storage for shuffle files

### Cleanup

- `DiskBlockManager` registers a JVM shutdown hook to delete all local directories on clean exit
- On executor failure, the directories are left on disk - cleaned up by the node manager (YARN/K8s) when it reclaims the container

## NettyBlockTransferService

- Spark's block transport layer
- One instance per `BlockManager`
- Used for executor ↔ executor and driver ↔ executor transfers

### Server Side

- `NettyBlockTransferService` starts a Netty server on a random port (or `spark.blockManager.port`)
- Server handles two types of requests -
    - __Block fetch__ - client requests blocks; server opens `ManagedBuffer`s for each block and streams them back
    - __Block upload__ - client pushes a block to this server (used for replication)
- `ManagedBuffer` abstraction - wraps different data sources uniformly -
    - `NioManagedBuffer` - used for cached blocks from `MemoryStore`
    - `FileSegmentManagedBuffer` - used for shuffle blocks served from `DiskStore`
    - `NettyManagedBuffer` - used for in-flight transfers

### Client Side

- `fetchBlocks` -
    - Connects to remote server
    - Sends request
    - Streams response blocks
    - Non-blocking - uses callbacks
- `uploadBlock` -
    - Sends serialized block data
    - Used for replication

### Zero-Copy Transfer

- `FileChannel.transferTo()` → OS `sendfile()` syscall
- Data goes directly from OS page cache to network socket without copying into JVM heap
- Benefits -
    - No JVM heap copy
    - Lower GC pressure
    - Faster shuffle reads

### External Shuffle Service

- Separate daemon outside executor JVM
- Benefits -
    - Shuffle survives executor death
    - Required for dynamic allocation

- Config -
    ```properties
    spark.shuffle.service.enabled=true
    ```

### Push-Based Shuffle

- Traditional - Reducers pull all map outputs
- Push-based - Mappers proactively push shuffle data to `ExternalShuffleService` which are merged into larger files
- Benefits -
    - Fewer files
    - Fewer connections
    - Faster reducer startup
- Config -
    ```properties
    spark.shuffle.push.enabled=true
    ```

## ShuffleClient

- Abstraction for shuffle block fetching
- Implementations -
    - `BlockTransferService` - direct executor fetch
    - `ExternalShuffleClient` - external shuffle service fetch
- Configs -
    ```properties
    spark.shuffle.io.maxRetries=3                   // max retry attempts per block
    spark.shuffle.io.retryWait=5s                   // wait between retries
    ```

### BlockFetcherIterator

- Core reducer-side fetch mechanism
- Controls -
    ```properties
    spark.reducer.maxSizeInFlight
    spark.reducer.maxBlocksInFlightPerAddress
    ```

## RpcEnv

- Spark's internal RPC framework for all Driver ↔ Executor and Executor ↔ Executor communication
- Replaced Akka
- Implemented using Netty
- One per Driver and Executor
- Core Abstractions -
    - `RpcEndpoint` - a named actor-like object that processes messages
    - `RpcEndpointRef` - a serializable reference to an `RpcEndpoint`; can be sent across the network
    - `RpcAddress` - `(host, port)` identifying an `RpcEnv`
- Communication Patterns -
    - `send()` - Fire-and-forget
    - `ask()` - Request-response
- Timeouts -
    ```properties
    spark.rpc.askTimeout=120s
    spark.rpc.lookupTimeout=120s
    spark.network.timeout=120s
    ```

### Thread Model

- Messages processed sequentially per endpoint
- Single-threaded endpoint execution
- Avoids race conditions without explicit synchronization

### Practical Failure Modes

- `RpcTimeoutException`
    - Driver overloaded
    - Long GC pause
    - Network issue

- `RpcEndpointNotFoundException`
    - Endpoint not registered
    - Startup race condition

- Lost executor
    - Missed heartbeat
    - Network timeout exceeded

> [!NOTE]
> `spark.network.timeout` is the master timeout
>   - Must be larger than worst-case GC pause
>   - If GC pause exceeds timeout -
>       - Driver marks executor dead
>       - Tasks are rescheduled
>       - Large performance impact
