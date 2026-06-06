# Garbage Collection

- Garbage Collection (GC) involves -
  - Periodically searching the heap for unused objects (_Mark Phase_)
    - Start from GC roots and mark all unreachable objects.
  - Reclaiming memory of dead objects (_Sweep phase_)
    - Memory of unreachable objects is freed.
  - Reducing fragmentation (_Compaction / Relocation phase_)
    - Live objects may be moved so free memory becomes contiguous.

## Stop-the-world
  
- GC often runs on multiple threads. 
- There are two logical groups of threads -
  - Mutator Threads - performing application logic
  - GC Threads - performing GC
- When GC threads track object references or move objects around in memory - application threads must not be using those objects.
- Therefore, JVM stops all application threads to safely perform garbage collection.

## Heap Generations

- Heap is divided into generations -
  - Young generation - Eden + Survivor spaces (usually two - S0, S1)
  - Old generation

## Minor GC / Young GC
  
- Objects are first allocated in the young generation.
- When the young generation fills up -
  - GC stops all the application threads.
  - Empty the young generation -
    - Unused objects are discarded.
    - Live objects are either moved to survivor spaces or old generation.
- Young Generation collectors use a copying collection algorithm i.e. instead of compacting objects in-place, the JVM -
  - Reads live objects from one area
  - Copies them into another empty area
  - Frees the entire old area at once
- Survivor spaces keep swapping roles during Minor GC -
  - After one GC, surviving objects may move from Eden → S0
  - During next GC, surviving objects from Eden + S0 are copied into S1
  - Next GC copies survivors from Eden + S1 back into S0
  - This continues as S0 ↔ S1
- Since all surviving objects are copied during Minor GC -
  - The young generation is automatically compacted.
  - Fragmentation is avoided.
  - Allocation remains very fast.
- Objects that survive multiple Minor GCs are eventually promoted to the old generation.

> [!NOTE]
> Common GC algorithms have stop-the-world pauses during collection of the young generation.

## Full GC

- Old generation eventually fills up.
- JVM stops all threads, scans the entire heap (both young and old generations), removes unreachable objects, and compacts heap.
- Causes long application pause.

> [!TIP]
> `System.gc()` triggers a full GC.

> [!TIP]
> To run off third-party code incorrectly calling `System.gc()` - `-XX:+DisableExplicitGC` - by default, it is set to `false`.

## Concurrent / Low-Pause Collectors

- Scan for unused objects while application threads are running.
- Minimize pause times, but increase CPU usage.
- Eg - G1GC

## Basic GC Tuning

- __Sizing the Heap__ -
  - `-Xms` sets initial size
  - `-Xmx` sets maximum size
  - JVM performs ergonomic heap sizing -
    - Starts with initial heap
    - Expands when GC frequency is high
    - Stops growing at maximum heap size
  - If heap size is predictable -
    - Set `-Xms` equal to `-Xmx`
    - Avoids heap resizing and improves GC efficiency
  - Common rule of thumb for choosing max heap size -
    - After a Full GC, heap usage should be around ~30% of max heap.
  - Heap sizing approach -
    - Run the application until it reaches steady state.
    - Connect to application using `jconsole` and trigger a Full GC.
    - Observe memory usage after GC completes.
    - Size the heap accordingly.

> [!WARNING]
> Avoid allocating heap larger than physical memory -
>   - OS may start swapping heap pages to disk.
>   - Swapping makes GC extremely slow since disk is far slower than RAM.
>   - Full GC can trigger massive swap activity and cause huge pause times.
>   - If multiple JVMs run on the same machine, consider the combined heap sizes.
>   - Always leave memory for JVM native memory, the OS, and other applications (typically at least ~1 GB headroom).

- __Sizing the Heap__ -
  - Sizing young generation - 
    - `-XX:NewRatio=N` - set the ratio of the young generation to the old generation.
    - `-XX:NewSize=N` - set the initial size of the young generation.
    - `-XX:MaxNewSize=N` - set the maximum size of the young generation.
    - `-XmnN` - shorthand for setting both `NewSize` and `MaxNewSize` to the same value.
  