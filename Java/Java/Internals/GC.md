# Garbage Collection

- Garbage Collection (GC) involves -
  - Periodically search the heap for unused objects (_Mark Phase_)
    - Start from GC roots and mark all unreachable objects.
  - Reclaiming memory of dead objects (_Sweep phase_)
    - Memory of unreachable objects is freed.
  - Reducing fragmentation (_Compaction / Relocation phase_)
    - Live objects may be moved so free memory becomes contiguous.

- __Stop-the-world__ -
  - GC often runs on multiple threads. 
  - Two logical groups of threads -
    - Mutator Threads - performing application logic
    - GC Threads - performing GC
  - When GC threads track object references or move objects around in memory - application threads must not be using those objects.
  - Therefore, JVM stops all application threads to safely perform garbage collection.

## Heap Generations

- Heap is divided into generations -
  - Young generation - Eden + Survivor spaces
  - Old (or tenured) generation

- __Minor GC / Young GC__ -
  - Objects are first allocated in the young generation.
  - When the young generation fills up -
    - GC stops all the application threads.
    - Empty the young generation -
      - Unused objects are discarded.
      - Live objects are either moved to survivor spaces or old generation.
  - Since all surviving objects are moved, the young generation is automatically compacted.
  - Objects that remain in the young generation are compacted within the other survivor space.

> [!NOTE]
> Common GC algorithms have stop-the-world pauses during collection of the young generation.

- __Full GC__ -
  - Old generation eventually fills up.
  - JVM stops all threads, scans the entire heap, removes unreachable objects, and compacts heap.
  - Causes long application pause.

> [!TIP]
> `System.gc()` triggers a full GC.

> [!TIP]
> To run off third-party code incorrectly calling `System.gc()` - `-XX:+DisableExplicitGC` - by default, it is set to `false`.

- __Concurrent Collectors__ -
  - Scan for unused objects without stopping application threads.
  - Minimize pause times, but increase CPU usage.

- __GC Tradeoffs__ -
  - REST / low-latency applications -
    - Response time is affected by GC pause times.
    - Full GC pauses cause worst latency spikes (outliers).
    - If minimizing latency spikes (p90/p99) is important - prefer concurrent GC (reduces pause duration).
    - If average latency matters more than tail latency - prefer a non-concurrent GC.
  - Batch applications -
    - Goal - finish job faster (throughput), not low latency.
    - If CPU is available - Concurrent GC helps avoid long Full GC pauses → faster completion.
    - If CPU is limited - Concurrent GC overhead slows down application → slower overall job.

## GC Roots

- Starting reference points for GC.
- JVM begins from these and marks everything reachable as alive.

- Major GC Roots -
  - __Thread Stacks__ -
    - Include local variables, method params, operand stack, registers holding live object references.
    - Stored in - Thread stacks, CPU registers.
    - JVM pauses threads at _safepoints_ and uses _OopMaps_ to precisely find object references in stack and registers for GC.
    - Safepoint = a state where all application threads are paused at known execution points, so their stack and registers are stable and predictable.
    - OopsMaps = metadata that mark which stack/register locations contain object references.
  - __Static field__ -
    - `static` object references of loaded classes.
    - Stored in - Class metadata (Metaspace)
    - JVM iterates all ClassLoaders, then all their loaded classes, and scans static fields to find GC roots.
  - __JNI Global References__ -
    - Native code holding Java objects.
    - Stored in - JNI global reference tables (native memory).
    - JVM scans these tables directly.

## GC Root Scanning Flow

- __Safepoint & Thread Stabilization__ -
  - JVM brings all threads to a safepoint - threads reach safepoint via cooperative polling (not force-paused).
  - Safepoint ensures OopMaps are valid and reference locations are reliable.
  - A dedicated `VMThread` executes VM operations that typically require a safepoint.
  - JVM waits until all threads reach safepoint-safe state.
  - Threads in native are prevented from re-entering JVM unsafely.

- __Stack Walking Across All Threads__ -
  - JVM iterates over all threads from its global thread list (`JavaThread`).
  - This phase is part of root enumeration / root processing, separate from heap traversal.
  - For each thread, it walks stack frame-by-frame (top to bottom).
  - Frame types -
    - _Interpreted frames_ have a fixed layout, so reference locations are directly known.
    - _Compiled frames_ have an optimized layout, so reference locations must be determined using metadata.
  - Stack walking uses internal structures like `frame` and `RegisterMap`.

- __Precise Reference Extraction (Stack + Registers)__ -
  - JVM does not scan raw memory or guess pointers.
  - It uses OopMaps (`OopMapSet`) generated by JIT at safepoints.
  - Internal paths like `frame::oops_do()` and `OopMapSet::all_do()` are used.
  - OopMaps provides -
    - exact stack slot positions with references.
    - exact registers holding references.
  - JVM extracts only live object references from stack and registers.

- __Scan JVM Runtime Structures__ -
  - JVM scans other known reference-holding structures outside stacks.
  - Includes -
    - class metadata (static fields)
    - JNI handle tables (local/global references)
    - Internal runtime tables (string intern table, reflection caches)
  - These are explicitly tracked by JVM, so they are directly iterated.
  - No dynamic memory scanning or pointer guessing is performed.

- __Build Initial Work Set__ -
  - All extracted references are passed into GC via - 
    - `OopsDoClosure` — a callback used by the JVM to process each object reference (oop) found during stack or heap scanning.
    - `RootClosure` — a specialized callback used to process object references discovered specifically during root scanning.
  - References are pushed into GC work queues.
  - This forms the initial frontier for graph traversal.
  - Root processing is typically parallelized across GC worker threads.

- __Transition to Traversal Phase__ -
  - GC starts processing the work queues.
  - For each reference -
    - object fields are scanned
    - connected objects are discovered
  - Recursively builds the full set of reachable objects.

> [!TIP]
> Root scanning is a latency-critical phase and often contributes significantly to GC pause time.
> 
> Its cost scales with -
>   - number of threads
>   - stack depth
>   - compiled frame complexity
> 
> It does not depend on heap size, but on the complexity of the JVM’s runtime state.

## Serial Garbage Collector

- Default collector for 32-bit JVM (client mode) or on a single-processor machine. 
- Uses a single thread.
- During a full GC - performs full compaction of the entire heap.
- Enable with - `-XX:+UseSerialGC`
- On systems where it is default, it can only be replaced by explicitly selecting another GC.

## Throughput Collector / Parallel Collector

- Default collector for 64-bit JVMs on multi-core machines.
- Uses multiple threads to collect the young and old generations.
- Stops all application threads during both minor and full GCs.
- Fully compacts the old generation during a full GC.
- Enable with - `-XX:+UseParallelGC`

## CMS (Concurrent Mark Sweep) Collector

- CMS was the first concurrent garbage collector designed to reduce pause times.
- Young GC - stop-the-world, typically using multiple threads (via ParNew in older JVMs).
- Old GC - collected concurrently, where the JVM -
  - Marks live objects in the background.
  - Sweeps unreachable objects while the application continues running.
- Limitation - 
  - Does not perform compaction during normal operation.
  - Therefore, the heap gradually becomes fragmented over time.
  - When fragmentation becomes severe, CMS must fall back to a Full GC (stop-the-world compaction), causing long pauses and negating its advantage.
- Deprecated in JDK 11+ and no longer recommended.
- Enable with - `-XX:+UseConcMarkSweepGC` - `false` by default.
- Historically, it required setting `-XX:+UseParNewGC` flag also -
  - otherwise the young generation would be collected by a single thread.
  - but now it is obsolete.

## G1/GC (Garbage First Garbage Collector) Collector

- Default collector in JDK 11+ (and modern JDKs) for 64-bit JVMs on multi-core machines.
- Heap is split into many equal-sized regions -
  - still uses Young + Old generation concept, but implemented using regions.
- Young GC - stop-the-world pause using multiple threads to copy live objects.
- Old GC - uses concurrent marking + background processing (not full cleanup every time).
- Follows a Garbage First strategy - prioritizes collecting regions with the _most garbage_.
- Avoids Full GC in most cases by doing incremental concurrent evacuation -
    - Live objects are copied between regions gradually.
- Performs incremental compaction during normal operation, reducing fragmentation (but not eliminating it completely)
- Trade-off -
  - Lower and more predictable pause times.
  - Higher CPU usage due to concurrent GC threads.
- Enable with - `-XX:+UseG1GC`

## Choosing a GC

| GC Algorithm         | Scenario / Condition                  | Why                                     |
| -------------------- | ------------------------------------- | --------------------------------------- |
| **G1 GC**            | Default choice (most apps)            | Balanced, predictable pause times       |
| **G1 GC**            | Low-latency / APIs / microservices    | Avoids long pauses, better tail latency |
| **G1 GC**            | Plenty of CPU available               | Background threads run efficiently      |
| **Serial GC**        | Single CPU / constrained environment  | No overhead of multiple threads         |
| **Serial GC**        | CPU-bound batch job (1 core)          | No CPU contention with GC threads       |
| **Parallel GC**      | CPU-bound batch job (multi-core)      | Maximizes throughput using all CPUs     |
| **Parallel GC**      | High throughput, latency not critical | Faster overall execution                |
| **Parallel GC**      | Few or no Full GCs                    | Avoids its main disadvantage            |
| **Parallel GC**      | CPU already heavily utilized          | No background GC thread contention      |
| **CMS (Deprecated)** | Legacy system using CMS               | Deprecated, migrate to G1               |

## GC Tuning

- __Sizing the Heap__ -
  - Heap sizing is a balance problem -
    - A small heap increases GC frequency and reduces application throughput
    - A large heap reduces GC frequency but increases pause duration
  - Larger heaps increase pause time -
    - GC must scan more memory
    - Full GC latency grows with heap size
  - Very large heaps can exceed physical memory -
    - OS starts using virtual memory (swapping to disk)
    - JVM is unaware of swapping since it is handled by the OS
  - Swapping must be strictly avoided -
    - Disk access is orders of magnitude slower than RAM
    - Full GC will touch most of the heap and trigger heavy swapping
    - GC pauses can increase by an order of magnitude
  - Therefore, never assign heap larger than physical memory -
    - If multiple JVMs run, consider the sum of all heap sizes
    - Leave memory for OS, JVM native memory, and other processes
    - Keep at least ~1 GB headroom.
  - Heap is controlled by JVM flags -
    - `-Xms` sets initial size
    - `-Xmx` sets maximum size
  - JVM performs ergonomic heap sizing -
    - Starts with initial heap
    - Expands when GC frequency is high
    - Stops growing at maximum heap size
    - Goal is to maintain acceptable GC overhead
  - If heap size is predictable
    - Set `-Xms` equal to `-Xmx`
    - Avoids heap resizing and improves GC efficiency
  - Practical sizing approach -
    - Run application to steady state
    - Trigger a full GC
    - Measure used heap after GC
    - Target ~30% utilization after full GC
