# Memory

- **Main Memory / Physical Memory** - Fast data storage, typically implemented with DRAM.
- **Virtual memory** - Abstraction of main memory that makes memory appear infinite and non-shared.
- **Resident memory** - Memory currently in main memory.
- **Anonymous memory** - Memory without a file system location, including process heap.
- **Address space** - A memory context; each process and the kernel have virtual address spaces.
- **Segment** - Virtual memory area with a specific purpose (e.g., executable or writable pages).
- **Instruction Text** - CPU instructions in memory, usually in a segment.
- **OOM (Out of memory)** - When the kernel detects low available memory.
- **Page** - Memory unit as used by OS and CPUs, typically 4–8 KB; modern CPUs support multiple & larger sizes.
- **Page fault** - An invalid memory access. These are normal occurrences when using on-demand virtual memory.
- **Paging** - The transfer of pages between main memory and the storage devices.
- **Swapping** - Linux term for anonymous paging to swap device (i.e. transfer of swap pages). Unix historically refers to the transfer of entire processes between main memory and the swap devices.
- **Swap** - An on-disk area for paged anonymous data. It may be an area on a storage device, also called a physical swap device, or a file system file, called a swap file.

## Concepts

### Virtual Memory

- Provides each process and the kernel a large, linear, private address space. The process address space is mapped by the virtual memory subsystem to main memory and the physical swap device.
- Simplifies software development by letting the OS manage physical memory.
- Supports multitasking - Processes have separate address spaces.
- Supports oversubscription - In-use memory can exceed main memory.
- Virtual memory is page-based. Pages of memory can be moved between them by the kernel as needed, a process Linux calls _swapping_ (and other OSes call _anonymous paging_). This allows the kernel to oversubscribe main memory.
- Kernel limits - Oversubscription may be capped at physical memory + swap; exceeding this causes “out of virtual memory” errors.
- Overcommit - Linux can allow no bounds on allocations, relying on paging and demand paging.

### Paging

- Movement of pages between main memory and storage -
    - Page-ins - Pages loaded into memory.
    - Page-outs - Pages written back to storage.
- Benefits -
    - Allows partially loaded programs to execute.
    - Allows programs larger than main memory to execute.
    - Enables efficient movement between main memory and storage.
- Two types of paging - file system paging and anonymous paging.

**File System Paging**

- Caused by the reading and writing of pages in _memory-mapped files_.

> [!TIP]
> A memory-mapped file is a file whose contents are mapped directly into a process’s virtual address space, allowing the process to access the file data as if it were part of its own memory. Accessing the file via memory reads and writes is automatically managed by the operating system, which handles loading pages into RAM and writing modified pages back to the storage device.

- This is normal behavior for applications that use file memory mappings (`mmap(2)`) and on file systems that use the page cache.
- Often called “good” paging.
- Kernel frees memory by paging pages out -
    - Dirty page - modified in memory → must be written to disk.
    - Clean page - Not modified → can be freed immediately; disk already has a copy.

**Anonymous Paging (Swapping)**

- Handles memory private to a process, such as the heap and stack, which is not backed by any file.
- Called anonymous because it has no file system path.
- Anonymous page-outs move this memory to swap devices or swap files.
- Considered “bad” paging because accessing paged-out memory blocks the application on disk I/O.
- Anonymous page-in - Reading a swapped-out page into RAM introduces _synchronous_ latency.
- Anonymous page-out - Can be done _asynchronously_, usually without directly affecting performance.
- Fast swap devices (e.g., 3D XPoint) reduce latency, making swapping a viable way to extend memory.
- Best Practices -
    - Avoid anonymous paging when possible for optimal performance.
    - Keep applications within available RAM.
    - Monitor page scanning, memory utilization, and anonymous paging to prevent memory shortages.

### Demand Paging

- Most OSes map pages to physical memory on demand, deferring CPU overhead until pages are accessed rather than at allocation.
- Process Flow Example -
    - Step 1 - `malloc()` allocates memory.
    - Step 2 - Store instruction accesses the memory.
    - Step 3 - MMU performs virtual-to-physical lookup for the page.
    - Step 4 - Page fault occurs because mapping does not exist.
    - Step 5 - Kernel creates on-demand mapping.
    - Step 6 - Page may later be paged out to swap if memory pressure occurs.
- Types of page faults -
    - Minor faults - 
        - When the mapping can be satisfied from another page in memory.
        - This may occur when -
            - mapping a new page from available memory, during memory growth of the process.
            - mapping to another existing page, such as reading a page from a mapped shared library.
    - Major faults -
        - Page faults that require storage device access such as accessing an uncached memory-mapped file.
- Page States in Virtual Memory can be in any one of the following states -
    - Unallocated
    - Allocated but unmapped (unpopulated, not yet faulted) - causes page faults.
    - Allocated and mapped to RAM - page fault triggers mapping.
    - Allocated and mapped to swap (paged out due to memory pressure).
- Memory Usage Terms -
    - Resident Set Size (RSS) - Size of pages in main memory (RAM).
    - Virtual Memory Size - Size of all allocated pages.

### Overcommit

- Allows allocation of more memory than physically available (RAM + swap).
- Relies on demand paging and the fact that applications often use only part of their allocated memory.
- Memory requests (e.g., `malloc()`) succeed even if total allocation exceeds limits - an application programmer can allocate memory generously and later use it sparsely on demand.
- Overcommit behavior can be tuned via a kernel parameter.
- Impact - Consequences depend on how the kernel handles memory pressure.

