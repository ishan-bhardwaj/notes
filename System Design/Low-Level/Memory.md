## Memory

- __Main memory (physical memory)__ -
    - Fast data storage, typically DRAM
- __Virtual memory__ -
    - Abstraction over main memory
    - Appears infinite and non-contended
    - Not real memory
- __Resident memory__ -
    - Memory currently residing in main memory
- __Anonymous memory__ -
    - Memory with no filesystem location
    - Includes working data of a process address space, called heap
- __Address space__ -
    - Memory context
    - Each process and the kernel has its own virtual address space
- __Segment__ -
    - Region of virtual memory with a designated purpose
    - Eg - executable or writable pages
- __Instruction text__ -
    - CPU instructions in memory, typically in a dedicated segment
- __OOM__ -
    - Out of memory, when the kernel detects low available memory
- __Page__ -
    - OS/CPU unit of memory
    - Traditionally $4KB$ or $8KB$
    - Modern processors have _multiple page size support_ for larger sizes
- __Page fault__ -
    - Invalid memory access
    - Usually happens when using on-demand virtual memory
- __Paging__ -
    - Transfer of pages between main memory and storage
- __Swapping (Linux)__ -
    - Also called _anonymous paging_ in other operating systems
    - Transfer of _swap pages_ between main memory and the swap devices
- __Swap__ -
    - On-disk backing area for anonymous pages
    - Either an area on a storage device (called _physical swap device_), or a file (called _swap file_)

## Virtual Memory

- Abstraction providing each process and the kernel with a large, linear, private address space
- Delegates physical memory placement to the OS
- Supports multitasking - virtual address spaces are isolated by design
- Supports oversubscription - in-use memory can exceed physical main memory
- __Address space mapping__ -
    - Process address space mapped by virtual memory subsystem to main memory and swap device
    - Kernel moves pages between them as needed - swapping
    - This enables kernel to oversubscribe main memory
- __Oversubscription limits__ -
    - Kernel may cap oversubscription, commonly at main memory + swap device size
    - Allocations exceeding this limit fail
    - Results in "out of virtual memory" errors

## Paging

- Fine-grained movement of pages in and out of main memory - called _page-ins_ and _page-outs_
- Enables partially loaded or oversized programs to execute
- Two types of paging -
    - __File System Paging__ -
        - Caused by reading/writing pages in memory-mapped files
        - Referred as _good paging_
        - __Dirty page__ - modified in memory, must be written to disk on page-out
        - __Clean page__ - unmodified, page-out just frees memory (copy already on disk)
    - __Anonymous Paging (Swapping)__ -
        - Involves data private to processes - heap and stacks
        - No filesystem path, hence _anonymous_
        - Referred as _bad paging_ because it hurts performance
        - Page-out - asynchronous, may not directly impact application
        - Page-in - synchronous, blocks application on disk I/O
        - Best case - no anonymous paging at all
        - Achieved by sizing applications within available main memory and monitoring page scanning, memory utilization, and anonymous paging

## Demand Paging

- Pages of virtual memory mapped to physical memory on demand, not at allocation time
- Defers CPU overhead of creating mappings until actually accessed
- __Page fault sequence__ -
    - `malloc()` allocates virtual memory
    - Store/load instruction triggers MMU virtual-to-physical lookup
    - Lookup fails as no mapping exists yet - triggers page fault
    - Kernel creates on-demand mapping
    - Page may later be swapped out to free memory
- __Minor fault__ -
    - Mapping satisfied from another page in memory
    - Occurs when -
        - Mapping a new page from available memory, during memory growth of the process
        - Mapping to another existing page, such as reading a page from a mapped shared library
- __Major fault__ -
    - Requires storage device access
    - Eg - accessing an uncached memory-mapped file
- __Virtual memory page states__ -
    - Unallocated - not yet allocated
    - Allocated, unmapped - allocated but not yet faulted (unpopulated)
    - Allocated, mapped to RAM - in main memory
    - Allocated, mapped to swap - paged out to disk
- __Memory sizing terms__ -
    - Resident Set Size (RSS) - size of allocated pages currently in RAM
    - Virtual memory size - size of all allocated areas (unmapped + RAM + swap)

## Overcommit

- Allows more memory to be allocated than physical memory + swap combined
- Relies on demand paging and applications tendency to not use much of the memory they have been allocated
- `malloc()` succeeds even when conservative limits would have failed
- Enables generous allocation with sparse on-demand use
- Consequences depend on how kernel manages memory pressure

## Process Swapping

- Movement of entire processes between main memory and swap device
- Original Unix technique for managing main memory
- Swap-out includes -
    - Process heap (anonymous data)
    - Open file table and process metadata
    - Unmodified filesystem data is dropped, re-read from disk when needed
- Severely hurts performance - process requires numerous disk I/Os to resume
- Linux does not use process swapping - relies solely on paging

## File System Cache Usage

- OS uses spare memory to cache the filesystem after boot - improves performance
- Available free memory shrinks to near zero post-boot - expected, not a problem
- Kernel reclaims filesystem cache quickly when applications need memory

## Utilization and Saturation

- __Utilization__ -
    - Used memory vs total memory
    - Filesystem cache counts as unused - reclaimable by applications
- __Saturation__ -
    - Occurs when memory demand exceeds physical main memory
    - Kernel responds with paging, process swapping (if supported), or OOM killer
    - Any of these is an indicator of saturation
- Virtual memory saturation -
    - If kernel imposes a virtual memory limit, exhaustion causes `malloc()` to fail with `ENOMEM`
    - Linux overcommit does not impose such a limit

## Allocators

- Handle allocation and placement within a virtual address space
- Either user-land libraries or kernel routines (e.g. `malloc()`, `free()`)
- Can significantly impact performance
- A system may provide multiple user-level allocator libraries to pick from
- Performance improvements - uses techniques such as per-thread object caching
- Performance degradation - fragmented or wasteful allocation

## Shared Memory

- Memory shared between processes
- Common use - system libraries share read-only instruction text across all processors that use it
- Observability challenge -
    - Per-process memory reporting is ambiguous - should shared memory be included?
- __Proportional Set Size (PSS)__ -
    - Linux metric to address this
    - Private memory + (shared memory / number of sharers)

## Working Set Size (WSS)

- Amount of main memory a process frequently uses to perform work
- Performance implications -
    - WSS fits in CPU cache - significant performance improvement
    - WSS exceeds main memory - severe degradation, application must swap
- Not directly observable - tools report RSS, not WSS
- WSS estimation requires experimental methodology

## Word Size

- Address space size bounded by word size - $32$-bit caps addressable range at $4GB$
- Applications requiring $> 4GB$ must be compiled for $64$-bit
- Kernel address reservation -
    - Portion of address space reserved for kernel, unavailable to applications
    - 32-bit Linux - 1GB kernel reservation, 3GB for application
    - 32-bit Windows - 2GB kernel reservation by default, 3GB with `/3GB` option
    - 64-bit - address space large enough that reservation is not a concern
- Larger word sizes can improve memory performance - instructions operate on larger chunks
- Minor memory waste possible when data types have unused bits at larger widths

## Hardware

- Includes - 
    - Main memory
    - Buses
    - CPU caches
    - MMU

### Main Memory (DRAM)

- __DRAM (Dynamic Random-Access Memory)__ - 
    - Dominant main memory type today
    - Volatile - contents lost on power loss
    - High density - each bit implemented with one capacitor + one transistor
    - Capacitor requires periodic refresh to maintain charge
- __Typical DRAM sizes__ -
    - Enterprise servers - $1\,\text{GB}$ to $1\,\text{TB}$ depending on workload
    - Cloud instances - $512\,\text{MB}$ to $256\,\text{GB}$ each
    - Cloud spreads load across many instances - collectively larger DRAM pool, but at higher coherency cost

### Latency

- __CAS latency (Column Address Strobe latency)__ - 
    - Time between sending a memory module the target column address and data becoming readable
    - DDR4 typical range: $10$–$20\,\text{ns}$
    - Multiple CAS cycles may occur per cache line transfer (Eg - $64$-byte cache line over a $64$-bit bus requires several transactions)
- Additional latency introduced by CPU and MMU when consuming the fetched data
- __Read instructions__ - bypass main memory latency entirely when data is in CPU cache
- __Write instructions__ - may bypass latency too if processor supports write-back caching (Eg - Intel processors)

### Memory Architecture

- __UMA (Uniform Memory Access)__ - 
    - All CPUs share a single system bus to reach memory
    - Equal latency for all CPUs to all memory
    - __SMP (Symmetric Multiprocessing)__ - single OS kernel instance running uniformly across all CPUs in a UMA setup
- __NUMA (Non-Uniform Memory Access)__ - 
    - CPUs each have directly attached memory, connected to each other via a CPU interconnect
    - __Local memory__ - DRAM directly attached to the accessing CPU via its memory bus (1 hop)
    - __Remote memory__ - DRAM attached to a different CPU, accessed via the CPU interconnect (2+ hops, higher latency)
    - __Memory node__ - bank of DRAM connected to a specific CPU
    - OS uses node topology (provided by processor) to schedule threads and allocate memory close to local DRAM
    - Performance strategy - keep threads and their working data on the same NUMA node

### Buses

- Three connection models between CPU and main memory -
    - __Shared system bus__ - 
        - Single or multi-CPU, via shared system bus → memory bridge controller → memory bus
        - Controller historically called Northbridge (Eg - Intel front-side bus architecture)
    - __Direct__ - single CPU with memory bus directly to DRAM
    - __Interconnect__ - multi-CPU, each with local DRAM via memory bus, CPUs linked via CPU interconnect
        - This is the NUMA topology
- To identify which model a system uses, trace the data path from CPU to DRAM in the system functional diagram, noting all intermediate components

### DDR SDRAM

- __DDR SDRAM (Double Data Rate Synchronous DRAM)__ - 
    - __Double data rate__ - transfers data on both the rising and falling edge of the clock signal
        - Also called _double-pumped_
    - __Synchronous__ - memory clocked in sync with the CPU
- Speed governed by the memory interface standard supported by the processor and system board
- Naming convention - 
    - `DDR<generation>-<MT/s>`, bandwidth variant named `PC-<MB/s>`
    - Eg - DDR3-1600 → $1600\,\text{MT/s}$, marketed as PC-12800 ($12800\,\text{MB/s}$)

### Multichannel

- Multiple memory buses in parallel to increase aggregate bandwidth
- Common configurations - dual-, triple-, quad-channel
- Eg - Intel Core i7 with quad-channel DDR3-1600 → $51.2\,\text{GB/s}$ peak bandwidth

### CPU Caches

- On-chip hardware caches that reduce effective memory access latency
- Hierarchy - decreasing speed, increasing size as level increases
    - __L1__ - 
        - Fastest, smallest
        - Split into separate instruction cache (I-cache) and data cache (D-cache)
        - Typically addressed by _virtual_ memory addresses
    - __L2__ - 
        - Unified cache for instructions and data
        - Typically addressed by _physical_ memory addresses
    - __L3__ - 
        - Larger unified cache, shared across cores on most modern processors
        - Addressed by physical memory addresses

### MMU

- __MMU (Memory Management Unit)__ - 
    - Hardware unit responsible for virtual-to-physical address translation
    - Translation performed per page
    - Page offset bits mapped directly - only the page number requires translation

### Multiple Page Sizes

- Modern processors support multiple page sizes simultaneously
    - Common sizes - $4\,\text{KB}$, $2\,\text{MB}$, $1\,\text{GB}$
- __Linux huge pages__ - 
    - Kernel feature exposing large page sizes ($2\,\text{MB}$, $1\,\text{GB}$) to applications
    - Reduces number of page table entries needed for large allocations

### TLB

- __TLB (Translation Lookaside Buffer)__ - 
    - First-level cache for virtual-to-physical address mappings within the MMU
    - Falls back to page tables in main memory on a miss
    - May be split into separate caches for instruction pages and data pages
    - May have separate sub-caches per page size to retain large-page mappings independently
- __TLB reach__ - 
    - Total memory range coverable from TLB entries
    - Larger page sizes → fewer entries needed to cover the same address range → greater reach → fewer misses
- __TLB miss__ - 
    - Mapping not in TLB, requires page table walk in main memory
    - Costly - adds memory latency to every affected access
- __Multi-level TLB__ - 
    - Some processors include L1 and L2 TLBs, mirroring the CPU cache hierarchy
    - Eg - Intel Core microarchitecture supports two levels of data TLB
- TLB layout is processor-specific - consult vendor processor manuals for entry counts and page size support per level

> [!NOTE]
> L1 cache is virtually addressed to avoid requiring a translation before the first cache lookup - saving critical cycles on the fast path. L2+ are physically addressed for correctness across context switches.

> [!TIP]
> TLB pressure is a common but overlooked performance issue in large-heap JVM workloads. Enabling huge pages (2 MB) reduces TLB misses significantly - measurable via `perf stat -e dTLB-load-misses`.

## Software

### Freeing Memory

- When available memory is low, the kernel uses several techniques to reclaim pages back to the free list -
    - __Free list__ - list of unused (idle) pages available for immediate allocation, implemented as multiple per-NUMA-node free page lists
    - __Page cache reclaim__ - filesystem cache pages reclaimed before swapping, controlled by the `swappiness` tunable
    - __Swapping__ -
        - `kswapd` (page-out daemon) finds not-recently-used pages and pages them out
        - Writes to a swap file (filesystem-based) or swap device
        - Only available if a swap file or device is configured
    - __Reaping (shrinking)__ - when a low-memory threshold is crossed, kernel modules and the slab allocator are instructed to immediately free easily reclaimable memory
    - __OOM killer__ -
        - Last resort - finds and kills a sacrificial process to reclaim its memory
        - Process selected via `select_bad_process()`, killed via `oom_kill_process()`
        - Logged in `/var/log/messages` as `Out of memory: Kill process`

- __Swappiness__ -
    - Kernel tunable controlling the balance between paging application memory vs reclaiming page cache
    - Range: `0`–`100`, default `60`
        - Higher values favour paging out application memory
        - Lower values favour reclaiming filesystem cache
    - Goal - preserve warm filesystem cache while evicting cold application memory to improve throughput

- __No Swap Configured__ -
    - Limits virtual memory size - if overcommit is disabled, allocations fail sooner
    - On Linux with overcommit, OOM killer triggers sooner instead
    - Trade-off -
        - With swap - memory exhaustion first manifests as a performance degradation (paging), giving opportunity to debug live
        - Without swap - application hits OOM error or is killed immediately, potentially masking slow memory leaks that only surface after hours

> [!NOTE]
> Netflix cloud instances run without swap - OOM-killed instances are immediately removed from load balancing and traffic is rerouted to healthy instances, which is preferred over a single instance degrading under swap pressure.

- __cgroup Memory Limits__ -
    - The same freeing techniques apply within a memory cgroup
    - A system may have abundant free memory but still trigger swapping or OOM killer if a container exhausts its cgroup-imposed limit
    - Common gotcha in containerised environments - system-level memory metrics look healthy while individual containers are OOM killing

### Free Lists

- Original Unix allocator used a memory map with a first-fit scan
- BSD paged virtual memory introduced the free list + page-out daemon
- __Free list__ -
    - Allows available memory to be located immediately
    - Freed memory added to the head for future allocations
    - Memory freed by the page-out daemon (may still contain useful cached filesystem pages) added to the tail
    - If a cached page at the tail is requested before reuse, it is reclaimed and removed from the free list

### Linux Free List Hierarchy

- Linux uses the __buddy allocator__ for page management
    - Multiple free lists for different allocation sizes, following a power-of-two scheme
    - __Buddy__ - neighbouring free pages found and allocated together
    - For single-page allocations (most common), per-CPU lists maintained to reduce lock contention
- Free lists consumed by allocators -
    - __Slab allocator__ - kernel-space
    - `libc malloc()` - user-space (maintains its own internal free lists)
- Hierarchy rooted at `pg_data_t` (per-memory node) -
    - __Nodes__ - banks of memory, NUMA-aware
    - __Zones__ - ranges of memory for specific purposes (DMA, normal, highmem)
    - __Migration types__ - unmovable, reclaimable, movable, etc.
    - __Sizes__ - power-of-two number of pages (buddy allocator lists)
- Allocating within node-local free lists improves memory locality and performance

### Reaping

- Primarily frees memory from kernel slab allocator caches
    - Caches hold unused slab-sized memory chunks ready for reuse
    - Reaping returns these chunks to the system as free pages
- Kernel modules can register custom reaping functions via `register_shrinker()`

### Page Scanning

- Managed by `kswapd` (kernel page-out daemon)
- Triggered when free memory in the free list drops below a threshold
- Only occurs when needed - a balanced system scans infrequently or in short bursts
- `kswapd` scans LRU page lists (inactive first, then active) to find pages to free
- Three watermark thresholds with hysteresis -
    - __High__ - free memory is adequate, `kswapd` sleeps
    - __Low__ - `kswapd` wakes and begins background reclaim
    - __Min__ - `kswapd` runs in the foreground, synchronously freeing pages on every allocation request (direct reclaim)
- `vm.min_free_kbytes` - tunable for the min watermark
    - Low watermark = $2\times$ min, high watermark = $3\times$ min
- Additional tunables for high-burst workloads: `vm.watermark_scale_factor`, `vm.watermark_boost_factor`
- Page cache maintains separate inactive and active LRU lists - `kswapd` finds reclaimable pages quickly
- A page may be ineligible to free even if found during scan - if locked or dirty

> [!NOTE]
> `kswapd` scanning (walking LRU lists) is distinct from the original Unix page-out daemon scanning (walking all of memory).

### Process Virtual Address Space

- Range of virtual pages mapped to physical pages on demand, managed by hardware and software
- Address space divided into segments -
    - __Executable text__ - read-only, execute permission, mapped from binary text segment on filesystem
    - __Executable data__ - read/write, private flag (modifications not flushed to disk), mapped from binary data segment
    - __Heap__ - anonymous memory (no filesystem location), grows on demand via `malloc(3)`
    - __Stack__ - read/write, one per thread
- Library text segments shared across processes using the same library
    - Each process has its own private copy of the library data segment
- 64-bit x86 - AMD spec allows 48-bit address implementations, creating two canonical ranges -
    - `0x0000000000000000`–`0x00007fffffffffff` - user space
    - `0xffff800000000000`–`0xffffffffffffffff` - kernel space

### Heap Growth

- `free(3)` in simple allocators does not return memory to the OS - memory is retained for future allocations
    - Process RSS can only grow - this is normal, not necessarily a leak
- Methods to actually return memory to the OS -
    - `execve(2)` - re-exec the process, starts from empty address space
    - `mmap(2)` / `munmap(2)` - memory-mapped regions returned to OS on unmap
- glibc on Linux supports -
    - `mmap` mode for large allocations (returned to OS on free)
    - `malloc_trim(3)` - releases top-of-heap free memory via `sbrk(2)`
    - `malloc_trim(3)` called automatically by `free(3)` when free memory at top of heap exceeds `M_TRIM_THRESHOLD` (default `128 KB`)

### Allocators

- Handle memory allocation within a virtual address space
- Key properties of a good allocator -
    - Simple API - Eg - `malloc(3)`, `free(3)`
    - Efficient memory usage - coalesces fragmented unused regions for reuse
    - Performance - uses per-thread or per-CPU caches, minimises lock contention
    - Observability - exposes stats and debug modes, allocation call paths

#### Slab (kernel)

- Manages caches of fixed-size objects for fast recycling without page allocation overhead
- Effective for kernel allocations which are frequently for fixed-size structs
- Two allocation styles -
    - `kmem_alloc(size, flags)` - size-based, kernel maps to an appropriate slab cache internally
    - `kmem_cache_alloc(cache, flags)` - operates directly on a named custom slab cache
- Enhanced with per-CPU __magazine caches__ - each CPU holds `M` pre-allocated objects, reloads from a full magazine when exhausted
- Originated in Solaris 2.4, adopted by BSD (called UMA - universal memory allocator), and Linux (introduced in 2.2)

#### SLUB (kernel, Linux default)

- Linux default since kernel `2.6.23`, based on slab but simplified
- Removes per-CPU object queues and internal caches
- Delegates NUMA optimisation to the buddy page allocator instead

#### glibc (user-space)

- Based on `dlmalloc` by Doug Lea
- Allocation strategy varies by size -
    - Small - served from size-bucketed bins, coalesced via buddy-like algorithm
    - Large - tree lookup to find space efficiently
    - Very large - falls back to `mmap(2)`

#### TCMalloc (user-space)

- Thread-caching `malloc` - per-thread cache for small allocations reduces lock contention
- Periodic garbage collection migrates cached memory back to a central heap

#### jemalloc (user-space)

- Originated as FreeBSD libc allocator, available on Linux as `libjemalloc`
- Uses multiple arenas, per-thread caching, and small object slabs to improve scalability and reduce fragmentation
- Prefers `mmap(2)` over `sbrk(2)` for system memory
- Facebook extended it with profiling and additional optimisations

## Methodology

### Tools Method

- Iterates over available tools examining key metrics - may miss issues with poor tool visibility
- Key checks for Linux memory -
    - __Page scanning__ - continual scanning (>10s) indicates memory pressure
        - `sar -B`, check `pgscan` columns
    - __PSI (Pressure Stall Information)__ - `cat /proc/pressure/memory` (Linux 4.20+), shows memory saturation over time
    - __Swapping__ - `vmstat 1`, check `si` (swap-in) and `so` (swap-out) columns
    - __Free memory__ - `vmstat 1`, check `free` column
    - __OOM killer__ - search `/var/log/messages` or `dmesg(1)` for `Out of memory`
    - __Top consumers__ - `top(1)` for per-process RSS and virtual memory usage
    - __Allocation tracing__ - `perf(1)`, BCC, or `bpftrace` for stack-traced allocation profiling (high overhead)

### USE Method

- Check system-wide -
    - __Utilization__ - physical and virtual memory in use vs available
    - __Saturation__ - page scanning, paging, swapping, OOM kills
    - __Errors__ - failed allocations, OOM killer events, hardware ECC errors
- Check saturation first - continual saturation is the primary indicator of a memory issue
- Tools - `vmstat(8)`, `sar(1)` for swapping; `dmesg(1)` for OOM events; `/proc/pressure/memory` for PSI
- Physical memory reporting varies by tool - some exclude reclaimable filesystem cache, making available memory appear artificially low
- Virtual memory utilisation matters on systems without overcommit - exhaustion causes `malloc()` to fail (`ENOMEM`)
- Hardware ECC errors - `dmidecode(8)`, `edac-utils`, `ipmitool sel` - correctable ECC errors are a leading indicator of uncorrectable failures
    - Uncorrectable errors manifest as unexplained crashes, segfaults, or bus error signals

> [!NOTE]
> In cgroup/container environments, the OS instance may hit its software memory limit and begin swapping even when the host has abundant physical memory. Measure per-cgroup utilisation and saturation separately.

### Characterising Usage

- Key attributes to measure -
    - System-wide physical and virtual memory utilisation
    - Degree of saturation - swapping rate, OOM kill frequency
    - Kernel and filesystem cache memory usage
    - Per-process physical (RSS) and virtual memory usage
    - Memory resource control usage, if present
- Memory usage varies over time - cache warmup is normal growth, leaks are not
- Advanced checklist -
    - What is the WSS for each application?
    - How is kernel memory distributed across slabs?
    - What ratio of filesystem cache is active vs inactive?
    - What are the allocation call paths for processes and kernel?
    - Are any processes or kernel modules growing without bound (leak)?
    - In NUMA systems - how balanced is memory distribution across nodes?
    - What are IPC and memory stall cycle rates?
    - How balanced are the memory buses?
    - What is the ratio of local vs remote memory I/O?

### Cycle Analysis

- Memory bus load measured via CPU PMCs (performance monitoring counters)
- Key starting metric - IPC (instructions per cycle), reflects memory dependency of workload
- Low IPC → high memory stall cycles → memory-bound workload

### Performance Monitoring

- Key metrics to track over time -
    - __Utilisation__ - percent used, inferred from available memory
    - __Saturation__ - swap activity, OOM kills
- Monitor per-process memory growth over time to detect leaks

### Leak Detection

- Two root causes -
    - __Memory leak__ - memory no longer used but never freed, software bug, fixed by code change or patch
    - __Memory growth__ - memory consumed normally but at excessive rate, fixed by configuration or code change
- Memory growth is frequently misidentified as a leak - first verify whether the growth is expected (Eg - cache warmup)
- Analysis approaches -
    - Allocator debug modes - record allocation details for postmortem analysis
    - Heap dump analysis - runtime-specific (JVM, etc.)
    - `memleak(8)` (BCC) - tracks allocations not freed within an interval, with allocation call paths
        - Cannot distinguish leaks from normal growth - requires manual call path analysis
        - High overhead at high allocation rates

### Static Performance Tuning

- Configuration checklist -
    - Total main memory available
    - Per-application memory configuration limits
    - Allocators in use per application
    - Memory speed - DDR generation, is it the fastest available?
    - Has memory been fully tested - Eg - `memtester`
    - Architecture - UMA vs NUMA
    - Is OS NUMA-aware, are NUMA tunables configured?
    - Memory socket topology - same socket or split across sockets
    - Number of memory buses
    - CPU cache sizes and TLB sizes
    - BIOS memory settings
    - Huge pages - configured and in use?
    - Overcommit - enabled and configured?
    - Active memory tunables
    - Software memory limits (resource controls / cgroups)

### Micro-Benchmarking

- Used to measure main memory speed and CPU cache/cache line characteristics
- Useful when comparing systems - memory latency can dominate performance more than CPU clock speed depending on workload
- Results can reveal CPU cache sizes, cache line sizes, and NUMA latency differences

### Memory Shrinking (WSS Estimation)

- Negative experiment - progressively reduces available memory while measuring performance and swap activity
- WSS identified at the point where performance sharply degrades and swapping spikes
- Requires swap to be configured
- Not recommended for production - deliberately induces performance degradation
- Alternative - `wss(8)` tool for non-destructive WSS estimation

## Memory Observability Tools

## vmstat

- Virtual memory statistics command — high-level view of system memory health, free memory, and paging activity
- Columns (kilobytes by default) —

| Column | Description |
|--------|-------------|
| `swpd` | Amount of swapped-out memory |
| `free` | Free available memory |
| `buff` | Memory in the buffer cache |
| `cache` | Memory in the page cache |
| `si` | Memory swapped in (paging) |
| `so` | Memory swapped out (paging) |

- First line shows current status immediately, not summary-since-boot
- Continually non-zero `si`/`so` — system under memory pressure, actively swapping

```bash
vmstat 1              # 1-second interval, kilobytes
vmstat -Sm 1          # output in megabytes (m = 1000000, M = 1048576)
vmstat -a 1           # inactive/active page cache breakdown
vmstat -m             # print slab statistics (same source as slabtop)
vmstat -s             # print memory stats as a list
```

> [!NOTE]
> Low free memory post-boot is normal — kernel uses spare memory for filesystem cache, reclaimed when applications need it.

## PSI (Pressure Stall Information)

- Linux 4.20+ — memory saturation statistics, shows pressure trend over time
- Read from `/proc/pressure/memory`
- Two lines —
    - `some` — percentage of time at least one task was stalled on memory
    - `full` — percentage of time all runnable tasks were stalled on memory (more severe)
- Three averages per line — `avg10`, `avg60`, `avg300` (10s, 60s, 300s windows)
- Rising `avg10` vs `avg300` indicates worsening pressure
- Also tracked per `cgroup2` — useful in containerised environments

```bash
cat /proc/pressure/memory
# some avg10=2.84 avg60=1.23 avg300=0.32 total=1468344
# full avg10=1.85 avg60=0.66 avg300=0.16 total=702578
```

## swapon

- Shows configured swap devices and their current utilisation
- No output means no swap configured
- Active swap I/O visible via `si`/`so` in `vmstat(1)` and as device I/O in `iostat(1)`

```bash
swapon
# NAME      TYPE      SIZE   USED PRIO
# /dev/dm-2 partition 980M 611.6M   -2
# /swap1    file       30G  10.9M   -3
```

## sar

- System activity reporter — observe current activity and archive historical statistics
- Memory-relevant options —

| Option | Description |
|--------|-------------|
| `-B` | Paging statistics |
| `-H` | Huge pages statistics |
| `-r` | Memory utilisation |
| `-S` | Swap space statistics |
| `-W` | Swapping statistics |

### `-B` Paging Statistics

| Statistic | Description | Units |
|-----------|-------------|-------|
| `pgpgin/s` | Page-ins | KB/s |
| `pgpgout/s` | Page-outs | KB/s |
| `fault/s` | Major and minor faults combined | count/s |
| `majflt/s` | Major faults only | count/s |
| `pgfree/s` | Pages added to free list | count/s |
| `pgscank/s` | Pages scanned by `kswapd` background daemon | count/s |
| `pgscand/s` | Direct page scans — application blocking on allocation entering direct reclaim | count/s |
| `pgsteal/s` | Page and swap cache reclaims | count/s |
| `%vmeff` | Ratio of page steal / page scan — page reclaim efficiency | % |

- `%vmeff` interpretation —
    - Near 100% — healthy, pages successfully stolen from inactive list
    - Below 30% — system struggling to reclaim memory

### `-r` Memory Utilisation Statistics

| Statistic | Description | Units |
|-----------|-------------|-------|
| `kbmemfree` | Completely unused memory | KB |
| `kbavail` | Available memory including readily reclaimable page cache | KB |
| `kbmemused` | Used memory excluding kernel | KB |
| `%memused` | Memory usage | % |
| `kbbuffers` | Buffer cache size | KB |
| `kbcached` | Page cache size | KB |
| `kbcommit` | Estimate of memory needed for current workload | KB |
| `%commit` | Committed memory as percentage | % |
| `kbactive` | Active list memory size | KB |
| `kbinact` | Inactive list memory size | KB |
| `kbdirtyw` | Modified memory pending write to disk | KB |
| `kbanonpg` | Process anonymous memory (`-r ALL`) | KB |
| `kbslab` | Kernel slab cache size (`-r ALL`) | KB |
| `kbkstack` | Kernel stack space size (`-r ALL`) | KB |
| `kbpgtbl` | Lowest-level page table size (`-r ALL`) | KB |
| `kbvmused` | Used virtual address space (`-r ALL`) | KB |

### `-S` Swap Statistics

| Statistic | Description | Units |
|-----------|-------------|-------|
| `kbswpfree` | Free swap space | KB |
| `kbswpused` | Used swap space | KB |
| `%swpused` | Used swap percentage | % |
| `kbswpcad` | Cached swap — resides in both main memory and swap device, can be paged out without disk I/O | KB |
| `%swpcad` | Ratio of cached swap vs used swap | % |

### `-W` Swapping Statistics

| Statistic | Description | Units |
|-----------|-------------|-------|
| `pswpin/s` | Swap-ins | pages/s |
| `pswpout/s` | Swap-outs | pages/s |

> [!NOTE]
> `pgscand` is critical — shows rate at which applications block on memory allocation and enter direct reclaim. Use `drsnoop` to measure time spent in direct reclaim.

> [!TIP]
> Deeper analysis beyond `sar` requires tracing `mm/vmscan.c` kernel functions via `perf(1)` or `bpftrace`.

## slabtop

- Prints kernel slab cache usage in real time (like `top` for slabs)
- Source data from `/proc/slabinfo` — also accessible via `vmstat -m`
- Sort options — `-sc` sorts by cache size (largest first)
- Per-slab columns —

| Column | Description |
|--------|-------------|
| `OBJS` | Total object count |
| `ACTIVE` | Active (in-use) object count |
| `USE` | Percentage of objects in use |
| `OBJ SIZE` | Size of each object (bytes) |
| `SLABS` | Number of slabs |
| `OBJ/SLAB` | Objects per slab |
| `CACHE SIZE` | Total cache size (bytes) |
| `NAME` | Cache name |

```bash
slabtop -sc    # sort by cache size, largest first
```

## numastat

- NUMA memory allocation statistics — typically used on multi-socket systems
- Linux tries to allocate memory on the nearest NUMA node — `numastat` shows how successful this is

| Statistic | Description |
|-----------|-------------|
| `numa_hit` | Allocations on the intended NUMA node |
| `numa_miss` | Local allocations that should have been on a different node |
| `numa_foreign` | Remote allocations that should have been local |
| `interleave_hit` | Interleaved allocations that landed on the intended node |
| `local_node` | Allocations on the local node for the running process |
| `other_node` | Allocations on this node while the process ran elsewhere |

- Low hit ratio — consider adjusting NUMA tunables via `sysctl(8)`, partitioning workloads, or reducing NUMA nodes
- Options —
    - `-n` — print in MB
    - `-m` — output in `/proc/meminfo` style
    - Available via `numactl` package on some distributions

## ps

- Lists per-process memory usage statistics
- Memory columns —

| Column | Description |
|--------|-------------|
| `%MEM` | RSS as percentage of total system memory |
| `RSS` | Resident set size (KB) |
| `VSZ` | Virtual memory size (KB) |
| `maj_flt` | Major page faults (Linux) |
| `min_flt` | Minor page faults (Linux) |

- RSS includes shared memory segments (system libraries) — summing RSS across processes can exceed total system memory due to shared memory overcounting
- Use `pmap(1)` to analyse shared memory per process

```bash
ps aux                          # BSD-style, includes %MEM, RSS, VSZ
ps -eo pid,pmem,vsz,rss,comm    # SVR4-style, custom columns
```

## top

- Interactive per-process monitor with memory statistics
- Summary header shows — total, used, free for main memory (`Mem`) and swap (`Swap`), plus buffer cache (`buffers`) and page cache (`cached`) sizes
- Per-process memory columns —

| Column | Description |
|--------|-------------|
| `%MEM` | RSS as percentage of total system memory |
| `VIRT` | Virtual memory size |
| `RES` | Resident set size (main memory in use) |
| `SHR` | Shared memory size |

```bash
top -o %MEM    # sort by memory usage descending
```

> [!TIP]
> Type `?` in `top` for built-in interactive command summary. See "Linux Memory Types" in the `top(1)` man page for full column definitions.

## pmap

- Lists all memory mappings of a process — sizes, permissions, and mapped objects
- Used to examine process memory in detail and quantify shared memory
- Key columns (`-x` extended mode) —

| Column | Description |
|--------|-------------|
| `Address` | Virtual memory address |
| `Kbytes` | Virtual memory size (KB) |
| `RSS` | Resident set size (KB) |
| `Dirty` | Private anonymous (dirty) memory (KB) |
| `Mode` | Permissions — r=read, w=write, x=execute, s=shared, p=private |
| `Mapping` | Mapped object — filename or `[anon]` / `[stack]` |

- `-x` — extended fields
- `-X` — even more fields including `Pss`, `Referenced`, `Anonymous`, `LazyFree`, `Swap`, `SwapPss`
- `-XX` — everything the kernel provides including `KernelPageSize`, `MMUPageSize`, `VmFlags`
- `Pss` (Proportional Set Size) — private memory + (shared memory / number of sharers) — more accurate than RSS for real memory cost
- Read-only mappings (`r----`) shared across processes — system libraries appear once in physical memory regardless of how many processes map them
- Heap appears as a wave of `[ anon ]` segments

```bash
pmap -x <pid>               # extended fields
pmap -X $(pgrep mysqld)     # more fields including Pss, Swap
pmap -XX $(pgrep mysqld)    # everything the kernel provides
```

## perf

- Official Linux profiler — multi-tool covering PMCs, tracepoints, and stack sampling for memory analysis

### One-Liners

```bash
perf record -e page-faults -a -g                        # sample page faults system-wide with stack traces
perf record -e page-faults -c 1 -p 1843 -g -- sleep 60 # all page faults for PID 1843 for 60s
perf record -e syscalls:sys_enter_brk -a -g             # heap growth via brk(2)
perf record -e migrate:mm_migrate_pages -a              # page migrations on NUMA systems
perf stat -e 'kmem:*' -a -I 1000                        # count all kmem events every second
perf stat -e 'vmscan:*' -a -I 1000                      # count all vmscan events every second
perf stat -e 'compaction:*' -a -I 1000                  # count all compaction events every second
perf record -e vmscan:mm_vmscan_wakeup_kswapd -ag       # trace kswapd wakeup with stack traces
perf mem record <command>                               # profile memory accesses for a command
perf mem report                                         # summarise a memory profile
```

### Page Fault Sampling

- Page faults occur as a process increases RSS — stack traces reveal why main memory is growing
- `-a` samples system-wide (default since Linux 4.11)
- `-c 1` records every single fault (not sampled)
- Raw output is large — use `perf report` for hierarchical summary or flame graphs for full visualisation

```bash
perf record -e page-faults -a -g -- sleep 60
perf script --header > out.stacks
perf report    # hierarchical summary
```

### Page Fault Flame Graphs

- Green background convention — memory flame graph (vs yellow for CPU)
- Width of a tower represents number of page faults attributed to that code path
- Eg - `JOIN::optimize()` responsible for 3,226 page faults = ~12 MB of memory growth (3226 × 4 KB pages)

```bash
perf record -e page-faults -a -g -- sleep 60
perf script --header > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-perf.pl < ../out.stacks | ./flamegraph.pl --hash \
    --bgcolor=green --count=pages --title="Page Fault Flame Graph" > out.svg
```

> [!NOTE]
> System-wide `-a` tracing captures short-lived processes — use `-p PID` to restrict to a single process.

## drsnoop

- BCC tool — traces direct reclaim events, showing which process was affected and latency incurred
- Direct reclaim — synchronous path where an application blocks on memory allocation while the kernel frees pages
- Quantifies application-visible performance impact of memory pressure
- Traces `mm_vmscan_direct_reclaim_begin` and `mm_vmscan_direct_reclaim_end` tracepoints
- Low-frequency events (burst pattern) — overhead negligible

```bash
drsnoop -T          # include timestamps
drsnoop -p <pid>    # filter to single process
```

- Output columns —

| Column | Description |
|--------|-------------|
| `TIME(s)` | Timestamp |
| `COMM` | Process name |
| `PID` | Process ID |
| `LAT(ms)` | Latency of the direct reclaim event |
| `PAGES` | Pages reclaimed |

> [!NOTE]
> `sar -B` `pgscand` shows the rate of direct reclaim — `drsnoop` shows the actual latency per event. Use both together.

## wss

- Experimental tool — measures process Working Set Size (WSS) using the PTE "accessed" bit
- WSS — amount of memory a process frequently accesses to perform work
- Not directly reported by any standard tool — RSS overstates, WSS is what matters for sizing

### Mechanism

- Resets the PTE accessed bit for every page in the process
- Pauses for a measurement interval
- Checks which bits were set — those pages were accessed during the interval
- Resolution is page size (typically 4 KB) — values rounded up to page boundary

### Output Columns

| Column | Description |
|--------|-------------|
| `Est(s)` | Estimated interval including time to set/read accessed bits |
| `RSS(MB)` | Resident set size |
| `PSS(MB)` | Proportional set size (accounts for shared pages) |
| `Ref(MB)` | Pages referenced during the interval — the WSS estimate |

```bash
./wss.pl $(pgrep -n mysqld) 1    # measure mysqld WSS, print every 1 second
```

> [!NOTE]
> Uses `/proc/PID/clear_refs` and `/proc/PID/smaps` — causes ~10% higher application latency while kernel walks page structures. For processes >100 GB, elevated latency can last over one second. Resets the referenced flag which may confuse kernel page reclaim decisions, especially under active swapping. Test in a lab environment first.

## bpftrace

- BPF-based tracer — high-level language for one-liners and short scripts
- Advantage over `perf` for stack traces — aggregates in kernel space, only unique stacks and counts written to user space (lower overhead)

### One-Liners

```bash
# Sum malloc() request bytes by user stack and process (high overhead)
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc { @[ustack, comm] = sum(arg0); }'

# Sum malloc() request bytes by user stack for PID 181 (high overhead)
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 181/ { @[ustack] = sum(arg0); }'

# malloc() request bytes as power-of-2 histogram by user stack for PID 181 (high overhead)
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 181/ { @[ustack] = hist(arg0); }'

# Sum kernel kmem cache allocation bytes by kernel stack
bpftrace -e 't:kmem:kmem_cache_alloc { @bytes[kstack] = sum(args->bytes_alloc); }'

# Count heap expansion brk(2) by code path
bpftrace -e 'tracepoint:syscalls:sys_enter_brk { @[ustack, comm] = count(); }'

# Count page faults by process
bpftrace -e 'software:page-fault:1 { @[comm, pid] = count(); }'

# Count user page faults by user-level stack trace
bpftrace -e 't:exceptions:page_fault_user { @[ustack, comm] = count(); }'

# Count vmscan operations by tracepoint
bpftrace -e 'tracepoint:vmscan:* { @[probe] = count(); }'

# Count swapins by process
bpftrace -e 'kprobe:swap_readpage { @[comm, pid] = count(); }'

# Count page migrations
bpftrace -e 'tracepoint:migrate:mm_migrate_pages { @ = count(); }'

# Trace compaction events
bpftrace -e 't:compaction:mm_compaction_begin { time(); }'

# List USDT probes in libc
bpftrace -l 'usdt:/lib/x86_64-linux-gnu/libc.so.6:*'

# List kernel kmem tracepoints
bpftrace -l 't:kmem:*'

# List all mm_ tracepoints
bpftrace -l 't:*:mm_*'
```

### User Allocation Stacks

- Trace `malloc(3)` with `hist(arg0)` keyed by `ustack` — shows allocation size distribution per code path
- Eg - output bucket `[32K, 64K) 676` means 676 `malloc()` calls of 32–64 KB from that stack

```bash
bpftrace -e 'uprobe:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 4840/ {
    @[ustack] = hist(arg0); }'
```

### malloc() Bytes Flame Graph

```bash
bpftrace -e 'u:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 4840/ {
    @[ustack] = hist(arg0); }' > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-bpftrace.pl < ../out.stacks | ./flamegraph.pl --hash \
    --bgcolor=green --count=bytes --title="malloc() Bytes Flame Graph" > out.svg
```

> [!NOTE]
> `malloc()` uprobes are very high overhead — millions of calls per second, can slow target 2x or more. Start with CPU profiling or page fault tracing instead, then drill into allocation tracing only for targeted investigation.

### Page Fault Flame Graphs

```bash
bpftrace -e 't:exceptions:page_fault_user { @[ustack, comm] = count(); }' > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-bpftrace.pl < ../out.stacks | ./flamegraph.pl --hash \
    --bgcolor=green --count=pages --title="Page Fault Flame Graph" > out.svg
```

### Memory Internals Probe Discovery

```bash
bpftrace -l 'tracepoint:kmem:*'    # 12 kmem tracepoints on kernel 5.3
bpftrace -l 't:*:mm_*'             # 47 mm_ tracepoints on kernel 5.3
bpftrace -l 'usdt:/lib/x86_64-linux-gnu/libc.so.6'    # 33 USDT probes in libc
bpftrace -lv tracepoint:kmem:kmalloc    # list arguments for a specific tracepoint
```

- Probe types available for memory analysis —
    - `tracepoint` — stable kernel tracepoints (`kmem:*`, `vmscan:*`, `mm_*`)
    - `uprobe` / `uretprobe` — dynamic user-space instrumentation (Eg - `malloc`)
    - `kprobe` / `kretprobe` — dynamic kernel instrumentation
    - `usdt` — user-space statically defined tracing probes in libraries
    - `watchpoint` — fires on read, write, or execute of a specific memory address

> [!NOTE]
> If tracepoints and USDT probes are insufficient, fall back to `kprobes` and `uprobes` for dynamic instrumentation. Memory events can be extremely frequent — always use maps to summarise rather than per-event output.

## Other Tools

### BPF Performance Tools Reference

| Tool | Description |
|------|-------------|
| `pmcarch` | CPU cycle usage including LLC misses |
| `tlbstat` | Summarises TLB cycles |
| `free` | Cache capacity statistics |
| `cachestat` | Page cache statistics |
| `oomkill` | Shows extra info on OOM kill events |
| `memleak` | Shows possible memory leak code paths |
| `mmapsnoop` | Traces `mmap(2)` calls system-wide |
| `brkstack` | Shows `brk()` calls with user stack traces |
| `shmsnoop` | Traces shared memory calls with details |
| `faults` | Shows page faults by user stack trace |
| `ffaults` | Shows page faults by filename |
| `vmscan` | Measures VM scanner shrink and reclaim times |
| `swapin` | Shows swap-ins by process |
| `hfaults` | Shows huge page faults by process |

### Additional Linux Tools and Sources

| Tool / Source | Description |
|---------------|-------------|
| `dmesg` | Check for `Out of memory` messages from OOM killer |
| `dmidecode` | BIOS information for memory banks — DDR type, speed, voltage, manufacturer |
| `tiptop` | `top(1)` variant displaying PMC statistics per process |
| `valgrind` / `memcheck` | User-level allocator wrapper for leak detection — 20-30x slowdown |
| `iostat` | If swap device is a physical disk, swap I/O visible as device I/O |
| `/proc/zoneinfo` | Statistics for memory zones (DMA, normal, highmem) |
| `/proc/buddyinfo` | Statistics for the kernel buddy allocator |
| `/proc/pagetypeinfo` | Kernel free memory page statistics — useful for diagnosing memory fragmentation |
| `/sys/devices/system/node/node*/numastat` | Per-NUMA-node statistics |
| `SysRq m` | Dumps memory info to console — useful when system is locked up |

### SysRq Memory Dump

```bash
echo m > /proc/sysrq-trigger
dmesg    # view output — shows active_anon, inactive_anon, active_file, slab_reclaimable, free, dirty, etc.
```

- Useful when system has locked up — can trigger via SysRq key sequence on physical console
- Output fields include — `active_anon`, `inactive_anon`, `active_file`, `inactive_file`, `unevictable`, `dirty`, `slab_reclaimable`, `slab_unreclaimable`, `free`, `pagetables` per NUMA node with watermark values

### dmidecode Memory Bank Output

```bash
dmidecode    # shows Type (DDR4/DDR5), Speed (MT/s), Manufacturer, Voltage, Form Factor per DIMM
```

- Key for static performance tuning — verifies DDR generation, configured speed, number of ranks
- Typically unavailable inside cloud guest VMs

## Memory Tuning

## Tunable Parameters

- Set via `sysctl(8)` — documented in `Documentation/sysctl/vm.txt` in kernel source
- `dirty_background_bytes` and `dirty_background_ratio` are mutually exclusive
- `dirty_bytes` and `dirty_ratio` are mutually exclusive — setting one overrides the other

| Parameter | Default | Description |
|-----------|---------|-------------|
| `vm.dirty_background_bytes` | 0 | Amount of dirty memory to trigger `pdflush` background write-back |
| `vm.dirty_background_ratio` | 10 | Percentage of dirty system memory to trigger `pdflush` background write-back |
| `vm.dirty_bytes` | 0 | Amount of dirty memory that causes a writing process to start write-back |
| `vm.dirty_ratio` | 20 | Ratio of dirty system memory to cause a writing process to begin write-back |
| `vm.dirty_expire_centisecs` | 3000 | Minimum time (centiseconds) for dirty memory to be eligible for `pdflush` — promotes write cancellation |
| `vm.dirty_writeback_centisecs` | 500 | `pdflush` wake-up interval in centiseconds — 0 to disable |
| `vm.min_free_kbytes` | dynamic | Desired free memory amount — some kernel atomic allocations consume from this pool |
| `vm.watermark_scale_factor` | 10 | Distance between `kswapd` watermarks (min, low, high) — unit is fractions of 10000, so 10 = 0.1% of system memory |
| `vm.watermark_boost_factor` | 5000 | How far past the high watermark `kswapd` scans when memory is fragmented — unit is fractions of 10000, so 5000 = up to 150% of high watermark — 0 to disable |
| `vm.percpu_pagelist_fraction` | 0 | Overrides max fraction of pages allocatable to per-CPU page lists — value of 10 limits to 1/10th of pages |
| `vm.overcommit_memory` | 0 | 0 = heuristic overcommit, 1 = always overcommit, 2 = never overcommit |
| `vm.swappiness` | 60 | Degree to favour swapping over reclaiming from page cache — 0 retains application memory as long as possible |
| `vm.vfs_cache_pressure` | 100 | Degree to reclaim cached directory and inode objects — 0 = never reclaim (risks OOM) |
| `kernel.numa_balancing` | 1 | Enables automatic NUMA page balancing |
| `kernel.numa_balancing_scan_size_mb` | 256 | MB of pages scanned per NUMA balancing scan |

### Key Tuning Notes

- `vm.min_free_kbytes` —
    - Set dynamically as a non-linear fraction of main memory (see `mm/page_alloc.c`)
    - Reducing it frees memory for applications but risks kernel being overwhelmed under pressure, triggering OOM sooner
    - Increasing it helps avoid OOM kills
- `vm.overcommit_memory` —
    - Set to `2` to disable overcommit and prevent OOM caused by overcommit
    - Per-process OOM control via `/proc/<pid>/oom_adj` or `/proc/<pid>/oom_score_adj` (see `Documentation/filesystems/proc.txt`)
- `vm.swappiness` —
    - Setting to `0` retains application memory as long as possible at the expense of page cache
    - Kernel can still swap under memory shortage even at `0`
- `kernel.numa_balancing` —
    - Was set to `0` at Netflix on Linux ~3.13 due to overly aggressive NUMA scanning consuming too much CPU
    - Fixed in later kernels — use `kernel.numa_balancing_scan_size_mb` to tune aggressiveness

## Multiple Page Sizes

- Large pages improve TLB hit ratio by increasing TLB reach — reduces TLB misses for large-heap workloads
- Linux calls large pages __huge pages__ — typically 2 MB (default page is 4 KB)
- Reference — `Documentation/vm/hugetlbpage.txt`

### Configuring Huge Pages

```bash
# Allocate 50 huge pages
echo 50 > /proc/sys/vm/nr_hugepages

# Verify
grep Huge /proc/meminfo
# HugePages_Total: 50
# HugePages_Free:  50
# Hugepagesize:    2048 kB
```

### Application Consumption Methods

- `shmget(2)` with `SHM_HUGETLBS` flag — via shared memory segments
- `hugetlbfs` filesystem — mount and map memory from it

```bash
mkdir /mnt/hugetlbfs
mount -t hugetlbfs none /mnt/hugetlbfs -o pagesize=2048K
```

- `mmap(2)` with `MAP_ANONYMOUS|MAP_HUGETLB` flags
- `libhugetlbfs` API

### Transparent Huge Pages (THP)

- Kernel mechanism — automatically promotes and demotes normal pages to huge pages
- No application changes required
- Reference — `Documentation/vm/transhuge.txt`, `admin-guide/mm/transhuge.rst`

> [!NOTE]
> THP historically had performance issues deterring its use — verify behaviour on your kernel version before relying on it in production.

## Allocators

- Different user-level allocators can be selected at compile time or runtime via `LD_PRELOAD`
- Useful for improving performance in multithreaded applications

```bash
# Select libtcmalloc at runtime
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc_minimal.so.4
```

- Place in application startup script for persistent effect

## NUMA Binding

- `numactl(8)` — bind processes to specific NUMA nodes to improve memory locality
- Useful for applications that fit within a single NUMA node's memory
- Combine `--membind` with `--physcpubind` to restrict both memory and CPU to the same socket — avoids CPU interconnect latency penalty

```bash
numactl --membind=0 3161           # bind PID 3161 to NUMA node 0
numactl --membind=0 --physcpubind=0 3161   # also restrict to CPUs on node 0
```

> [!NOTE]
> With `--membind`, memory allocations fail if they cannot be satisfied from the bound node. Use `numastat(8)` to list available NUMA nodes.

## Resource Controls

- Basic limits via `ulimit(1)` — main memory limit, virtual memory limit
- System-wide configuration in `/etc/security/limits.conf`

### cgroups Memory Controls

| Control | Description |
|---------|-------------|
| `memory.limit_in_bytes` | Maximum user memory including file cache (bytes) |
| `memory.memsw.limit_in_bytes` | Maximum memory and swap combined (bytes) — when swap is in use |
| `memory.kmem.limit_in_bytes` | Maximum kernel memory (bytes) |
| `memory.tcp.limit_in_bytes` | Maximum TCP buffer memory (bytes) |
| `memory.swappiness` | Per-cgroup swappiness — equivalent to `vm.swappiness` |
| `memory.oom_control` | 0 = OOM killer enabled for cgroup, 1 = OOM killer disabled |