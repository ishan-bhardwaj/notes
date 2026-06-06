# Garbage Collection Algorithms

## Serial Garbage Collector

- Default collector for 32-bit JVM (client mode) or a single-processor machine.
- Enable with - `-XX:+UseSerialGC`
- On systems where it is default, it can only be disabled by explicitly selecting another GC. 
- Uses a single GC thread to process the heap.
- Stops all application threads during both minor and full GCs.

## Throughput Collector / Parallel Collector

- Default collector for 64-bit JVMs on multi-core machines.
- Enable with - `-XX:+UseParallelGC`
- Uses multiple threads to collect the young and old generations.
- Stops all application threads during both minor and full GCs.

## CMS (Concurrent Mark Sweep) Collector

- Deprecated in JDK 11+ (in favor of G1 GC).
- Enable with - `-XX:+UseConcMarkSweepGC`
- Historically, it required setting `-XX:+UseParNewGC` flag also -
  - otherwise the young generation would be collected by a single thread.
  - but now it is obsolete.
- CMS was the first concurrent garbage collector designed to reduce pause times.
- Young GC - stop-the-world, typically using multiple threads.
- Old GC - collected concurrently, where the JVM -
  - Marks live objects in the background.
  - Sweeps unreachable objects while the application continues running.
- Limitation - 
  - Does not perform compaction during normal operation.
  - Therefore, the heap gradually becomes fragmented over time.
  - When fragmentation becomes severe, CMS must fall back to a Full GC (stop-the-world compaction).

## G1 GC (Garbage First Garbage Collector) Collector

- G1 GC (Garbage First GC) splits the heap into many equal-sized regions instead of fixed contiguous spaces.
- Still follows the generational model -
  - Some regions act as Young regions.
  - Some act as Old regions.
  - Region roles can change dynamically.
- Young GC -
  - Stop-The-World (STW) pause.
  - Multiple GC threads evacuate/copy live objects from Young regions to Survivor or Old regions.
  - Empty regions are reclaimed completely.
- Old GC -
  - Uses concurrent marking and background processing.
  - Most marking work happens concurrently with application threads.
  - Avoids cleaning the entire old generation at once.
- G1 follows a Garbage-First strategy -
  - Prioritizes collecting regions containing the most garbage first.
  - Focuses on reclaiming maximum memory with minimum work.
- G1 avoids Full GC in most cases through incremental evacuation -
  - Live objects are gradually copied between regions over multiple GC cycles.
  - Cleanup work is spread across time instead of one massive pause.
- Compaction -
  - Performed incrementally during normal evacuation.
  - Fragmentation is reduced but not completely eliminated.
- Trade-offs -
  - Lower and more predictable pause times.
  - Better suited for large heaps and server applications.
  - Higher CPU and memory overhead due to concurrent/background GC threads.
