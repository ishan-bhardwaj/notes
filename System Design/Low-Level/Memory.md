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
        - Normal behavior for apps using file memory mappings (`mmap()`) and file systems using page cache
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