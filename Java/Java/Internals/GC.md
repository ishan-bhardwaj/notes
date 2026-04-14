# Garbage Collection

- Garbage Collection (GC) involves -
  - Periodically search the heap for unused objects (_Mark Phase_)
    - Start from GC roots and mark all unreachable objects.
  - Reclaiming memory of dead objects (_Sweep phase_)
    - Memory of unreachable objects is freed.
  - Reducing fragmentation (_Compaction / Relocation phase_)
    - Live objects may be moved so free memory becomes contiguous.

- __GC roots__ -
  - Objects accessible from outside the heap.
  - Example - static fields, thread stacks, system classes.
  - These objects are _always_ reachable.
  - GC algorithm scans all objects that are reachable via one of the root objects.

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