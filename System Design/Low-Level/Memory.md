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

- __DDR SDRAM (Double Data Rate Synchronous DRAM)__ — 
    - __Double data rate__ — transfers data on both the rising and falling edge of the clock signal
        - Also called _double-pumped_
    - __Synchronous__ — memory clocked in sync with the CPU
- Speed governed by the memory interface standard supported by the processor and system board
- Naming convention — 
    - `DDR<generation>-<MT/s>`, bandwidth variant named `PC-<MB/s>`
    - Eg - DDR3-1600 → $1600\,\text{MT/s}$, marketed as PC-12800 ($12800\,\text{MB/s}$)

### Multichannel

- Multiple memory buses in parallel to increase aggregate bandwidth
- Common configurations — dual-, triple-, quad-channel
- Eg - Intel Core i7 with quad-channel DDR3-1600 → $51.2\,\text{GB/s}$ peak bandwidth

### CPU Caches

- On-chip hardware caches that reduce effective memory access latency
- Hierarchy — decreasing speed, increasing size as level increases
    - __L1__ — 
        - Fastest, smallest
        - Split into separate instruction cache (I-cache) and data cache (D-cache)
        - Typically addressed by _virtual_ memory addresses
    - __L2__ — 
        - Unified cache for instructions and data
        - Typically addressed by _physical_ memory addresses
    - __L3__ — 
        - Larger unified cache, shared across cores on most modern processors
        - Addressed by physical memory addresses

### MMU

- __MMU (Memory Management Unit)__ — 
    - Hardware unit responsible for virtual-to-physical address translation
    - Translation performed per page
    - Page offset bits mapped directly — only the page number requires translation

### Multiple Page Sizes

- Modern processors support multiple page sizes simultaneously
    - Common sizes — $4\,\text{KB}$, $2\,\text{MB}$, $1\,\text{GB}$
- __Linux huge pages__ — 
    - Kernel feature exposing large page sizes ($2\,\text{MB}$, $1\,\text{GB}$) to applications
    - Reduces number of page table entries needed for large allocations

### TLB

- __TLB (Translation Lookaside Buffer)__ — 
    - First-level cache for virtual-to-physical address mappings within the MMU
    - Falls back to page tables in main memory on a miss
    - May be split into separate caches for instruction pages and data pages
    - May have separate sub-caches per page size to retain large-page mappings independently
- __TLB reach__ — 
    - Total memory range coverable from TLB entries
    - Larger page sizes → fewer entries needed to cover the same address range → greater reach → fewer misses
- __TLB miss__ — 
    - Mapping not in TLB, requires page table walk in main memory
    - Costly — adds memory latency to every affected access
- __Multi-level TLB__ — 
    - Some processors include L1 and L2 TLBs, mirroring the CPU cache hierarchy
    - Eg - Intel Core microarchitecture supports two levels of data TLB
- TLB layout is processor-specific — consult vendor processor manuals for entry counts and page size support per level

> [!NOTE]
> L1 cache is virtually addressed to avoid requiring a translation before the first cache lookup — saving critical cycles on the fast path. L2+ are physically addressed for correctness across context switches.

> [!TIP]
> TLB pressure is a common but overlooked performance issue in large-heap JVM workloads. Enabling huge pages (2 MB) reduces TLB misses significantly — measurable via `perf stat -e dTLB-load-misses`.

## Software

### Freeing Memory

- When available memory is low, the kernel uses several techniques to reclaim pages back to the free list —
    - __Free list__ — list of unused (idle) pages available for immediate allocation, implemented as multiple per-NUMA-node free page lists
    - __Page cache reclaim__ — filesystem cache pages reclaimed before swapping, controlled by the `swappiness` tunable
    - __Swapping__ —
        - `kswapd` (page-out daemon) finds not-recently-used pages and pages them out
        - Writes to a swap file (filesystem-based) or swap device
        - Only available if a swap file or device is configured
    - __Reaping (shrinking)__ — when a low-memory threshold is crossed, kernel modules and the slab allocator are instructed to immediately free easily reclaimable memory
    - __OOM killer__ —
        - Last resort — finds and kills a sacrificial process to reclaim its memory
        - Process selected via `select_bad_process()`, killed via `oom_kill_process()`
        - Logged in `/var/log/messages` as `Out of memory: Kill process`

- __Swappiness__ -
    - Kernel tunable controlling the balance between paging application memory vs reclaiming page cache
    - Range: `0`–`100`, default `60`
        - Higher values favour paging out application memory
        - Lower values favour reclaiming filesystem cache
    - Goal — preserve warm filesystem cache while evicting cold application memory to improve throughput

- __No Swap Configured__ -
    - Limits virtual memory size — if overcommit is disabled, allocations fail sooner
    - On Linux with overcommit, OOM killer triggers sooner instead
    - Trade-off —
        - With swap — memory exhaustion first manifests as a performance degradation (paging), giving opportunity to debug live
        - Without swap — application hits OOM error or is killed immediately, potentially masking slow memory leaks that only surface after hours

> [!NOTE]
> Netflix cloud instances run without swap — OOM-killed instances are immediately removed from load balancing and traffic is rerouted to healthy instances, which is preferred over a single instance degrading under swap pressure.

- __cgroup Memory Limits__ -
    - The same freeing techniques apply within a memory cgroup
    - A system may have abundant free memory but still trigger swapping or OOM killer if a container exhausts its cgroup-imposed limit
    - Common gotcha in containerised environments — system-level memory metrics look healthy while individual containers are OOM killing

### Free Lists

- Original Unix allocator used a memory map with a first-fit scan
- BSD paged virtual memory introduced the free list + page-out daemon
- __Free list__ —
    - Allows available memory to be located immediately
    - Freed memory added to the head for future allocations
    - Memory freed by the page-out daemon (may still contain useful cached filesystem pages) added to the tail
    - If a cached page at the tail is requested before reuse, it is reclaimed and removed from the free list

### Linux Free List Hierarchy

- Linux uses the __buddy allocator__ for page management
    - Multiple free lists for different allocation sizes, following a power-of-two scheme
    - __Buddy__ — neighbouring free pages found and allocated together
    - For single-page allocations (most common), per-CPU lists maintained to reduce lock contention
- Free lists consumed by allocators —
    - __Slab allocator__ — kernel-space
    - `libc malloc()` — user-space (maintains its own internal free lists)
- Hierarchy rooted at `pg_data_t` (per-memory node) —
    - __Nodes__ — banks of memory, NUMA-aware
    - __Zones__ — ranges of memory for specific purposes (DMA, normal, highmem)
    - __Migration types__ — unmovable, reclaimable, movable, etc.
    - __Sizes__ — power-of-two number of pages (buddy allocator lists)
- Allocating within node-local free lists improves memory locality and performance

### Reaping

- Primarily frees memory from kernel slab allocator caches
    - Caches hold unused slab-sized memory chunks ready for reuse
    - Reaping returns these chunks to the system as free pages
- Kernel modules can register custom reaping functions via `register_shrinker()`

### Page Scanning

- Managed by `kswapd` (kernel page-out daemon)
- Triggered when free memory in the free list drops below a threshold
- Only occurs when needed — a balanced system scans infrequently or in short bursts
- `kswapd` scans LRU page lists (inactive first, then active) to find pages to free
- Three watermark thresholds with hysteresis —
    - __High__ — free memory is adequate, `kswapd` sleeps
    - __Low__ — `kswapd` wakes and begins background reclaim
    - __Min__ — `kswapd` runs in the foreground, synchronously freeing pages on every allocation request (direct reclaim)
- `vm.min_free_kbytes` — tunable for the min watermark
    - Low watermark = $2\times$ min, high watermark = $3\times$ min
- Additional tunables for high-burst workloads: `vm.watermark_scale_factor`, `vm.watermark_boost_factor`
- Page cache maintains separate inactive and active LRU lists — `kswapd` finds reclaimable pages quickly
- A page may be ineligible to free even if found during scan — if locked or dirty

> [!NOTE]
> `kswapd` scanning (walking LRU lists) is distinct from the original Unix page-out daemon scanning (walking all of memory).

### Process Virtual Address Space

- Range of virtual pages mapped to physical pages on demand, managed by hardware and software
- Address space divided into segments —
    - __Executable text__ — read-only, execute permission, mapped from binary text segment on filesystem
    - __Executable data__ — read/write, private flag (modifications not flushed to disk), mapped from binary data segment
    - __Heap__ — anonymous memory (no filesystem location), grows on demand via `malloc(3)`
    - __Stack__ — read/write, one per thread
- Library text segments shared across processes using the same library
    - Each process has its own private copy of the library data segment
- 64-bit x86 — AMD spec allows 48-bit address implementations, creating two canonical ranges —
    - `0x0000000000000000`–`0x00007fffffffffff` — user space
    - `0xffff800000000000`–`0xffffffffffffffff` — kernel space

### Heap Growth

- `free(3)` in simple allocators does not return memory to the OS — memory is retained for future allocations
    - Process RSS can only grow — this is normal, not necessarily a leak
- Methods to actually return memory to the OS —
    - `execve(2)` — re-exec the process, starts from empty address space
    - `mmap(2)` / `munmap(2)` — memory-mapped regions returned to OS on unmap
- glibc on Linux supports —
    - `mmap` mode for large allocations (returned to OS on free)
    - `malloc_trim(3)` — releases top-of-heap free memory via `sbrk(2)`
    - `malloc_trim(3)` called automatically by `free(3)` when free memory at top of heap exceeds `M_TRIM_THRESHOLD` (default `128 KB`)

### Allocators

- Handle memory allocation within a virtual address space
- Key properties of a good allocator —
    - Simple API — Eg - `malloc(3)`, `free(3)`
    - Efficient memory usage — coalesces fragmented unused regions for reuse
    - Performance — uses per-thread or per-CPU caches, minimises lock contention
    - Observability — exposes stats and debug modes, allocation call paths

#### Slab (kernel)

- Manages caches of fixed-size objects for fast recycling without page allocation overhead
- Effective for kernel allocations which are frequently for fixed-size structs
- Two allocation styles —
    - `kmem_alloc(size, flags)` — size-based, kernel maps to an appropriate slab cache internally
    - `kmem_cache_alloc(cache, flags)` — operates directly on a named custom slab cache
- Enhanced with per-CPU __magazine caches__ — each CPU holds `M` pre-allocated objects, reloads from a full magazine when exhausted
- Originated in Solaris 2.4, adopted by BSD (called UMA — universal memory allocator), and Linux (introduced in 2.2)

#### SLUB (kernel, Linux default)

- Linux default since kernel `2.6.23`, based on slab but simplified
- Removes per-CPU object queues and internal caches
- Delegates NUMA optimisation to the buddy page allocator instead

#### glibc (user-space)

- Based on `dlmalloc` by Doug Lea
- Allocation strategy varies by size —
    - Small — served from size-bucketed bins, coalesced via buddy-like algorithm
    - Large — tree lookup to find space efficiently
    - Very large — falls back to `mmap(2)`

#### TCMalloc (user-space)

- Thread-caching `malloc` — per-thread cache for small allocations reduces lock contention
- Periodic garbage collection migrates cached memory back to a central heap

#### jemalloc (user-space)

- Originated as FreeBSD libc allocator, available on Linux as `libjemalloc`
- Uses multiple arenas, per-thread caching, and small object slabs to improve scalability and reduce fragmentation
- Prefers `mmap(2)` over `sbrk(2)` for system memory
- Facebook extended it with profiling and additional optimisations

## Methodology

### Tools Method

- Iterates over available tools examining key metrics — may miss issues with poor tool visibility
- Key checks for Linux memory —
    - __Page scanning__ — continual scanning (>10s) indicates memory pressure
        - `sar -B`, check `pgscan` columns
    - __PSI (Pressure Stall Information)__ — `cat /proc/pressure/memory` (Linux 4.20+), shows memory saturation over time
    - __Swapping__ — `vmstat 1`, check `si` (swap-in) and `so` (swap-out) columns
    - __Free memory__ — `vmstat 1`, check `free` column
    - __OOM killer__ — search `/var/log/messages` or `dmesg(1)` for `Out of memory`
    - __Top consumers__ — `top(1)` for per-process RSS and virtual memory usage
    - __Allocation tracing__ — `perf(1)`, BCC, or `bpftrace` for stack-traced allocation profiling (high overhead)

### USE Method

- Check system-wide —
    - __Utilization__ — physical and virtual memory in use vs available
    - __Saturation__ — page scanning, paging, swapping, OOM kills
    - __Errors__ — failed allocations, OOM killer events, hardware ECC errors
- Check saturation first — continual saturation is the primary indicator of a memory issue
- Tools — `vmstat(8)`, `sar(1)` for swapping; `dmesg(1)` for OOM events; `/proc/pressure/memory` for PSI
- Physical memory reporting varies by tool — some exclude reclaimable filesystem cache, making available memory appear artificially low
- Virtual memory utilisation matters on systems without overcommit — exhaustion causes `malloc()` to fail (`ENOMEM`)
- Hardware ECC errors — `dmidecode(8)`, `edac-utils`, `ipmitool sel` — correctable ECC errors are a leading indicator of uncorrectable failures
    - Uncorrectable errors manifest as unexplained crashes, segfaults, or bus error signals

> [!NOTE]
> In cgroup/container environments, the OS instance may hit its software memory limit and begin swapping even when the host has abundant physical memory. Measure per-cgroup utilisation and saturation separately.

### Characterising Usage

- Key attributes to measure —
    - System-wide physical and virtual memory utilisation
    - Degree of saturation — swapping rate, OOM kill frequency
    - Kernel and filesystem cache memory usage
    - Per-process physical (RSS) and virtual memory usage
    - Memory resource control usage, if present
- Memory usage varies over time — cache warmup is normal growth, leaks are not
- Advanced checklist —
    - What is the WSS for each application?
    - How is kernel memory distributed across slabs?
    - What ratio of filesystem cache is active vs inactive?
    - What are the allocation call paths for processes and kernel?
    - Are any processes or kernel modules growing without bound (leak)?
    - In NUMA systems — how balanced is memory distribution across nodes?
    - What are IPC and memory stall cycle rates?
    - How balanced are the memory buses?
    - What is the ratio of local vs remote memory I/O?

### Cycle Analysis

- Memory bus load measured via CPU PMCs (performance monitoring counters)
- Key starting metric — IPC (instructions per cycle), reflects memory dependency of workload
- Low IPC → high memory stall cycles → memory-bound workload

### Performance Monitoring

- Key metrics to track over time —
    - __Utilisation__ — percent used, inferred from available memory
    - __Saturation__ — swap activity, OOM kills
- Monitor per-process memory growth over time to detect leaks

### Leak Detection

- Two root causes —
    - __Memory leak__ — memory no longer used but never freed, software bug, fixed by code change or patch
    - __Memory growth__ — memory consumed normally but at excessive rate, fixed by configuration or code change
- Memory growth is frequently misidentified as a leak — first verify whether the growth is expected (Eg - cache warmup)
- Analysis approaches —
    - Allocator debug modes — record allocation details for postmortem analysis
    - Heap dump analysis — runtime-specific (JVM, etc.)
    - `memleak(8)` (BCC) — tracks allocations not freed within an interval, with allocation call paths
        - Cannot distinguish leaks from normal growth — requires manual call path analysis
        - High overhead at high allocation rates

### Static Performance Tuning

- Configuration checklist —
    - Total main memory available
    - Per-application memory configuration limits
    - Allocators in use per application
    - Memory speed — DDR generation, is it the fastest available?
    - Has memory been fully tested — Eg - `memtester`
    - Architecture — UMA vs NUMA
    - Is OS NUMA-aware, are NUMA tunables configured?
    - Memory socket topology — same socket or split across sockets
    - Number of memory buses
    - CPU cache sizes and TLB sizes
    - BIOS memory settings
    - Huge pages — configured and in use?
    - Overcommit — enabled and configured?
    - Active memory tunables
    - Software memory limits (resource controls / cgroups)

### Micro-Benchmarking

- Used to measure main memory speed and CPU cache/cache line characteristics
- Useful when comparing systems — memory latency can dominate performance more than CPU clock speed depending on workload
- Results can reveal CPU cache sizes, cache line sizes, and NUMA latency differences

### Memory Shrinking (WSS Estimation)

- Negative experiment — progressively reduces available memory while measuring performance and swap activity
- WSS identified at the point where performance sharply degrades and swapping spikes
- Requires swap to be configured
- Not recommended for production — deliberately induces performance degradation
- Alternative — `wss(8)` tool for non-destructive WSS estimation