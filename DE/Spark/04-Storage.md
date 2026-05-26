# Storage & Block Management

## BlockManager Overview

- __`BlockManager`__ - the storage engine of every Spark process; manages all data blocks on a single node whether they are cached RDD partitions, shuffle data, broadcast variables, or spill files
- One `BlockManager` instance per JVM process - one on each executor, one on the Driver
- Lives inside `SparkEnv` - initialized during `SparkContext` creation before any tasks run
- Every piece of data in Spark that needs to be stored, retrieved, or transferred is a __block__ - identified by a `BlockId`, stored by `BlockManager`

### What BlockManager Manages

- __RDD blocks__ - `RDDBlockId(rddId, partitionIndex)` - cached RDD partitions
- __Shuffle blocks__ - `ShuffleBlockId(shuffleId, mapId, reduceId)` - shuffle map output chunks
- __Broadcast blocks__ - `BroadcastBlockId(broadcastId, field)` - broadcast variable data
- __Temp shuffle blocks__ - `TempShuffleBlockId(uuid)` - intermediate shuffle write buffers
- __Stream blocks__ - `StreamBlockId(streamId, uniqueId)` - Structured Streaming data
- __Spill blocks__ - managed indirectly via `DiskBlockManager`

### BlockManager Components

- __`MemoryStore`__ - stores blocks in JVM heap or off-heap memory
- __`DiskStore`__ - stores blocks on local disk
- __`DiskBlockManager`__ - manages the directory structure and file naming for disk blocks
- __`BlockManagerMaster`__ - proxy to the `BlockManagerMasterEndpoint` on the Driver; used to report block status and query block locations
- __`NettyBlockTransferService`__ - serves blocks to remote executors and fetches blocks from remote executors over the network
- __`MemoryManager`__ - `BlockManager` delegates memory allocation to `MemoryManager` (specifically `StorageMemoryPool`)
- __`SerializerManager`__ - handles serialization and compression for blocks being written to disk or transferred over network

### BlockManager Initialization

- Each executor's `BlockManager` registers with the Driver's `BlockManagerMaster` on startup -
    - Sends `RegisterBlockManager(blockManagerId, maxOnHeapMemory, maxOffHeapMemory, slaveEndpoint)`
    - `BlockManagerMaster` assigns a `BlockManagerId(executorId, host, port, topologyInfo)`
    - Driver maintains registry of all active `BlockManager`s and their memory capacities
- `BlockManagerId` is the cluster-wide identifier for a `BlockManager` instance -
    - Contains `executorId`, `host`, `port` (of the `NettyBlockTransferService`)
    - Used in `MapStatus` to tell reduce tasks where to fetch shuffle data

### Block Storage Decision

- `BlockManager.putIterator` / `putBytes` decides where to store a block based on `StorageLevel` -
    - If `StorageLevel.useMemory` → attempt `MemoryStore.putIteratorAsValues` or `putIteratorAsBytes`
    - If memory insufficient and `StorageLevel.useDisk` → fall back to `DiskStore.put`
    - If `StorageLevel.replication > 1` → replicate to remote `BlockManager` after local store
- Block lookup (`BlockManager.get`) -
    - Check `MemoryStore` first (fastest)
    - Check `DiskStore` if not in memory
    - If not local - query `BlockManagerMaster` for remote location, fetch via `NettyBlockTransferService`

### Block Reporting

- Every block put or removed triggers a report to `BlockManagerMaster` -
    - `reportBlockStatus(blockId, storageLevel, memSize, diskSize)`
    - `BlockManagerMaster` updates its global block registry
    - This is how the Driver knows which executor holds which cached partition

---

## BlockManagerMaster

- __`BlockManagerMaster`__ - the Driver-side coordinator for all block metadata across the cluster; the authoritative registry of what blocks exist, where they are, and how much memory each executor has
- Runs as an `RpcEndpoint` named `"BlockManagerMaster"` in the Driver's `RpcEnv`
- Every executor's `BlockManager` holds a `BlockManagerMaster` proxy (an `RpcEndpointRef`) and communicates with it for all metadata operations

### BlockManagerMasterEndpoint

- The actual RPC endpoint - `BlockManagerMasterEndpoint` - runs on the Driver
- Handles messages -
    - `RegisterBlockManager` - new executor registers; stored in `blockManagerInfo` map
    - `UpdateBlockInfo` - executor reports a block was added or removed; updates `blockLocations` map
    - `GetLocations(blockId)` - returns list of `BlockManagerId`s that hold a given block
    - `GetLocationsMultipleBlockIds` - batch version; used by DAGScheduler for locality computation
    - `RemoveBlock(blockId)` - tells all executors holding a block to remove it
    - `RemoveRdd(rddId)` - remove all blocks for an RDD (unpersist)
    - `RemoveShuffle(shuffleId)` - remove all shuffle blocks for a completed shuffle
    - `RemoveBroadcast(broadcastId)` - remove broadcast blocks
    - `HeartbeatReceived` - executor heartbeat with memory usage update
    - `BlockManagerHeartbeat` - keepalive; Driver uses to detect dead executors

### Global Block Registry

- `blockManagerInfo: Map[BlockManagerId, BlockManagerInfo]` -
    - Maps each executor's `BlockManagerId` to its info (max memory, used memory, block list)
    - Updated on registration and heartbeat
- `blockLocations: Map[BlockId, Set[BlockManagerId]]` -
    - Maps each `BlockId` to the set of executors that hold it
    - Updated on every `UpdateBlockInfo` message
    - Used by `DAGScheduler` to compute task preferred locations

### Topology-Aware Replication

- `BlockManagerMaster` uses topology information for replication placement -
    - Each `BlockManagerId` can carry `topologyInfo` (rack information)
    - When replicating a block, `BlockManagerMaster.getPeers` returns executors preferring different racks
    - Ensures replicas are on different failure domains

### Driver-Side BlockManager

- The Driver also has a `BlockManager` - used for -
    - Storing broadcast variables (Driver creates broadcast, stores locally, executors fetch)
    - Storing results of `collect()`, `take()` etc - Driver's `BlockManager` receives result blocks
    - The Driver's `BlockManagerMaster` proxy points to a local endpoint (no network hop)

---

## BlockManagerSlave

- __`BlockManagerSlave`__ (also called `BlockManagerSlaveEndpoint`) - the executor-side RPC endpoint that receives commands from `BlockManagerMaster`
- Each executor's `BlockManager` registers a `BlockManagerSlaveEndpoint` in its local `RpcEnv`
- Receives commands pushed from the Driver -
    - `RemoveBlock(blockId)` - remove a specific block from local storage
    - `RemoveRdd(rddId)` - remove all blocks for an RDD
    - `RemoveShuffle(shuffleId)` - remove shuffle blocks
    - `RemoveBroadcast(broadcastId, removeFromDriver)` - remove broadcast blocks
    - `GetBlockStatus(blockId)` - return current storage level, memory size, disk size of a block
    - `GetMatchingBlockIds(filter)` - return all block IDs matching a predicate
    - `ReplicateBlock(blockId, existingReplicas, maxReplicas)` - trigger replication of a block to additional executors

### Slave vs Master Interaction Pattern

- Master → Slave communication is command-driven (Driver tells executor what to do)
- Slave → Master communication is event-driven (executor reports what happened)
- This asymmetry means the Driver is always in control of block lifecycle; executors only act on local events spontaneously for things like LRU eviction (which they then report back)

---

## MemoryStore

- __`MemoryStore`__ - stores blocks in memory; the fastest storage tier in `BlockManager`
- Backed by a `LinkedHashMap[BlockId, MemoryEntry]` - linked for LRU ordering; most recently accessed blocks move to the end
- Two types of `MemoryEntry` -
    - `DeserializedMemoryEntry[T]` - stores deserialized JVM objects as `Array[T]`; used for `MEMORY_ONLY`
    - `SerializedMemoryEntry` - stores serialized bytes as `ChunkedByteBuffer`; used for `MEMORY_ONLY_SER`

### Storing a Block

- `putIteratorAsValues[T](blockId, iterator, classTag)` -
    - Iterates through the iterator, collecting elements into an `SizeEstimatingPartiallyUnrolledIterator`
    - Estimates size as it unrolls; requests more unroll memory from `MemoryManager` as needed
    - If unroll completes successfully - converts unroll memory to storage memory; stores `DeserializedMemoryEntry`
    - If unroll fails (OOM) - returns a `PartiallyUnrolledIterator` so the caller can fall back to disk
- `putIteratorAsBytes(blockId, iterator, storageLevel, classTag)` -
    - Serializes the iterator directly into a `ChunkedByteBuffer`
    - Uses `SerializerManager` for serialization and optional compression
    - Stores `SerializedMemoryEntry`
- `putBytes(blockId, bytes, storageLevel)` -
    - Directly stores pre-serialized bytes
    - Used when the data is already in wire format (eg - receiving a replicated block from remote)

### Retrieving a Block

- `getValues(blockId)` - returns `Option[Iterator[T]]`; for deserialized blocks, wraps the array in an iterator
- `getBytes(blockId)` - returns `Option[ChunkedByteBuffer]`; for deserialized blocks, serializes on the fly
- Access updates LRU order - block moved to end of `LinkedHashMap`

### Size Estimation

- `SizeEstimator.estimate(obj)` - estimates JVM object graph size using reflection and sampling
- Critical for deciding when to evict - inaccurate estimates cause either premature eviction or OOM
- Estimates are approximate - actual memory usage may differ due to JVM internals
- Serialized blocks have exact sizes (byte count); deserialized blocks have estimated sizes

### LRU Eviction

- `evictBlocksToFreeSpace(blockId, space, memoryMode)` -
    - Called by `MemoryManager` when execution needs to borrow storage memory
    - Iterates `LinkedHashMap` from head (least recently used)
    - Skips blocks that are pinned (currently being read) or belong to the protected storage fraction
    - For each eviction candidate -
        - If `MEMORY_AND_DISK` - writes to `DiskStore` first, then removes from `MemoryStore`
        - If `MEMORY_ONLY` - removes directly (block lost; recomputed from lineage on next access)
    - Calls `BlockManagerMaster.updateBlockInfo` to report eviction
    - Stops when enough space is freed

### ChunkedByteBuffer

- Serialized blocks are stored as `ChunkedByteBuffer` - a sequence of `ByteBuffer`s rather than one large array
- Avoids large contiguous allocation requirements; works with JVM's max array size limit ($2^{31} - 1$ bytes)
- Each chunk is a fixed size (default $64 MB$)
- `toNetty()` - converts to Netty `ByteBuf` for network transfer without copying

---

## DiskStore

- __`DiskStore`__ - stores blocks as files on local disk; the slower but durable storage tier
- Each block is stored as a separate file; file path determined by `DiskBlockManager`
- Handles serialization/deserialization and compression via `SerializerManager`

### Storing a Block

- `put(blockId, serializationStream => ...)` -
    - Gets file path from `DiskBlockManager.getFile(blockId)`
    - Opens `FileOutputStream` → `BufferedOutputStream` → optionally compressed stream → serialization stream
    - Caller writes data to the serialization stream
    - File atomically renamed from temp name to final name after write completes (prevents partial reads)
- `putBytes(blockId, bytes)` -
    - Writes pre-serialized `ChunkedByteBuffer` directly to file
    - No serialization overhead - used for shuffle data which is already serialized

### Retrieving a Block

- `getBytes(blockId)` - memory-maps the file and returns a `ChunkedByteBuffer`
    - Memory mapping allows reading without loading entire file into heap
    - OS page cache handles caching; repeated reads of the same file are fast
- `get(blockId, classTag)` - deserializes the file contents and returns an iterator

### File Naming

- File names are deterministic from `BlockId` -
    - `RDDBlockId(1, 5)` → `"rdd_1_5"`
    - `ShuffleBlockId(2, 3, 4)` → `"shuffle_2_3_4"`
    - `BroadcastBlockId(7)` → `"broadcast_7"`
- Deterministic naming means Spark can reconstruct file paths without metadata lookups

### Disk Write Performance

- Write buffer size - `DiskStore` uses a `BufferedOutputStream` with default $32 KB$ buffer
- Compression at write time - if `spark.shuffle.compress` or `spark.rdd.compress` is `true`, data is compressed before writing; reduces disk I/O at cost of CPU
- Sync behavior - `DiskStore` does NOT call `fsync` after writes; relies on OS write-back; data may be lost on hard crash but Spark tolerates this via lineage recomputation

---

## Block Lifecycle

- A block goes through distinct lifecycle phases from creation to deletion

### Creation

- __RDD partition cached__ -
    - Action triggers task execution
    - Task's `compute()` produces an iterator
    - `BlockManager.putIteratorAsValues` called with `RDDBlockId` and the iterator
    - Block stored to `MemoryStore` (and/or `DiskStore` based on `StorageLevel`)
    - `BlockManagerMaster` notified via `UpdateBlockInfo`
- __Shuffle block written__ -
    - Map task runs `ShuffleWriter.write(iterator)`
    - `ShuffleWriter` writes partition data to local disk via `DiskBlockManager`
    - `MapStatus` created with output sizes per partition
    - `MapStatus` sent to `DAGScheduler` which forwards to `MapOutputTrackerMaster`
- __Broadcast variable created__ -
    - Driver creates broadcast via `SparkContext.broadcast(value)`
    - Value serialized and stored in Driver's `BlockManager` as `BroadcastBlockId`
    - Executors fetch on first access via `TorrentBroadcast` (BitTorrent-style p2p)

### Access

- `BlockManager.getOrElseUpdate(blockId, storageLevel, iterator)` -
    - Check local `MemoryStore` → `DiskStore`
    - If not local, check `BlockManagerMaster` for remote locations
    - If remote - fetch via `NettyBlockTransferService`
    - If not found anywhere (first access or evicted) - compute from `iterator` and store
- Remote fetch triggers `BlockTransferService.fetchBlockSync` - synchronous block fetch from remote executor

### Eviction

- Triggered by memory pressure - `MemoryStore.evictBlocksToFreeSpace`
- LRU policy - least recently accessed block evicted first
- `MEMORY_AND_DISK` blocks - written to `DiskStore` before eviction from `MemoryStore`
- `MEMORY_ONLY` blocks - dropped entirely; recomputed from lineage
- `BlockManagerMaster` notified of eviction

### Deletion

- Explicit `unpersist()` -
    - `SparkContext.unpersistRDD(rddId)` calls `BlockManagerMaster.removeRdd(rddId)`
    - `BlockManagerMaster` sends `RemoveRdd` to all executors holding blocks for that RDD
    - Each executor's `BlockManagerSlave` removes the blocks from `MemoryStore` and `DiskStore`
- Shuffle cleanup -
    - After a shuffle stage's downstream stages complete, `DAGScheduler` calls `mapOutputTracker.unregisterShuffle`
    - Shuffle files cleaned up by `BlockManager.removeShuffle`
- Broadcast cleanup -
    - `broadcast.unpersist()` or `broadcast.destroy()`
    - `destroy()` also removes from Driver's `BlockManager`; `unpersist()` only removes from executors

---

## Block Replication

- __Block replication__ - storing copies of a block on multiple executors for fault tolerance; activated when `StorageLevel.replication > 1` (eg - `MEMORY_ONLY_2`, `DISK_ONLY_2`)
- Replication is synchronous - `BlockManager.putIterator` blocks until all replicas are confirmed stored before returning

### Replication Process

- After storing a block locally, `BlockManager.replicate(blockId, bytes, storageLevel, classTag)` -
    1. Asks `BlockManagerMaster.getPeers(blockManagerId)` for candidate replica targets
    2. `BlockManagerMaster` returns other `BlockManagerId`s, preferring different racks if topology info available
    3. Serializes the block to bytes if not already serialized
    4. Sends bytes to target executor's `NettyBlockTransferService` via `BlockTransferService.uploadBlock`
    5. Target executor stores the block locally and reports to `BlockManagerMaster`
    6. Repeats until `replication - 1` successful remote copies made

### Replica Selection

- `BlockManagerMaster.getPeers` shuffles available executors (excluding self) and returns them
- Topology-aware selection - if rack information available, prefers executors on different racks
- Replication retries on failure - if target executor is unreachable, tries next peer

### Replication on Eviction

- When a replicated block is evicted from one executor, the remaining replicas are still valid
- `BlockManagerMaster` updates `blockLocations` to remove the evicted location
- Other replicas remain accessible - no replication triggered automatically to restore replication factor
- Replication factor is only maintained at write time; Spark does not background-replicate to restore lost replicas

### Cost of Replication

- Doubles (or triples) network bandwidth usage at cache time
- Doubles memory usage across the cluster
- Adds latency to `persist()` operations
- Use only for RDDs where recomputation is prohibitively expensive and executor failures are expected

---

## DiskBlockManager

- __`DiskBlockManager`__ - manages the directory structure for all disk-based blocks on a single executor; translates `BlockId`s to file system paths
- Sits below `DiskStore` and `ExternalSorter` - any component writing blocks to disk goes through `DiskBlockManager`

### Directory Structure

- Creates local directories under each path in `spark.local.dir` (comma-separated list of directories)
- Directory structure -
    ```
    spark.local.dir/
    └── blockmgr-{uuid}/          ← one per BlockManager instance
    ├── 00/                       ← 64 subdirectories (00 - 3f hex)
    ├── 01/
    ├── ...
    └── 3f/
    ```

- $64$ subdirectories prevent a single directory from accumulating too many files (filesystem performance degrades with thousands of files in one directory)
- Subdirectory for a block - `hash(blockId) % 64` → hex directory name

### File Path Resolution

- `getFile(blockId: BlockId)` - returns `File` object for a block
    - Computes `hash(blockId.name) % dirCount` to select the `blockmgr-{uuid}` directory
    - Computes `hash(blockId.name) % subDirsPerLocalDir` to select the subdirectory
    - Returns `File(localDir, subDir, blockId.name)`
- `getFile(filename: String)` - same but takes raw filename; used for shuffle files
- `createTempLocalBlock()` - creates a `TempLocalBlockId` and its corresponding file; used for intermediate shuffle data

### Multiple Local Directories

- If `spark.local.dir` has multiple paths (different disks) -
    - `DiskBlockManager` creates one `blockmgr-{uuid}` directory per path
    - Block assignment distributed across directories via hash
    - Stripe across multiple disks for higher aggregate I/O throughput
    - Best practice - one path per physical disk; SSDs preferred for shuffle data

### Cleanup

- `DiskBlockManager` registers a JVM shutdown hook to delete all local directories on clean exit
- On executor failure, the directories are left on disk - cleaned up by the node manager (YARN/K8s) when it reclaims the container
- `spark.local.dir` should be on ephemeral storage (not HDFS-mounted); these are truly temporary files

---

## NettyBlockTransferService

- __`NettyBlockTransferService`__ - the network layer of `BlockManager`; handles all block transfers between executors and between Driver and executors using Netty
- One instance per `BlockManager` - one server + one client per executor/Driver
- Replaces the old `NioBlockTransferService` (removed in Spark 2.0)

### Server Side

- `NettyBlockTransferService` starts a Netty server on a random port (or `spark.blockManager.port`)
- Server handles two types of requests -
    - __Block fetch__ - `OpenBlocks(appId, execId, blockIds)` - client requests blocks; server opens `ManagedBuffer`s for each block and streams them back
    - __Block upload__ - `UploadBlock(appId, execId, taskAttemptId, blockId, metadata, data)` - client pushes a block to this server (used for replication)
- `ManagedBuffer` abstraction - wraps different data sources uniformly -
    - `FileSegmentManagedBuffer` - a segment of a file on disk (used for shuffle blocks served from `DiskStore`)
    - `NioManagedBuffer` - a `ByteBuffer` in memory (used for cached blocks from `MemoryStore`)
    - `NettyManagedBuffer` - a Netty `ByteBuf` (used for in-flight transfers)

### Client Side

- `fetchBlocks(host, port, execId, blockIds, listener, tempFileManager)` -
    - Connects to remote server
    - Sends `OpenBlocks` request
    - Streams response blocks; calls `listener.onBlockFetchSuccess` or `listener.onBlockFetchFailure` per block
    - Non-blocking - uses callbacks; caller can issue many fetches concurrently
- `uploadBlock(host, port, execId, blockId, buf, storageLevel, classTag)` -
    - Sends `UploadBlock` message with serialized block data
    - Used for replication

### Zero-Copy Transfer

- When serving a shuffle block from disk, Netty uses `FileRegion` for zero-copy transfer -
    - `FileChannel.transferTo()` → OS `sendfile()` syscall
    - Data goes directly from OS page cache to network socket without copying into JVM heap
    - Critical for shuffle performance - avoids GC pressure from loading shuffle file bytes into JVM heap

### External Shuffle Service

- __`ExternalShuffleService`__ - a separate daemon process running on each worker node outside executor JVMs
- When enabled (`spark.shuffle.service.enabled=true`) -
    - Executors register their shuffle file locations with the local `ExternalShuffleService` on startup
    - Reduce tasks fetch shuffle data from `ExternalShuffleService` instead of directly from executor's `NettyBlockTransferService`
    - Executor can die or be killed (dynamic allocation); shuffle files remain accessible via `ExternalShuffleService`
- `ExternalShuffleService` runs as a separate JVM; same Netty server protocol as `NettyBlockTransferService`
- Required for safe dynamic allocation - without it, killing idle executors destroys their shuffle output

### Push-Based Shuffle (Spark 3.2+)

- Traditional shuffle - reduce tasks pull data from map outputs after map stage completes
- Push-based shuffle - map outputs are pushed to `ExternalShuffleService` during map stage; merged into larger files before reduce stage starts
- Benefits -
    - Reduce-side fetch reads larger merged files instead of many small individual map output chunks
    - Better I/O patterns, fewer connections, lower reducer startup latency
- Enable via `spark.shuffle.push.enabled=true`
- Requires `ExternalShuffleService` and Spark 3.2+

---

## ShuffleClient

- __`ShuffleClient`__ - abstract interface for fetching shuffle blocks; decouples shuffle fetch logic from the transport implementation
- Two implementations -
    - `BlockTransferService` (extends `ShuffleClient`) - used when no External Shuffle Service; fetches directly from executor's `NettyBlockTransferService`
    - `ExternalShuffleClient` - used when External Shuffle Service is enabled; fetches from the node daemon

### ShuffleClient Interface

- `fetchBlocks(host, port, execId, blockIds, listener, downloadFileManager)` -
    - Fetches a set of blocks from a remote host
    - `BlockFetchingListener.onBlockFetchSuccess(blockId, data)` - called per successful block
    - `BlockFetchingListener.onBlockFetchFailure(blockId, exception)` - called per failed block
- `init(appId)` - initialize with application ID; `ExternalShuffleClient` uses this to register with the daemon
- `close()` - release resources

### RetryingBlockFetcher

- Wraps `ShuffleClient` with retry logic -
    - `spark.shuffle.io.maxRetries` (default $3$) - max retry attempts per block
    - `spark.shuffle.io.retryWait` (default $5s$) - wait between retries
    - Retries only on `IOException` (transient network errors); not on `BlockNotFoundException`
    - Exponential backoff not implemented - fixed wait between retries

### BlockFetcherIterator

- Used by `ShuffleBlockFetcherIterator` inside reduce tasks -
    - Manages a queue of blocks to fetch
    - Fetches blocks in batches limited by `spark.reducer.maxSizeInFlight` ($48 MB$) and `spark.reducer.maxBlocksInFlightPerAddress`
    - Local blocks (same executor) fetched directly from `BlockManager` without network
    - Remote blocks fetched via `ShuffleClient`
    - Returns an iterator that yields `(BlockId, InputStream)` pairs; reduce task consumes this iterator

---

## RpcEnv

- __`RpcEnv`__ - the messaging infrastructure for all Driver ↔ Executor and Executor ↔ Executor communication in Spark; replaces the Akka actor system removed in Spark 2.0
- Netty-based implementation - `NettyRpcEnv`; one per JVM process (one on Driver, one on each Executor)
- Every internal Spark component that needs to communicate across JVM boundaries registers as an `RpcEndpoint` in its local `RpcEnv`

### Core Abstractions

- __`RpcEndpoint`__ - a named actor-like object that processes messages
    - `receive` - handles fire-and-forget messages (`send`)
    - `receiveAndReply` - handles request-response messages (`ask`)
    - `onStart` / `onStop` - lifecycle hooks
    - `onConnected` / `onDisconnected` - connection event hooks
    - `onError` - error handler
- __`RpcEndpointRef`__ - a serializable reference to an `RpcEndpoint`; can be sent across the network
    - `send(message)` - fire-and-forget; no delivery guarantee
    - `ask[T](message, timeout)` - returns `Future[T]`; request-response
    - `askSync[T](message, timeout)` - blocking version of `ask`
- __`RpcAddress`__ - `(host, port)` identifying an `RpcEnv`
- __`RpcCallContext`__ - provided to `receiveAndReply`; call `reply(response)` or `sendFailure(exception)`

### Key RpcEndpoints in Spark

- __Driver-side__ -
    - `BlockManagerMasterEndpoint` - block registry; receives `UpdateBlockInfo`, answers `GetLocations`
    - `MapOutputTrackerMasterEndpoint` - shuffle map output registry; answers `GetMapOutputStatuses`
    - `DAGSchedulerEventProcessLoop` - receives task completion events, executor lost events
    - `DriverEndpoint` (CoarseGrainedSchedulerBackend) - receives executor registration, task status updates
    - `HeartbeatReceiver` - receives executor heartbeats; detects dead executors
- __Executor-side__ -
    - `BlockManagerSlaveEndpoint` - receives block removal commands from Driver
    - `CoarseGrainedExecutorBackend` - receives task launch commands from Driver; sends task status updates

### Message Passing Internals

- Messages serialized with Java serialization (not Kryo - must be Java-serializable)
- Transport layer - Netty TCP connections between `RpcEnv` instances
- Connection pooling - `TransportClientFactory` maintains a pool of Netty connections to each remote address
- Thread model -
    - `RpcEndpoint.receive` called on a thread pool (`dispatcher-event-loop`)
    - Endpoints are single-threaded by default - messages processed sequentially per endpoint
    - Prevents race conditions in endpoint state without explicit synchronization

### RpcEnv Lifecycle

- `RpcEnv.create(name, host, port, conf, securityManager)` - creates and starts the Netty server
- `rpcEnv.setupEndpoint(name, endpoint)` - registers an endpoint; returns `RpcEndpointRef`
- `rpcEnv.setupEndpointRef(address, endpointName)` - gets a ref to a remote endpoint by address + name
- `rpcEnv.awaitTermination()` - blocks until the RpcEnv is stopped (used by Driver main thread)
- `rpcEnv.stop(endpointRef)` - stops and unregisters an endpoint
- `rpcEnv.shutdown()` - shuts down the entire RpcEnv and closes all connections

### Ask Timeout and Failure Handling

- `spark.rpc.askTimeout` (default $120s$) - timeout for `ask` operations
- `spark.rpc.lookupTimeout` (default $120s$) - timeout for `setupEndpointRef` (remote endpoint lookup)
- On timeout - `ask` Future fails with `RpcTimeoutException`
- On remote endpoint failure - `ask` Future fails with the exception thrown by `receiveAndReply`
- `RpcEndpointNotFoundException` - thrown when endpoint name not found at the given address

### Dispatcher and Thread Model

- `Dispatcher` - internal component that routes messages to endpoints
    - Maintains a `MessageLoop` thread pool
    - Each `RpcEndpoint` has an `EndpointData` with an inbox (message queue)
    - `Dispatcher.postMessage` enqueues to inbox; `MessageLoop` drains inboxes
    - Single-threaded per endpoint - no concurrent message delivery to the same endpoint
- Outbox - each `RpcEnv` maintains an `Outbox` per remote address
    - Messages to remote endpoints enqueued to `Outbox`
    - `Outbox` drainer serializes and sends messages over Netty when connection is available
    - If connection is unavailable - messages buffered; connection established lazily

> [!NOTE]
> `RpcEnv` is not a public API - it is internal to Spark. However, understanding it is essential for debugging distributed coordination failures. Common failure modes -
>   - `RpcTimeoutException` on `ask` - remote endpoint overloaded or dead; check GC pauses on Driver
>   - `RpcEndpointNotFoundException` - endpoint stopped or not yet started; race condition in startup
>   - Lost executor detected via missed heartbeat in `HeartbeatReceiver` - configurable via `spark.executor.heartbeatInterval` (default $10s$) and `spark.network.timeout` (default $120s$)

> [!TIP]
> `spark.network.timeout` is the master timeout that overrides `spark.rpc.askTimeout`, `spark.storage.blockManagerSlaveTimeoutMs`, and others if they are not set explicitly. Always set `spark.network.timeout` to a value larger than your longest GC pause. A GC pause longer than `spark.network.timeout` causes the Driver to declare the executor dead and reschedule all its tasks - extremely disruptive.
