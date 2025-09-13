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

### Process Swapping

- Process swapping is the movement of entire processes between main memory and the physical swap device or swap file.
- To swap out a process, all of its private data must be written to the swap device, including the process heap (anonymous data), its open file table, and other metadata that is only needed when the process is active.
- Data that originated from file systems and has not been modified can be dropped and read from the original locations again when needed.
- Linux systems do not swap processes at all and rely only on paging.

### File System Cache Usage

- Memory usage rises after boot because the OS caches the filesystem using spare RAM to improve performance. The principle is - _If there is spare main memory, use it for something useful_.
- This can make free memory appear close to zero, which may worry users, but it’s not an issue since the kernel can quickly release cached memory for applications when needed.
- While calculating main memory utilization, memory used by the file system cache can be treated as unused, as it is available for reuse by applications.

### Saturation

- If demands for memory exceed the amount of main memory, main memory becomes saturated.
- The operating system may then free memory by employing paging, process swapping (if supported), and, on Linux, the OOM killer. Any of these activities is an indicator of main memory saturation.

### Allocators

- Virtual memory enables multitasking, but actual memory allocation within a virtual address space is managed by allocators—either in user space or the kernel (e.g., malloc, free).
- Allocators affect performance - they can speed up memory use through techniques like per-thread caching but may degrade performance if memory becomes fragmented or wasted.

### Shared Memory

- Memory can be shared between processes. This is commonly used for system libraries to save memory by sharing one copy of their read-only instruction text with all processes that use it- this presents difficulties for observability tools that show per-process main memory usage.
- One technique in use by Linux is to provide an additional measure, the proportional set size (PSS), which includes private memory (not shared) plus shared memory divided by the number of users.

### Working Set Size

- Working set size (WSS) is the amount of main memory a process frequently uses to perform work. 
- Performance should greatly improve if the WSS can fit into the CPU caches, rather than main memory. 
- Also, performance will greatly degrade if the WSS exceeds the main memory size, and the application must swap to perform work.

### Word Size

- A processor’s “word size” (like 32-bit or 64-bit) determines how much memory it can directly address.
    - 32-bit - can handle up to 4 GB of memory in a single program.
    - 64-bit - can handle way more (practically unlimited for most apps).
- 32-bit CPUs have a trick called Physical Address Extension (PAE) feature to use more memory, but each program still can’t use more than 4 GB.
- Part of the address space is reserved for the kernel—e.g., on 32-bit Windows, 2 GB is reserved, leaving 2 GB for applications; Linux or Windows with the `/3GB` option reserves 1 GB.
- With a 64-bit word size (if the processor supports it) the address space is so much larger that the kernel reservation should not be an issue.
- Larger word sizes can improve memory performance by allowing instructions to process more data at once, though a small amount of memory may be wasted, in cases where a data type has unused bits at the larger bit width.

## Memory Architecture

### Hardware

**Main Memory** -

- Dynamic random-access memory (DRAM) - common type of main memory used today.
- Volatile memory — its contents are lost when power is lost.
- DRAM provides high-density storage, as each bit is implemented using only two logical components - a capacitor and a transistor.
- The capacitor requires a periodic refresh to maintain charge.
- Enterprise servers - typically 1 GB – 1 TB+ of DRAM.
- Cloud instances - usually 512 MB – 256 GB each.
- Cloud computing can scale memory by pooling many instances, but this leads to higher coherency cost.




