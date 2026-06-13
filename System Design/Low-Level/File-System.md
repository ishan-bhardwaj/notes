## File System Performance

## Terminology

| Term | Definition |
|------|------------|
| File system | Organisation of data as files and directories with file-based interface and permissions |
| File system cache | Area of main memory (DRAM) used to cache file system contents |
| Operations | File system requests — `read(2)`, `write(2)`, `open(2)`, `close(2)`, `stat(2)`, `mkdir(2)` |
| I/O | Operations that directly read and write — `read(2)`, `write(2)`, `stat(2)`, `mkdir(2)` — excludes `open(2)` and `close(2)` |
| Logical I/O | I/O issued by application to the file system |
| Physical I/O | I/O issued directly to disks by the file system |
| Block size (record size) | Size of file system on-disk data groups |
| Throughput | Current data transfer rate between applications and file system (bytes/s) |
| inode | Index node — data structure containing file system object metadata — permissions, timestamps, data pointers |
| VFS | Virtual file system — kernel interface to abstract and support different file system types |
| Volume | Instance of storage — may be a portion of a device or multiple devices |
| Volume manager | Software for managing physical storage devices to create virtual volumes |

## Models

### File System Interfaces

- File system exposes object operations to applications — logical I/O occurs here
- Physical I/O occurs at the disk device layer
- Linux provides additional syscall variants — `readv(2)`, `writev(2)`, `openat(2)`, etc.
- Performance analysis approach — treat as black box, measure operation latency

### File System Cache

- Stored in main memory (DRAM)
- Read — returns data from cache (hit) or disk (miss) — misses populate cache
- Write — buffered in cache, flushed to disk later (write-back caching)
- Cache bypass available via raw I/O or direct I/O (`O_DIRECT`)

### Second-Level Cache

- Any memory type beyond main memory — typically flash (SSDs)
- Eg - ZFS L2ARC — first developed for ZFS in 2007

## Concepts

### File System Latency

- Primary metric — time from logical file system request to completion
- Includes — file system time, disk I/O subsystem time, disk device wait time
- Application threads often block during file system requests — latency directly and proportionally affects application performance
- Non-blocking cases — non-blocking I/O, prefetch, async background flush threads
- OS historically focused on disk device statistics — misleading as background flushes appear as high-latency disk I/O but no application is waiting

### Caching

- File system uses main memory as transparent cache — logical I/O latency served from RAM
- Free memory post-boot shrinks as cache grows — normal behaviour — kernel reclaims quickly when needed
- Multiple cache types in use simultaneously —

| Cache | Example |
|-------|---------|
| Page cache | OS page cache |
| File system primary cache | ZFS ARC |
| File system secondary cache | ZFS L2ARC |
| Directory cache | dentry cache |
| inode cache | inode cache |
| Device cache | ZFS vdev |
| Block device cache | Buffer cache |

### Random vs Sequential I/O

- Sequential — each I/O offset begins at end of previous I/O
- Random — no apparent relationship between offsets
- Fragmentation — poor file placement causing sequential logical I/O to become random physical I/O
- File systems measure access patterns — identify sequential workloads — apply prefetch/read-ahead

### Prefetch

- Detects sequential read workload from current and previous file I/O offsets
- Issues disk reads before application requests them — populates cache ahead of time
- Sequence —
    - Application issues `read(2)` — data not cached — file system issues disk read
    - Previous offset compared to current — if sequential — additional reads issued (prefetch)
    - First read completes — returns to application
    - Prefetch reads complete — cache populated for future reads
- Poor prefetch detection — issues unnecessary I/O — pollutes cache — wastes disk bandwidth
- Tunable per file system type

### Write-Back Caching

- Treats writes as completed after transfer to main memory — writes to disk later asynchronously
- Flush — kernel process writing dirty data to disk
- Trade-off — performance vs reliability — dirty data lost on power failure — incomplete writes can corrupt on-disk state
- File systems offer both write-back (default) and synchronous write options

### Synchronous Writes

- Completed only when fully written to persistent storage — includes all necessary metadata changes
- Much slower than async writes — incur disk device I/O latency
- Used by — database log writers, applications where data corruption is unacceptable
- Two forms —
    - Individual synchronous I/O — `O_SYNC`, `O_DSYNC`, `O_RSYNC` flags on `open(2)`
    - Synchronous commit of previous async writes — `fsync(2)` at checkpoints

### Raw and Direct I/O

- __Raw I/O__ — issued directly to disk offsets — bypasses file system entirely — used by some databases — harder to administer (no standard toolset)
- __Direct I/O__ — uses file system but bypasses file system cache — `O_DIRECT` flag on `open(2)` — I/O may be resized to match file system record size or fail with `EINVAL` — may disable prefetch

### Non-Blocking I/O

- Default — file system I/O either completes immediately (cache) or blocks waiting (disk)
- Non-blocking — `O_NONBLOCK` or `O_NDELAY` flags — returns `EAGAIN` instead of blocking
- Async I/O interfaces — `aio_read(3)`, `aio_write(3)`
- `io_uring` — Linux 5.1 — new async I/O interface — improved ease of use, efficiency, performance

### Memory-Mapped Files

- Files mapped to process address space via `mmap(2)` — removed via `munmap(2)`
- Avoids syscall execution and context switch overhead of `read(2)`/`write(2)`
- May avoid double copying if kernel maps file data buffer directly to process address space
- Disadvantage on multiprocessor — TLB shootdown overhead to keep CPU MMUs in sync
- Tunable via `madvise(2)`

> [!NOTE]
> Using `mmap(2)` to solve file system performance issues without analysis first often accomplishes little if the root cause is disk I/O latency — the syscall overhead savings are dwarfed by disk latency.

### Metadata

- __Logical metadata__ — file system information readable via POSIX interface —
    - Explicit — `stat(2)`, `creat(2)`, `unlink(2)`, `mkdir(2)`, `chown(2)`, `chmod(2)`
    - Implicit — access timestamp updates, directory modification timestamps, free block bitmaps
- __Physical metadata__ — on-disk layout metadata — superblocks, inodes, block pointer arrays, free lists
- Metadata-heavy workload — Eg - web servers calling `stat(2)` at high rates to check file modification

### Logical vs Physical I/O

- Logical I/O (application to file system) may not match physical I/O (file system to disk) —

| Category | Cause |
|----------|-------|
| Unrelated | Other applications, other cloud tenants, kernel RAID rebuild, admin backups |
| Indirect | Prefetch adding I/O, write-back buffering deferring writes (may buffer for tens of seconds) |
| Implicit | mmap load/store triggering disk I/O — not via syscalls |
| Deflated | File system cache hits, write cancellation, compression, coalescing, in-memory file systems (tmpfs) |
| Inflated | File system metadata writes, record size rounding, journaling (double writes), RAID parity, mirror writes |

- Example — 1-byte write to existing file can cause — 128 KB read (record load), 128 KB write (record flush), additional metadata writes — multiple disk I/Os for one application byte

### Operations Are Not Equal

- Cannot determine workload performance from operation rate alone
- Determinant factors — cache hit vs miss, random vs sequential, read vs write, sync vs async, I/O size, CPU load, storage device characteristics

### Special File Systems

- Linux uses special file systems for — `/tmp` (tmpfs), `/dev` (devtmpfs), `/proc` (procfs), `/sys` (sysfs)
- These never or rarely access persistent storage
- List non-storage file systems via `grep '^nodev' /proc/filesystems`

### Access Timestamps

- Many file systems record time each file and directory was accessed (read)
- Creates write workload on every read — consumes disk I/O resources
- Disable via `noatime` mount option — or use `relatime` (Linux default since 2.6.30)

### Capacity

- File systems approaching full — performance may degrade —
    - More CPU time and disk I/O to find free blocks
    - Free space smaller and more sparse — smaller or random I/O
- ZFS switches to slower free-block-finding algorithm above ~80% usage

## Architecture

### File System I/O Stack

- Application path — syscalls via VFS — file system — disk device subsystem
- Raw I/O path — syscalls directly to disk device subsystem (bypasses VFS and file system)
- Direct I/O path — via VFS and file system — skips file system cache

### VFS

- Abstracts all file system types behind a common interface — originated in SunOS
- Linux VFS uses terms inode and superblock for in-memory VFS objects — prefixed names (Eg - `ext4_inode`) for on-disk structures
- Common instrumentation point — measure performance of any file system type at VFS layer

### File System Caches

#### Buffer Cache

- Original Unix cache — cached disk device blocks at block device interface
- Linux 2.4+ — buffer cache stored inside page cache (unified buffer cache) — avoids double caching and synchronisation overhead
- Buffer cache functionality still exists — improves block device I/O performance
- Dynamic size — observable from `/proc`

#### Page Cache

- Caches virtual memory pages including memory-mapped file system pages
- More efficient than buffer cache — file offset lookup avoids disk offset translation
- Dynamic size — grows to use available memory, freed when applications need it
- Dirty pages flushed by per-device flusher threads (named `flush`) — replaced `pdflush` pool in Linux 2.6.32
- Flush triggers —
    - After 30-second interval
    - `sync(2)`, `fsync(2)`, `msync(2)` syscalls
    - Too many dirty pages (`dirty_ratio`, `dirty_bytes` tunables)
    - No available pages in cache
- `kswapd` (page-out daemon) may also flush dirty pages to free memory under pressure

#### Dentry Cache (Dcache)

- Remembers mappings from directory entry (`struct dentry`) to VFS inode
- Improves path name lookup performance — direct inode mapping instead of walking directory contents
- Stored in hash table — hashed by parent dentry and directory entry name
- RCU-walk algorithm — walks path without updating dentry reference counts — improves scalability on multi-CPU
- Falls back to ref-walk if dentry not in cache
- Negative caching — remembers lookups for non-existent entries — improves failed lookup performance (Eg - shared library searches)
- Dynamic size — LRU eviction under memory pressure

#### Inode Cache

- Contains VFS inodes (`struct inode`) — file system object properties returned by `stat(2)`
- Stored in hash table — hashed by inode number and file system superblock
- Most lookups via dentry cache
- Dynamic size — holds at least all inodes mapped by Dcache
- Size visible via `/proc/sys/fs/inode*`

### File System Features

#### Block vs Extent

- __Block-based__ — fixed-size blocks referenced by metadata pointers — large files require many pointers — scattered blocks cause random I/O
- __Extent-based__ — preallocates contiguous space (extents) — variable length — improves streaming and random I/O performance — fewer metadata objects to track

#### Journaling

- Records changes to file system — allows atomic replay on crash — prevents corruption
- Without journaling — crash recovery requires full file system walk (hours for TB file systems)
- Journal written synchronously — can use separate device
- Journal modes —
    - Ordered mode — metadata only
    - Journal mode — metadata and data (double I/O overhead)
- Log-structured file system — entire file system is a journal — all writes sequential — optimal write performance

#### Copy-on-Write (COW)

- Does not overwrite existing blocks — writes to new location, updates references, adds old blocks to free list
- Improves integrity on failure — turns random writes into sequential writes
- Near-capacity performance — fragmentation may occur — defragmentation may help

#### Scrubbing

- Asynchronously reads all data blocks and verifies checksums
- Detects failed drives early — ideally while still recoverable via RAID
- Run at low priority or during low workload periods — scrub I/O can hurt performance

### File System Types

#### ext3

- Key performance features —
    - Journaling — ordered mode (metadata only) or journal mode (metadata + data)
    - External journal device — separate journal from data workload
    - Orlov block allocator — spreads top-level directories across cylinder groups — co-locates related data
    - Directory indexes — hashed B-trees for faster directory lookups

#### ext4 (2008)

- Key performance features —
    - Extents — contiguous placement — reduces random I/O — larger sequential I/O sizes
    - Preallocation via `fallocate(2)` — preallocate likely-contiguous space
    - Delayed allocation — block allocation deferred until flush — groups writes via multiblock allocator — reduces fragmentation
    - Faster fsck — unallocated blocks and inode entries marked — skipped during check

```bash
ls /sys/fs/ext4/features    # view supported features
```

#### XFS (1993, Linux early 2000s)

- Key performance features —
    - Allocation Groups (AGs) — partition divided into equal-sized AGs — parallel access — independent metadata per AG
    - Extents — contiguous placement
    - Journaling — with external journal device option
    - Striped allocation — optimise for underlying RAID/LVM stripe geometry
    - Delayed allocation — extent allocation deferred until flush — reduces fragmentation
    - Online defragmentation — `xfs_fsr` utility — operates on mounted file system

```bash
cat /proc/fs/xfs/stat    # internal XFS performance data
```

#### ZFS (Sun Microsystems, 2005)

- Key performance features —

| Feature | Description |
|---------|-------------|
| Pooled storage | All devices in pool — parallel access — RAID 0/1/10/Z/Z2/Z3 |
| COW | Copies modified blocks — groups and writes sequentially |
| ARC | Adaptive Replacement Cache — MRU + MFU algorithms simultaneously — ghost lists track performance of each |
| Intelligent prefetch | Separate prefetch for metadata, znodes, vdevs |
| Multiple prefetch streams | Tracks individual streaming readers — avoids random I/O between streams |
| ZIO pipeline | Device I/O processed by staged pipeline with thread pools |
| Compression | lzjb lightweight — may improve performance by reducing I/O load |
| SLOG (Separate Intent Log) | Synchronous writes to separate device — avoids pool disk contention — replayed on failure |
| L2ARC | Second-level cache on flash SSDs — random read cache — clean data only |
| Data deduplication | Hash table-based — only performant when hash table fits in main memory |
| Snapshots | Near-instantaneous due to COW — defers block copying until needed |
| Logging (TXGs) | Transaction groups flushed as atomic batches — always consistent on-disk format |

> [!NOTE]
> ZFS issues cache flush commands to storage devices by default for integrity guarantees — induces latency for operations that must wait for the flush.

#### btrfs (Oracle, 2007)

- COW B-tree based — similar architecture to ZFS
- Features — pooled storage (RAID 0/1/10), COW, extents, online balancing, snapshots, compression (zlib, LZO), per-subvolume journaling
- Planned — RAID-5/6, object-level RAID, data deduplication

### Volumes and Pools

- __Volumes__ — multiple disks presented as one virtual disk — file system built on top — isolates workloads
- __Pooled storage__ — multiple disks in a pool — multiple file systems created from pool — more flexible than volumes — all devices usable by all file systems

| Consideration | Detail |
|--------------|--------|
| Stripe width | Match to workload |
| Observability | Virtual device utilisation can be misleading — check physical devices separately |
| CPU overhead | RAID parity computation — less of an issue with modern CPUs |
| Rebuilding (resilvering) | Populating empty disk added to RAID group — significant I/O impact — may last hours or days |

> [!NOTE]
> Offline rebuilds of unmounted drives can improve rebuild times. Storage device capacity grows faster than throughput — rebuild times worsen over time — risk of secondary failure during rebuild increases.

## Methodology

### Latency Analysis

- Measure latency of all file system operations — including non-I/O operations (Eg - `sync(2)`)
- `operation latency = time(completion) - time(request)`
- Four layers for measurement —

| Layer | Pros | Cons |
|-------|------|------|
| Application | Closest measure of impact on application — includes application context | Varies between applications and versions |
| Syscall interface | Well-documented — observable via OS tools and static tracing | Catches all file system types including non-storage — multiple syscall variants per function |
| VFS | Standard interface — one call per operation | Includes non-storage file systems — may need filtering |
| Top of file system | Target file system only — internal context available | File system-specific — tracing varies between versions |

- Latency presentation — per-interval averages, full distributions (histograms/heat maps), per-operation details
- High cache hit ratio (>99%) — averages dominated by cache hit latency — use distributions to find outliers

#### Transaction Cost

- `percent time in file system = 100 × total blocking file system latency / application transaction time`
- Quantifies file system cost in terms of application performance
- Eg - 180 ms of 200 ms transaction in file system = 90% — eliminating FS latency could improve performance up to 10x
- Eg - 2 ms of 200 ms transaction in file system = 1% — file system is not the bottleneck — investigate elsewhere

### Workload Characterization

- Basic attributes —
    - Operation rate and operation types
    - File I/O throughput
    - File I/O size
    - Read/write ratio
    - Synchronous write ratio
    - Random vs sequential file offset access

#### Advanced Checklist

- File system cache hit ratio and miss rate
- Cache capacity and current usage
- Other cache statistics — dentry, inode, buffer
- Applications and users using the file system
- Files and directories being accessed, created, deleted
- Errors encountered — invalid requests or file system issues
- Why file system I/O is issued — user-level call path
- Degree to which application directly (synchronously) requests I/O
- Distribution of I/O arrival times

### Performance Monitoring

- Key metrics —
    - Operation rate — most basic workload characteristic
    - Operation latency — resulting performance
- Monitor per operation type — reads, writes, stats, opens, closes
- Full distribution preferred (histogram, heat map) over averages for outlier detection
- Linux lacks readily available per-file-system operation statistics — exception: NFS via `nfsstat(8)`

### Static Performance Tuning

- Configuration checklist —
    - Number of file systems mounted and active
    - File system record size
    - Access timestamps enabled
    - Other enabled features — compression, encryption
    - File system cache maximum size configuration
    - Other cache configurations — directory, inode, buffer
    - Second-level cache presence and usage
    - Number and configuration of storage devices — RAID type
    - File system types in use
    - File system or kernel version — known bugs/patches
    - Resource controls in use for file system I/O

### Cache Tuning

- Examine all caches — buffer cache, directory cache, inode cache, page cache
- Check hit rates — tune workload for cache and cache for workload
- If swap configured — `swappiness` tunable controls balance between evicting page cache vs swapping

### Workload Separation

- Separate workloads to exclusive file systems and disk devices — "separate spindles"
- Especially important for rotational disks — avoid random I/O between two workload locations
- Eg - database log files and database data files on separate devices

### Micro-Benchmarking

- Factors to test —
    - Operation types — read, write, other operations
    - I/O size — 1 byte to 1 MB+
    - File offset pattern — random or sequential
    - Random-access pattern — uniform, random, or Pareto distribution
    - Write type — async or synchronous (`O_SYNC`)
    - Working set size (WSS) — critical factor
    - Concurrency — parallel I/O count or thread count
    - Memory mapping — `mmap(2)` vs `read(2)`/`write(2)`
    - Cache state — cold (unpopulated) vs warm
    - File system tunables — compression, deduplication

#### WSS Expectations

| System Memory | Total File Size | Benchmark | Expectation |
|---------------|----------------|-----------|-------------|
| 128 GB | 10 GB | Random read | 100% cache hits |
| 128 GB | 10 GB | Random read, direct I/O | 100% disk reads |
| 128 GB | 1000 GB | Random read | Mostly disk reads, ~12% cache hits |
| 128 GB | 10 GB | Sequential read | 100% cache hits |
| 128 GB | 1000 GB | Sequential read | Mix of prefetch cache hits and disk reads |
| 128 GB | 10 GB | Buffered writes | Mostly cache (buffering), some blocking |
| 128 GB | 10 GB | Synchronous writes | 100% disk writes |

## Observability Tools

## mount

```bash
mount    # list mounted file systems with type and mount flags
```

- Key mount flags for performance — `relatime` (default since Linux 2.6.30), `noatime`, `discard`
- `relatime` — only updates access time when modify/change times also updated or last update >1 day old

## free

```bash
free -m     # memory and swap in MB
free -mw    # wide mode — separate buffers and cache columns
```

| Column | Description |
|--------|-------------|
| `buffers` | Buffer cache size |
| `cache` | Page cache size |
| `buff/cache` | Combined (default output) |
| `available` | Memory available for applications without swapping — accounts for non-reclaimable memory |

## top

- Summary header includes `buff/cache` and `avail Mem` — same as `free(1)`

## vmstat

```bash
vmstat 1    # 1-second interval
```

| Column | Description |
|--------|-------------|
| `buff` | Buffer cache size (KB) |
| `cache` | Page cache size (KB) |

## sar

```bash
sar -v 1    # file system cache and inode statistics
sar -r 1    # includes kbbuffers and kbcached columns
```

| Column | Description |
|--------|-------------|
| `dentunusd` | Directory entry cache unused count |
| `file-nr` | File handles in use |
| `inode-nr` | Inodes in use |
| `kbbuffers` | Buffer cache size (KB) |
| `kbcached` | Page cache size (KB) |

## slabtop

```bash
slabtop -o    # once (not refreshing) — sort by cache size
```

### File System-Related Slab Caches

| Cache Name | Description |
|------------|-------------|
| `buffer_head` | Buffer cache |
| `dentry` | Dentry cache |
| `inode_cache` | Generic inode cache |
| `ext3_inode_cache` | ext3 inode cache |
| `ext4_inode_cache` | ext4 inode cache |
| `xfs_inode` | XFS inode cache |
| `btrfs_inode` | btrfs inode cache |

## strace

```bash
strace -ttT -p <PID>    # trace syscalls with timestamps and per-call timing
```

- `-tt` — relative timestamps, `-T` — syscall duration on right
- Current ptrace-based implementation has severe performance overhead — use only when other methods unavailable
- Observer effect — measured latency skewed by strace overhead

> [!NOTE]
> Prefer `ext4slower(8)` or BPF-based tools — use per-CPU buffered tracing — orders of magnitude lower overhead than `strace`.

## fatrace

```bash
fatrace                   # trace all file access events system-wide
fatrace -t O              # filter to open events only
```

- Uses Linux `fanotify` API — traces opens (O), reads (R), writes (W), closes (C)
- Output format — `process(PID): event_type /full/path`
- Useful for workload characterisation — reveals files accessed and unnecessary work
- Busy workloads — tens of thousands of lines/second — significant CPU overhead
- Prefer BPF tools for lower overhead

## LatencyTOP

- Reports sources of latency aggregated system-wide and per process — includes file system latency
- Requires non-default kernel options — `CONFIG_LATENCYTOP`, `CONFIG_HAVE_LATENCYTOP_SUPPORT`
- Not actively maintained — prefer BPF tools (`ext4dist`, `ext4slower`)

## opensnoop

- BCC and bpftrace tool — traces file opens via `open(2)` and `openat(2)` syscalls

```bash
opensnoop -T        # include timestamp column
opensnoop -x        # only failed opens (troubleshoot missing files)
opensnoop -p PID    # trace single process
opensnoop -n NAME   # filter by process name
```

| Column | Description |
|--------|-------------|
| `TIME(s)` | Timestamp |
| `PID` | Process ID |
| `COMM` | Process name |
| `FD` | File descriptor returned |
| `ERR` | Error code (0 = success) |
| `PATH` | File path |

- Negligible overhead — file opens are infrequent
- Use cases — discover config file locations, log file locations, data file paths, troubleshoot missing files, identify excessive open rates

## filetop

- BCC tool — `top` for files — shows most frequently read/written filenames

```bash
filetop           # top 20 files by read bytes, 1-second refresh
filetop -a        # all file types including sockets and device nodes
filetop -C        # rolling output (no screen clear) — preserves scrollback history
filetop -r ROWS   # number of rows to display
filetop -p PID    # single process
```

| Column | Description |
|--------|-------------|
| `TID` | Thread ID |
| `COMM` | Process name |
| `READS`, `WRITES` | Read/write count during interval |
| `R_Kb`, `W_Kb` | Read/write throughput (KB) |
| `T` | File type — R=regular, O=other, S=socket |
| `FILE` | Filename |

## cachestat

- BCC tool — page cache hit and miss statistics

```bash
cachestat -T 1    # 1-second interval with timestamps
```

| Column | Description |
|--------|-------------|
| `HITS` | Page cache hits |
| `MISSES` | Page cache misses |
| `DIRTIES` | Pages dirtied (writes) |
| `HITRATIO` | Hit percentage |
| `BUFFERS_MB` | Buffer cache size |
| `CACHED_MB` | Page cache size |

- Low hit ratio — consider reducing application memory footprint to leave more room for page cache
- Experimental tool using kprobes — may need maintenance across kernel versions

## ext4dist (xfs, zfs, btrfs, nfs)

- BCC and bpftrace tool — latency distribution histograms for common file system operations
- Operations traced — reads, writes, opens, fsync
- Variants — `xfsdist(8)`, `zfsdist(8)`, `btrfsdist(8)`, `nfsdist(8)`

```bash
ext4dist 10 1     # trace for 10 seconds, print once
ext4dist -m       # output in milliseconds
ext4dist -p PID   # single process
```

- Bi-modal read distribution — one mode 0–15 μs (cache hits), another 256–2048 μs (disk reads)
- Fast writes with slow fsync — write-back buffering visible as pattern
- Output suitable for heat map visualisation

## ext4slower (xfs, zfs, btrfs, nfs)

- BCC tool — per-event details for operations slower than a threshold
- Operations traced — reads, writes, opens, fsync
- Default threshold — 10 ms

```bash
ext4slower              # operations slower than 10 ms
ext4slower 0            # all operations (short duration only — high output volume)
ext4slower -p PID       # single process
ext4slower -j           # parsable CSV output
```

| Column | Description |
|--------|-------------|
| `TIME` | Timestamp |
| `COMM` | Process name |
| `PID` | Process ID |
| `T` | Type — R=read, W=write, O=open, S=sync |
| `BYTES` | I/O size |
| `OFF_KB` | Offset in KB |
| `LAT(ms)` | Operation latency (ms) |
| `FILENAME` | File name |

> [!TIP]
> Use `ext4dist` first for distribution overview, then `ext4slower` to identify specific slow events and their context. These show logical I/O latency — what applications experience — not physical disk latency.

## bpftrace

### One-Liners

```bash
# Trace files opened via openat(2) with process name
bpftrace -e 't:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# Count read syscalls by type
bpftrace -e 'tracepoint:syscalls:sys_enter_*read* { @[probe] = count(); }'

# Count write syscalls by type
bpftrace -e 'tracepoint:syscalls:sys_enter_*write* { @[probe] = count(); }'

# Distribution of read() request sizes
bpftrace -e 'tracepoint:syscalls:sys_enter_read { @ = hist(args->count); }'

# Distribution of read() return bytes (including errors)
bpftrace -e 'tracepoint:syscalls:sys_exit_read { @ = hist(args->ret); }'

# Count read() errors by error code
bpftrace -e 't:syscalls:sys_exit_read /args->ret < 0/ { @[-args->ret] = count(); }'

# Count VFS calls
bpftrace -e 'kprobe:vfs_* { @[probe] = count(); }'

# Count VFS calls for PID 181
bpftrace -e 'kprobe:vfs_* /pid == 181/ { @[probe] = count(); }'

# Count ext4 tracepoints
bpftrace -e 'tracepoint:ext4:* { @[probe] = count(); }'

# Count xfs tracepoints
bpftrace -e 'tracepoint:xfs:* { @[probe] = count(); }'

# Count ext4 file reads by process and user stack
bpftrace -e 'kprobe:ext4_file_read_iter { @[ustack, comm] = count(); }'

# Count dcache references by process and PID
bpftrace -e 'kprobe:lookup_fast { @[comm, pid] = count(); }'

# Trace ZFS spa_sync() times
bpftrace -e 'kprobe:spa_sync { time("%H:%M:%S ZFS spa_sync()\n"); }'
```

### Syscall Tracing Notes

- `openat(2)` — excellent tracing target — `filename` argument directly available
- `read(2)` — problematic — FD argument does not differentiate file system from socket or `/proc`
- Solutions for `read(2)` ambiguity —
    - Print PID and FD — look up later via `lsof(8)` or `/proc`
    - Trace from VFS — more data structures available
    - Trace file system functions directly — excludes other I/O types (approach used by `ext4dist`/`ext4slower`)

### VFS Latency Tracing (vfsreadlat.bt)

```bpftrace
#include <linux/fs.h>

kprobe:vfs_read
{
    @file[tid] = ((struct file *)arg0)->f_inode->i_sb->s_type->name;
    @ts[tid] = nsecs;
}

kretprobe:vfs_read
/@ts[tid]/
{
    @us[str(@file[tid])] = hist((nsecs - @ts[tid]) / 1000);
    delete(@file[tid]); delete(@ts[tid]);
}
```

- Breaks down `vfs_read()` latency by file system type — `ext4`, `sockfs`, `proc`, `tmpfs`, etc.
- Include `ustack` as additional histogram key to determine if latency occurs during application request or async background task

### File System Tracepoints

```bash
# List ext4 tracepoints (105 on kernel 5.3)
bpftrace -l 'tracepoint:ext4:*'

# List ext4 kprobe targets (538 on kernel 5.3)
bpftrace -lv 'kprobe:ext4_*'

# List tracepoint arguments
bpftrace -lv t:syscalls:sys_enter_openat
```

## Other Tools

| Tool | Description |
|------|-------------|
| `syscount` | Count syscalls including file system-related |
| `statsnoop` | Trace calls to `stat(2)` varieties |
| `syncsnoop` | Trace `sync(2)` and variants with timestamps |
| `mmapfiles` | Count `mmap(2)` files |
| `vfsstat` | Common VFS operation statistics |
| `vfscount` | Count all VFS operations |
| `vfssize` | VFS read/write sizes |
| `fsrwstat` | VFS reads/writes by file system type |
| `fileslower` | Slow file reads/writes |
| `writeback` | Write-back events and latencies |
| `dcstat` | Directory cache hit statistics |
| `dcsnoop` | Trace directory cache lookups |
| `icstat` | Inode cache hit statistics |
| `readahead` | Read-ahead hits and efficiency |
| `df(1)` | File system usage and capacity |
| `inotify` | Linux framework for monitoring file system events |

### ZFS arcstat

```bash
arcstat 1    # ARC and L2ARC size, hit/miss rates per second
```

| Column | Description |
|--------|-------------|
| `read` | Total ARC accesses |
| `miss` | Total ARC misses |
| `miss%` | Miss percentage |
| `dmis`, `pmis`, `mmis` | Demand, prefetch, metadata misses |
| `arcsz` | Current ARC size |
| `c` | ARC target size |

## Visualisations

### Line Graphs

- Plot reads, writes, other operations separately over time — identifies time-based patterns
- Average latency alone is misleading — hides bimodal distribution

### Latency Heat Maps

- x-axis — time, y-axis — I/O latency, z-axis — count (colour)
- Expected bimodal distribution — low latency (cache hits) and high latency (disk I/O)
- Eg - ZFS L2ARC heat map — left half (no L2ARC) shows gap between cache hits and disk latency — right half (L2ARC) fills gap with intermediate flash latency

## Experimentation

### Ad Hoc (dd)

```bash
# Write 1 GB file with 1 MB I/O size
dd if=/dev/zero of=file1 bs=1024k count=1k

# Read 1 GB file with 1 MB I/O size
dd if=file1 of=/dev/null bs=1024k
```

### fio (recommended)

```bash
fio --runtime=60 --time_based --name=randread \
    --numjobs=1 --rw=randread \
    --random_distribution=pareto:0.9 \
    --bs=8k --size=5g --filename=fio.tmp
```

- Key advantages —
    - Non-uniform random distributions (Pareto) — more realistic real-world simulation
    - Latency percentiles — 99.00, 99.50, 99.90, 99.95, 99.99

### Cache Flushing

```bash
echo 1 > /proc/sys/vm/drop_caches    # free page cache
echo 2 > /proc/sys/vm/drop_caches    # free reclaimable slab objects (dentries, inodes)
echo 3 > /proc/sys/vm/drop_caches    # free slab objects and page cache
```

- Use before benchmarks for consistent cold-cache starting state

## Tuning

### Application Calls

#### posix_fadvise()

```c
int posix_fadvise(int fd, off_t offset, off_t len, int advice);
```

| Advice | Description |
|--------|-------------|
| `POSIX_FADV_SEQUENTIAL` | Data range will be accessed sequentially |
| `POSIX_FADV_RANDOM` | Data range will be accessed randomly |
| `POSIX_FADV_NOREUSE` | Data will not be reused |
| `POSIX_FADV_WILLNEED` | Data will be needed again soon — please cache |
| `POSIX_FADV_DONTNEED` | Data will not be needed again — no need to cache |

#### madvise()

```c
int madvise(void *addr, size_t length, int advice);
```

| Advice | Description |
|--------|-------------|
| `MADV_RANDOM` | Offsets accessed in random order |
| `MADV_SEQUENTIAL` | Offsets accessed in sequential order |
| `MADV_WILLNEED` | Data needed again — please cache |
| `MADV_DONTNEED` | Data not needed again — no need to cache |

### ext4

- Four tuning methods — mount options, `tune2fs(8)`, `/sys/fs/ext4` property files, `e2fsck(8)`

```bash
# Common performance mount options
mount -o noatime ...        # disable access timestamp updates
mount -o relatime ...       # default — reduce access timestamp writes

# View current settings
tune2fs -l /dev/nvme0n1p1   # show file system parameters
mount                        # show current mount flags

# View live tunables
cat /sys/fs/ext4/nvme0n1p1/inode_readahead_blks   # 32 default

# Directory reindexing
e2fsck -D -f /dev/hdX    # reindex directories — may improve lookup performance
```

- Key `/sys/fs/ext4` tunables — `inode_readahead_blks`, `mb_group_prealloc`, `mb_stream_req`, `max_writeback_mb_bump`
- Documented in `Documentation/admin-guide/ext4.rst`

### ZFS Dataset Parameters

```bash
zfs get all zones/var    # view all properties with source
zfs set recordsize=8K zones/var    # set record size to match application I/O
```

| Parameter | Options | Description |
|-----------|---------|-------------|
| `recordsize` | 512 to 128K | Suggested block size — tune to match application I/O size |
| `compression` | `on`, `off`, `lzjb`, `gzip`, `lz4` | `lzjb` lightweight — may improve performance by reducing I/O |
| `atime` | `on`, `off` | Disable if access timestamps not needed |
| `primarycache` | `all`, `none`, `metadata` | Use `none` or `metadata` for low-priority file systems (Eg - archives) to avoid ARC pollution |
| `secondarycache` | `all`, `none`, `metadata` | L2ARC policy |
| `logbias` | `latency`, `throughput` | `latency` — use SLOG device; `throughput` — use pool devices |
| `sync` | `standard`, `always`, `disabled` | Synchronous write behaviour |

> [!TIP]
> Record size is usually the most impactful ZFS tunable — default 128 KB is inefficient for small random I/O. Match to application I/O size. Dynamic record size applies to files smaller than the configured record size.