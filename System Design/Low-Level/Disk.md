# Disk

## Terminology

| Term | Definition |
|------|------------|
| Virtual disk | Emulation of a storage device — appears as single physical disk, may be built from multiple disks or a fraction of a disk |
| Transport | Physical bus used for communication — data transfers and other disk commands |
| Sector | Block of storage on disk — traditionally 512 bytes, today often 4 KB |
| I/O | Disk reads and writes only — described by direction, disk address, and size |
| Disk commands | Non-data-transfer commands — Eg - cache flush |
| Throughput | Current data transfer rate (bytes/s) |
| Bandwidth | Maximum possible data transfer rate — hardware-limited |
| I/O latency | Time for an I/O operation from start to end |
| Latency outliers | Disk I/O with unusually high latency |
| IOPS | I/O operations per second |

## Models

### Simple Disk

- Disk maintains an on-disk queue for I/O requests
- I/O is either waiting on queue or being actively serviced
- On-disk controller can apply algorithms beyond FIFO —
    - Elevator seeking for rotational disks
    - Separate read/write queues for flash-based disks

### Caching Disk

- On-disk cache (DRAM inside the physical device) satisfies some read requests at low latency
- Cache misses still incur full disk-device latency
- __Write-back cache__ — signals write complete after transfer to cache, before transfer to persistent storage
- __Write-through cache__ — signals write complete only after full transfer to next level
- Write-back caches in production storage are typically battery-backed to survive power failures

### Controller (HBA)

- Bridges CPU I/O transport with storage transport and attached disks
- Also called host bus adapters (HBAs)
- Performance may be limited by — CPU transport, storage transport, controller itself, or the disks

## Concepts

### Measuring Time

| Term | Definition |
|------|------------|
| I/O request time (response time) | Full time from issuing I/O to completion |
| I/O wait time | Time spent waiting on a queue |
| I/O service time | Time during which I/O was actively processed |
| Block I/O wait time (OS wait time) | Time from I/O creation and insertion into kernel queue to when it left the final kernel queue and was issued to device — spans multiple kernel queues |
| Block I/O service time | Time from issuing request to device to its completion interrupt |
| Block I/O request time | Block I/O wait time + block I/O service time — full time from I/O creation to completion |
| Disk wait time | Time spent on on-disk queue |
| Disk service time | Time after on-disk queue for I/O to be actively processed |
| Disk request time (disk I/O latency) | Disk wait time + disk service time — equals block I/O service time |

- Average disk service time can be inferred (not observed directly from kernel stats) —
    - disk service time = utilization / IOPS

- Eg - 60% utilisation, 300 IOPS — average service time = 2 ms
- Inaccurate for disks that process multiple I/O in parallel — use event tracing (`biolatency`) for accurate measurement

### Time Scales

| Event | Latency | Scaled (cache hit = 1s) |
|-------|---------|------------------------|
| On-disk cache hit | < 100 μs | 1 s |
| Flash memory read | ~100–1000 μs | 1–10 s |
| Rotational disk sequential read | ~1 ms | 10 s |
| Rotational disk random read (7200 rpm) | ~8 ms | 1.3 min |
| Rotational disk random read (slow, queueing) | > 10 ms | 1.7 min |
| Rotational disk random read (dozens in queue) | > 100 ms | 17 min |
| Worst-case virtual disk (RAID-5, queueing, random) | > 1000 ms | 2.8 hours |

> [!NOTE]
> NVMe storage devices typically achieve 10–20 μs latency. Disk returns a bimodal latency distribution — cache hits vs misses — expressing them as an average is misleading.

### Caching

| Cache Level | Example |
|-------------|---------|
| Device cache | ZFS vdev |
| Block cache | Buffer cache |
| Disk controller cache | RAID card cache |
| Storage array cache | Array cache |
| On-disk cache | DDC-attached DRAM |

### Random vs Sequential I/O

- Random I/O — requires disk head seek and platter rotation between I/Os
- Sequential I/O (streaming) — next I/O located at end of current I/O, no seek or rotation wait
- Flash-based SSDs — typically no performance difference between random and sequential reads
- Writes smaller than block size on flash — may incur read-modify-write penalty, especially random writes
- Disk offsets seen by OS may not match physical offsets — hardware virtual disks may remap them

### Read/Write Ratio

- Expressed as percentage of reads vs writes over time — Eg - "80% reads since boot"
- High read rate — benefits most from adding cache
- High write rate — benefits most from adding more disks for throughput and IOPS
- Reads and writes may have different patterns — Eg - random reads with sequential writes (copy-on-write filesystems)

### I/O Size

- Larger I/O sizes — higher throughput, longer per-I/O latency
- I/O size may be altered by — filesystem, volume manager, device driver
- Flash-based disks — often have optimal I/O sizes — Eg - 4 KB reads, 1 MB writes
- Find optimal sizes via micro-benchmarking or vendor documentation

### IOPS Are Not Equal

- IOPS without context is meaningless — always include —
    - Random vs sequential
    - I/O size
    - Read/write direction
    - Buffered vs direct
    - Number of parallel I/Os
- Sequential workloads — throughput-sensitive — larger I/O at lower IOPS may be better
- Random workloads — latency-sensitive — high IOPS is desirable
- Use time-based metrics (utilisation, service time) for easier cross-device comparison

### Non-Data-Transfer Disk Commands

- ATA `TRIM` / SCSI `UNMAP` — tells drive a sector range is no longer needed — helps SSDs maintain write performance
- Cache flush — commands disk to flush on-disk RAM cache to persistent storage
- These commands consume disk utilisation and can cause other I/O to wait

### Utilization

- Percentage of time disk was busy actively performing work during an interval
- 100% utilisation — likely performance issue if sustained
- Queueing and performance degradation may begin at lower values — Eg - 60% depending on workload and M/D/1 queueing behaviour
- High utilisation alone is not enough — also check response time and whether application is blocking on I/O
- Utilisation is an interval summary — bursts (Eg - write flushes) can be disguised in longer intervals

### Virtual Disk Utilisation Caveats

- OS-reported virtual disk utilisation may not reflect underlying physical disk activity —
    - Virtual disk at 100% built on multiple physical disks — may still accept more work (not all physical disks busy simultaneously)
    - Write-back cache controller — virtual disk may appear idle while physical disks are still draining writes
    - Hardware RAID rebuild — physical disks busy with no corresponding OS-visible I/O

### Saturation

- Measure of work queued beyond what the resource can deliver
- Calculated as average length of device wait queue in the OS
- A disk at 100% utilisation may have no queue (no saturation) or significant queue
- Even sub-100% utilisation can have brief saturation in bursts — use tracing to inspect per-I/O events when needed

### I/O Wait

- Per-CPU metric — time spent idle while threads on the dispatcher queue are blocked on disk I/O
- High I/O wait per CPU — disks may be a bottleneck, leaving CPUs idle
- Misleading metric — a new CPU-hungry process can reduce I/O wait without changing disk behaviour
- More reliable metric — time that application threads are blocked on disk I/O (measured via static or dynamic instrumentation)
- Practical interpretation — any I/O wait is a sign of a system bottleneck — concurrent I/O (no I/O wait) is less likely to directly block applications

### Synchronous vs Asynchronous

- Disk I/O latency may not directly impact application performance if I/O is asynchronous —
    - Write-back caching — application I/O completes early, disk I/O issued later
    - Read-ahead / prefetch — initiated by filesystem to warm cache
    - Async I/O worker threads — other threads continue processing while I/O thread blocks
    - Kernel async I/O APIs — application requests I/O and is notified on completion

### Disk vs Application I/O

- Disk I/O rate/volume may not match application-level I/O due to —
    - Filesystem inflation/deflation
    - Paging due to memory shortage
    - Device driver I/O size rounding or fragmentation
    - RAID mirror writes, parity/checksum blocks, read verification

## Architecture

### Disk Types

#### Magnetic Rotational (HDD)

- Platters coated with iron oxide — bits stored as magnetic orientation of small regions
- Mechanical arm with disk heads reads/writes across rotating platters
- Data stored in circular tracks, each divided into sectors
- Performance characteristics —
    - Slow for random I/O — seek time + rotation time both add milliseconds of latency
    - Sequential I/O is fast — no seek or rotation wait between adjacent I/Os
    - Higher RPM — lower rotation wait but shorter disk lifespan due to heat and wear
    - Available speeds — 5400, 7200, 10K, 15K RPM
- Still competitive for high-density archival storage (low cost per MB)

##### Seek and Rotation Reduction Strategies

- Caching — eliminate I/O entirely
- Filesystem placement and copy-on-write — makes writes sequential
- Separate workloads to separate disks — avoid cross-workload seeking
- Elevator seeking — on-disk controller reorders I/O by offset to minimise head travel
- Higher-density disks — tighter workload location
- Short-stroking — use only outer tracks, bounds head movement to smaller range, better throughput due to sector zoning
- Faster RPM disks

##### Sector Zoning

- Outer tracks are longer — store more sectors per track
- Outer tracks deliver higher throughput (MB/s) due to constant rotation speed
- Short-stroking exploits this — outer tracks give both lower seek time and higher throughput

##### Sector Size

- Advanced Format standard — 4 KB sectors
    - Reduces per-sector metadata and ECC overhead
    - Improves throughput
- 512-byte emulation (Advanced Format 512e) — provided by firmware, may add read-modify-write overhead
- Misaligned 4 KB I/O that spans two sectors — inflates sector I/O count

##### On-Disk Cache and Command Queueing

- Small DRAM inside the disk — caches reads, buffers writes, queues and reorders commands
- SCSI — Tagged Command Queueing (TCQ)
- SATA — Native Command Queueing (NCQ)

##### Elevator Seeking

- Reorders queued I/O by on-disk offset to minimise disk head travel
- Similar to a building elevator — services floors in sweep order, not request order
- I/Os complete out of request order — visible in disk I/O traces
- Risk — starvation of I/O at one offset if constant I/O near another offset keeps the heads busy — deadline scheduler mitigates this

##### Data Integrity

- ECC stored at end of each sector — drive verifies correctness on read, may retry on next rotation
- Retry explains occasional slow I/O outliers
- Soft error rate increase — early warning sign of impending drive failure
- 4 KB sectors require fewer ECC bits per unit of data than 512-byte sectors

##### SMR (Shingled Magnetic Recording)

- Narrower overlapping tracks — ~25% higher density
- Write head wider than read head — writes destroy neighbouring tracks, which must be rewritten
- Suitable for write-once archival workloads
- Not suited for write-heavy workloads or RAID configurations

#### Solid-State Drives (SSD)

- No moving parts — physically durable, not susceptible to vibration
- Consistent latency across offsets — no seek or rotation component
- Random vs sequential distinction matters much less than with HDDs
- Easier capacity planning — predictable latency for given I/O sizes

##### Flash Memory Types

| Type | Bits/Cell | Approx P/E Cycles | Notes |
|------|-----------|-------------------|-------|
| SLC | 1 | 50,000–100,000 | Highest performance and reliability, highest cost |
| MLC | 2 | 5,000–10,000 | Higher density, lower reliability |
| eMLC | 2 | - | MLC with enterprise firmware |
| TLC | 3 | ~3,000 | Higher density |
| QLC | 4 | ~1,000 | Highest density, lowest endurance |
| 3D NAND / V-NAND | varies | - | Stacked layers for higher density, available since 2013 |

##### Flash Memory Characteristics

- NAND flash — electron-based trapped-charge storage
- Write requires erasing entire block before rewriting (usually 256–512 KB per block, 8–64 KB per page)
- Asymmetric performance — fast reads, slower writes
- Write-back cache with capacitor battery backup — mitigates write latency
- Write amplification — writes smaller than block size require read-modify-write cycle for entire block

##### Flash Translation Layer (FTL)

- Controller translates between flash native operations and block device interface exposed to OS
- Input — reads/writes per page, writes only to erased pages, erases per block
- Output — arbitrary sector reads/writes (512 B or 4 KB)
- FTL uses a log-structured filesystem internally to manage block maps and free space
- `TRIM` (ATA) / `UNMAP` (SCSI) / `WRITE SAME` — informs SSD that a region is free, reduces write amplification
    - Linux support — `discard` mount option, `fstrim(8)` command

##### SSD Lifespan Management

- Wear leveling — spreads writes across blocks to reduce per-block write cycles
- Memory overprovisioning — reserves extra capacity for remapping worn blocks
- Enterprise SSDs — SLC + aggressive overprovisioning — 1M+ write cycles
- Consumer SSDs — MLC — as few as 1,000 cycles

##### SSD Pathologies

- Latency outliers from aging — drive retries harder to read degraded cells (ECC)
- Higher latency from FTL fragmentation — reformatting may help by rebuilding block maps
- Lower throughput if SSD uses internal compression

#### Persistent Memory

- Battery-backed DRAM — used for storage controller write-back caches
- Orders of magnitude faster than flash — limited to specialised uses due to cost and battery lifespan
- 3D XPoint (Intel Optane) —
    - Byte-addressable stackable cross-gridded array
    - 14 μs access latency vs 200 μs for 3D NAND SSD
    - Consistent latency distribution vs wide distribution for NAND
    - Available since 2017 — released as Optane DIMM (persistent memory) and Optane SSD

### Interfaces

| Interface | Max Bandwidth | Key Characteristics |
|-----------|--------------|---------------------|
| SCSI (parallel) | Hundreds of MB/s | Shared bus — contention under heavy load, replaced by SAS |
| SAS-1 | 3 Gbit/s | Point-to-point, 8b/10b encoding (80% efficiency) |
| SAS-2 | 6 Gbit/s | Dual porting, multipathing, SATA compatibility |
| SAS-3 | 12 Gbit/s | Enterprise preferred |
| SAS-4 | 22.5 Gbit/s | Latest SAS generation |
| SATA 1.0 | 1.5 Gbit/s | 8b/10b encoding (80% efficiency), consumer use |
| SATA 2.0 | 3.0 Gbit/s | Added NCQ support |
| SATA 3.0 | 6.0 Gbit/s | Common consumer standard |
| FC (Gen 7) | Up to 51,200 MB/s full duplex | SAN fabric — enterprise, switchable, multi-host |
| NVMe 1.4 | Bounded by PCIe (PCIe 4.0 x16 = 31.5 GB/s) | Direct PCIe attachment, 64K commands/queue, SR-IOV, < 20 μs latency |

> [!NOTE]
> NVMe supports multiple hardware queues per CPU — promotes CPU cache warmth, avoids shared kernel locks (Linux multi-queue). SAS/SATA limited to 256 and 32 commands per queue respectively vs 64K for NVMe.

### Storage Types

| Type | Description | Observability |
|------|-------------|---------------|
| Disk devices (JBOD) | Internal disks, OS controls each directly | Easiest — each disk observable separately |
| RAID (hardware) | Controller presents virtual disk — striping, mirroring, parity | Harder — OS sees virtual disk only |
| RAID (software) | OS manages physical disks directly — ZFS, md | Good — OS observes individual disks |
| Storage array | Many disks + large battery-backed cache + dual-attach | OS sees virtual disk — physical disk activity hidden |
| NAS | Storage over network (NFS, SMB, iSCSI) | Network becomes a factor — analyze as separate system |

#### RAID Types and Performance

| Level | Description | Performance Characteristics |
|-------|-------------|----------------------------|
| RAID-0 (concat) | Drives filled one at a time | Improves random read when multiple drives active |
| RAID-0 (stripe) | I/O striped across all drives in parallel | Best random and sequential I/O — no redundancy |
| RAID-1 (mirror) | Two drives store identical content | Good random and sequential reads — write throughput halved |
| RAID-10 | RAID-0 stripes across RAID-1 groups | RAID-1 performance with more groups increasing bandwidth |
| RAID-5 | Stripes with parity across disks | Poor write performance — read-modify-write cycle + parity calc |
| RAID-6 | RAID-5 with two parity disks per stripe | Worse than RAID-5 for writes |

- RAID-5 write optimisation — read only modified strips + parity, XOR to calculate new parity, write modified strips and new parity (avoids reading entire stripe)
- RAID-5 write-back cache — mitigates read-modify-write overhead — must be battery-backed
- Stripe size tuning — balance stripe size with average write I/O size to reduce additional read overhead

### Linux I/O Stack

#### I/O Merging

- Linux merges adjacent or overlapping I/O requests before dispatch — reduces per-I/O CPU overhead and disk overhead
- Front merge — new I/O prepended to existing request
- Back merge — new I/O appended to existing request
- Merge statistics available in `iostat(1)`

#### I/O Schedulers

- Classic schedulers (removed in Linux 5.0) — single request queue with single lock — bottleneck at high I/O rates

| Scheduler | Type | Description |
|-----------|------|-------------|
| Noop | Classic | No scheduling — suitable for RAM disks |
| Deadline | Classic | Enforces latency deadlines — read/write expiry in ms — solves starvation via three queues (read FIFO, write FIFO, sorted) |
| CFQ | Classic | Fair I/O time slices per process — priority and class support via `ionice(1)` |
| None | Multi-queue | No queueing |
| BFQ | Multi-queue | Budget fair queuing — per-process queues with sector budgets and budget timeout — cgroup support |
| mq-deadline | Multi-queue | blk-mq version of deadline |
| Kyber | Multi-queue | Adjusts queue lengths to meet target latency — two tunables: `read_lat_nsec`, `write_lat_nsec` — used at Netflix by default |

- Multi-queue (blk-mq, Linux 3.13+) — separate submission queues per CPU, multiple dispatch queues per device — parallel processing, same-CPU locality
- Default since Linux 5.0 — only multi-queue schedulers available
- Documented in `Documentation/block` in kernel source

## Methodology

### Tools Method

- Iterates over available tools examining key metrics
- May miss issues with poor tool visibility

- For Linux disks —
    - `iostat` — extended mode, check utilisation >60%, average service time >10 ms, high IOPS
    - `iotop` / `biotop` — identify processes causing disk I/O
    - `biolatency` — I/O latency distribution histogram — look for multimodal distributions and outliers >100 ms
    - `biosnoop` — individual I/O examination
    - `perf(1)` / BCC / `bpftrace` — custom analysis including user and kernel stacks that issued I/O
    - Disk controller vendor tools

### USE Method

#### Disk Devices

- __Utilisation__ — time the device was busy
- __Saturation__ — degree to which I/O is waiting in queue
- __Errors__ — device errors — check first (system may function correctly despite failures in a redundant pool)
- SMART data — additional error counters beyond standard OS metrics — use `smartctl`, `MegaCLI`
- Virtual disk utilisation — may not reflect physical disk activity — see Utilisation section

#### Disk Controllers

- __Utilisation__ — current vs maximum throughput (bytes/s) and operation rate (ops/s) — not time-based
- __Saturation__ — I/O waiting due to controller saturation
- __Errors__ — controller errors
- Check each transport independently — errors, utilisation, saturation
- Per-controller metrics often absent from `iostat` — sum per-disk metrics grouped by controller

### Performance Monitoring

- Key metrics —
    - Disk utilisation — 100% for multiple seconds is an issue; >60% may cause queueing issues
    - Response time — monitor as per-second average, also capture maximum and standard deviation
- Monitor per-disk — identifies unbalanced workloads and individual poorly performing disks
- Full distribution (histogram, heat map) preferred over averages for outlier detection
- Also monitor resource control metrics if I/O throttling is in use

### Workload Characterization

- Basic attributes —
    - I/O rate (IOPS)
    - I/O throughput
    - I/O size
    - Read/write ratio
    - Random vs sequential
- Capture maximums as well as averages — writes buffered and flushed at intervals cause spikes
- Examine full distribution over time where possible

#### Advanced Checklist

- IOPS — system-wide, per-disk, per-controller (reads and writes separately)
- Throughput — system-wide, per-disk, per-controller
- Which applications or users are using the disks
- Which filesystems or files are being accessed
- Errors encountered — invalid requests or disk issues
- I/O balance across available disks
- IOPS and throughput per transport bus
- Non-data-transfer disk commands issued
- Why disk I/O is issued — kernel call path
- Degree to which disk I/O is application-synchronous
- Distribution of I/O arrival times

#### Performance Characterization

- Utilisation per disk
- Saturation per disk — wait queue depth
- Average I/O service time
- Average I/O wait time
- I/O latency outliers
- Full I/O latency distribution
- Resource controls (I/O throttling) — active or not
- Latency of non-data-transfer commands

### Latency Analysis

- Drill down through stack layers to find source of latency
- Measure I/O latency at different levels — application, filesystem, block layer, disk driver, disk device
- If latency is similar across all levels — disk or disk driver is the cause
- If latency appears at a higher level (Eg - filesystem) with lower latency below — filesystem locking/queueing is the cause
- Different layers may inflate or deflate I/O — size, count, and latency will differ between layers
- Present as — per-interval averages, full distributions (histograms/heat maps), or per-I/O values

### Static Performance Tuning

- Configuration checklist —
    - Number and types of disks (SMR, MLC, etc.) and firmware versions
    - Number of disk controllers and interface types
    - Controller cards in high-speed slots
    - Disks per HBA
    - Battery backup power levels for controllers
    - RAID configuration — level, stripe width
    - Multipathing configured
    - Disk device driver version and known bugs/patches
    - Server main memory size and page/buffer cache usage
    - Resource controls in use for disk I/O

### Cache Tuning

- Check which caches exist (application, filesystem, controller, on-disk)
- Check that each cache is working and measure hit rate
- Tune workload for cache (Eg - access patterns)
- Tune cache for workload (Eg - cache size)

### Micro-Benchmarking

- Test via raw device path — bypasses all filesystem behaviour (caching, buffering, I/O splitting, coalescing)
- Factors —
    - Direction — reads or writes
    - Offset pattern — random or sequential
    - Offset range — full disk or tight range (Eg - offset 0 only for on-disk cache measurement)
    - I/O size — 512 B to 1 MB
    - Concurrency — parallel I/O count or thread count
    - Number of devices — single disk or multi-disk to find controller/bus limits

#### Per-Disk Tests

| Test | Workload |
|------|----------|
| Max throughput | 128 KB or 1 MB reads, sequential |
| Max IOPS | 512-byte reads, offset 0 only |
| Max random IOPS | 512-byte reads, random offsets |
| Read latency profile | Sequential reads, repeat for 512 B, 1 KB, 2 KB, 4 KB, etc. |
| Random I/O latency profile | 512-byte reads, full span / beginning / end offsets |

#### Per-Controller Tests

| Test | Workload |
|------|----------|
| Max throughput | 128 KB reads, offset 0, across multiple disks |
| Max IOPS | 512-byte reads, offset 0, across multiple disks |

- Add disks one at a time until controller limit is found — may require 12+ disks

### Scaling

- Disks and controllers have hard throughput and IOPS limits — tuning cannot exceed them
- Scaling process —
    - Determine target workload in throughput and IOPS
    - Calculate number of disks required — use target utilisation (Eg - 50%), not max capacity
    - Calculate number of controllers required
    - Verify transport limits are not exceeded
    - Calculate CPU cycles per disk I/O and required CPU count
- Cache scaling — if cache is not scaled proportionally with disk count, cache-per-user ratio drops, disk workload increases

## Disk Observability Tools

## iostat

- First tool used for disk I/O investigation — summarises per-disk I/O statistics
- Statistics enabled by default in kernel — overhead negligible
- Source — kernel block layer accounting (enabled via `/sys/block/<dev>/queue/iostats`)

### Default Output Columns

| Column | Description |
|--------|-------------|
| `tps` | Transactions per second (IOPS) |
| `kB_read/s` | Kilobytes read per second |
| `kB_wrtn/s` | Kilobytes written per second |
| `kB_read` | Total kilobytes read since boot |
| `kB_wrtn` | Total kilobytes written since boot |

```bash
iostat              # summary since boot
iostat 1            # 1-second intervals
iostat 1 10         # 1-second intervals, 10 times
iostat -c           # CPU report
iostat -d           # disk report only
iostat -k           # kilobytes instead of 512-byte blocks
iostat -m           # megabytes
iostat -p           # include per-partition statistics
iostat -t           # timestamp output
iostat -x           # extended statistics
iostat -s           # short (narrow) output — fits 80 chars
iostat -z           # skip zero-activity devices
iostat -sxz 1       # short extended, skip zeros, 1-second interval
iostat -dmstxz -p ALL 1   # MB, disk only, timestamp, short extended, zero-skip, per-partition
```

### Extended Short Output Columns (`-sx`)

| Column | Description |
|--------|-------------|
| `tps` | Transactions per second (IOPS) |
| `kB/s` | Kilobytes per second |
| `rqm/s` | Requests queued and merged per second |
| `await` | Average I/O response time including OS queue time and device time (ms) |
| `aqu-sz` | Average number of requests waiting in driver queue and active on device |
| `areq-sz` | Average request size (KB) — after merging |
| `%util` | Percent of time device was busy (utilisation) |

- `await` — most important metric for delivered performance — increases due to queueing, larger I/O, random I/O, device errors
- `%util` — busyness measure only — misleading for virtual devices backed by multiple disks
- Non-zero `rqm/s` — contiguous requests merged before delivery — sign of sequential workload
- `areq-sz` ≤ 8 KB — indicator of random I/O workload that could not be merged

### Extended Full Output Columns (`-x`)

| Column | Description |
|--------|-------------|
| `r/s`, `w/s`, `d/s`, `f/s` | Read, write, discard, flush requests completed per second (after merges) |
| `rkB/s`, `wkB/s`, `dkB/s` | Read, write, discard KB/s |
| `%rrqm`, `%wrqm`, `%drqm` | Read, write, discard requests merged as percentage of total for that type |
| `r_await`, `w_await`, `d_await`, `f_await` | Read, write, discard, flush average response time (ms) |
| `rareq-sz`, `wareq-sz`, `dareq-sz` | Read, write, discard average size (KB) |

- Split read/write metrics — `r_await` is most relevant to application performance — writes often masked by write-back caching
- Discard stats added Linux 4.19 — flush stats added Linux 5.5
- `iostat` does not include disk errors — cannot complete full USE method from one tool

> [!NOTE]
> `iostat` reports block device reads and writes — it does not show file system I/O. An application performing I/O to the filesystem may not appear in `iostat` if all I/O is served from page cache.

## sar

- System activity reporter — disk summary via `-d` option
- Columns identical to `iostat -x` — `tps`, `rkB/s`, `wkB/s`, `dkB/s`, `areq-sz`, `aqu-sz`, `await`, `%util`
- Historical archiving capability — useful for post-incident analysis

```bash
sar -d 1    # disk statistics, 1-second interval
```

> [!NOTE]
> `svctm` (average inferred disk service time) has been removed from recent `sar` versions — its calculation was inaccurate for modern disks that process I/O in parallel.

## PSI (Pressure Stall Information)

- Linux 4.20+ — I/O saturation statistics and trend over last 5 minutes
- Read from `/proc/pressure/io`
- `some` line — percentage of time at least one task was I/O stalled
- `full` line — percentage of time all runnable tasks were I/O stalled
- Rising `avg10` vs `avg300` — increasing I/O pressure

```bash
cat /proc/pressure/io
# some avg10=63.11 avg60=32.18 avg300=8.62 total=667212021
# full avg10=60.76 avg60=31.13 avg300=8.35 total=622722632
```

- High-level alerting metric — drill into root cause with `pidstat`, `biolatency`, `biosnoop`

## pidstat

- Per-process disk I/O statistics — requires Linux kernel 2.6.20+
- Only root can access disk stats for processes it does not own — reads from `/proc/PID/io`

| Column | Description |
|--------|-------------|
| `kB_rd/s` | Kilobytes read per second |
| `kB_wr/s` | Kilobytes issued for write per second |
| `kB_ccwr/s` | Kilobytes cancelled for write (overwritten or deleted before flush) |
| `iodelay` | Time process was blocked on disk I/O (clock ticks) — includes swap I/O |

```bash
pidstat -d 1    # disk I/O stats, 1-second interval
```

> [!NOTE]
> `iodelay` shows magnitude of application pain from disk I/O — `kB_rd/s` and `kB_wr/s` show the workload applied. Write-back caching means writes may not show `iodelay` until the page cache flush occurs via a `kworker` process.

## perf

- Records block I/O tracepoints with stack traces — identifies code paths causing disk I/O

### Block Tracepoints

```bash
perf list 'block:*'    # list all available block tracepoints
```

| Tracepoint | Description |
|------------|-------------|
| `block:block_rq_insert` | I/O inserted into request queue |
| `block:block_rq_issue` | I/O issued to device driver |
| `block:block_rq_complete` | I/O completion |
| `block:block_bio_frontmerge` | Front merge of I/O |
| `block:block_bio_backmerge` | Back merge of I/O |
| `block:block_split` | I/O split |
| `block:block_rq_remap` | I/O remapped to different device |

### One-Liners

```bash
# Record block device issues with stack traces for 10 seconds
perf record -e block:block_rq_issue -a -g sleep 10
perf script --header

# Trace completions of size >= 100 KB (200 sectors at 512 bytes)
perf record -e block:block_rq_complete --filter 'nr_sector > 200'

# Trace synchronous write completions only
perf record -e block:block_rq_complete --filter 'rwbs == "WS"'

# Trace all write completions
perf record -e block:block_rq_complete --filter 'rwbs ~ "*W*"'

# Record issue and completion events for latency calculation (60s)
perf record -e block:block_rq_issue,block:block_rq_complete -a sleep 60
perf script --header > out.disk01.txt
```

- `block_rq_issue` tracepoint fields — disk major/minor, I/O type (rwbs), size (bytes), sector address, process name
- Stack trace reveals full kernel and user-space code path — Eg - `fsync()` via `ext4_writepages()` via `blk_mq_try_issue_list_directly()`
- I/O queued then issued by kernel thread — `block_rq_issue` may show kernel worker, not originating process — use `block_rq_insert` to capture originating process (but misses I/O that bypasses queueing)

> [!TIP]
> Post-process issue/completion output with awk, Python, or R to calculate per-I/O latency. Or use `biolatency`/`biosnoop` which do this efficiently in kernel space.

## biolatency

- BCC and bpftrace tool — shows disk I/O latency as a power-of-2 histogram
- Measures from device issue to device completion (disk request time)

```bash
biolatency 10 1         # trace for 10 seconds, print once
biolatency -m           # output in milliseconds
biolatency -Q           # include OS queue time (block I/O request time)
biolatency -F           # separate histogram per I/O flag set
biolatency -D           # separate histogram per disk device
```

- Bi-modal distribution — indicates two classes of I/O — Eg - on-disk cache hits vs misses, or reads vs writes
- `-F` flag breakdown — separates `Read`, `Write`, `Sync-Write`, `ReadAhead-Read`, `NoMerge-Write`, `Flush` etc. — identifies which I/O type is causing latency
- `-Q` flag — adds OS queue wait time — reports full block I/O request time vs device-only time
- Outliers at 16–32 ms range — likely caused by device-side queueing

> [!NOTE]
> Average latency from `iostat` hides multimodal distributions. Always check `biolatency` histogram when investigating disk performance — a bimodal distribution points to different I/O classes needing separate investigation.

## biosnoop

- BCC and bpftrace tool — prints one line per disk I/O with process, type, size, and latency

| Column | Description |
|--------|-------------|
| `TIME(s)` | I/O completion time (seconds) |
| `COMM` | Process name (best effort) |
| `PID` | Process ID (best effort) |
| `DISK` | Storage device name |
| `T` | Type — R = read, W = write |
| `SECTOR` | Address on disk (512-byte sector units) |
| `BYTES` | I/O size (bytes) |
| `LAT(ms)` | Duration from device issue to completion (disk request time) |

```bash
biosnoop               # trace all disk I/O
biosnoop -Q            # add QUE(ms) column — OS queue wait time (block I/O wait time)

# Outlier analysis
biosnoop > out.biosnoop01.txt
sort -n -k 8,8 out.biosnoop01.txt | tail -5    # top 5 slowest I/O
```

- Queued write pattern — group of writes issued simultaneously, completing in turn with increasing latency — subtract `LAT(ms)` from `TIME(s)` to get start times
- I/O reordering — identified by comparing start vs completion order
- Outlier analysis — look at events prior to outlier for similar latency (queueing) or a gap (VM de-schedule by hypervisor)
- `QUE(ms)` column — time from I/O creation to device issue — mostly OS queue wait, also includes memory allocation and lock acquisition

> [!TIP]
> Use R or Python scatter plots on `biosnoop` output to visualise patterns at scale when thousands of events are present.

## iotop and biotop

### iotop

- Linux tool based on kernel accounting statistics
- Refreshes screen every second showing top disk I/O processes

| Column | Description |
|--------|-------------|
| `DISK READ` | Read KB/s |
| `DISK WRITE` | Write KB/s |
| `SWAPIN` | Percent of time thread waited for swap-in I/O |
| `IO` | Percent of time thread waited for I/O |

```bash
iotop -bod5    # batch mode, I/O processes only, 5-second interval
iotop -a       # accumulated statistics instead of per-interval
iotop -p PID   # filter to process
```

> [!NOTE]
> `iotop` has been observed to significantly undercount write workloads. Verify against a known workload before relying on it.

### biotop

- BCC tool — `top` for disks, BPF-based instrumentation

| Column | Description |
|--------|-------------|
| `PID` | Cached process ID (best effort) |
| `COMM` | Cached process name (best effort) |
| `D` | Direction — R = read, W = write |
| `MAJ MIN` | Disk major and minor numbers |
| `DISK` | Disk name |
| `I/O` | Number of disk I/Os during interval |
| `Kbytes` | Total throughput during interval (KB) |
| `AVGms` | Average I/O latency from device issue to completion (ms) |

```bash
biotop              # 1-second interval default
biotop 5            # 5-second interval
biotop -C           # do not clear screen
biotop -r MAXROWS   # specify number of top processes to display
```

> [!NOTE]
> PID and COMM are best-effort — by the time disk I/O reaches the device, the requesting process may no longer be on CPU.

## biostacks

- bpftrace tool — traces block I/O request time with the I/O initialisation kernel stack trace
- Measures from OS enqueue to device completion (block I/O request time)
- Identifies root cause of disk I/O including background filesystem tasks not initiated by applications

```bash
biostacks.bt    # trace until Ctrl-C, print histogram per unique stack
```

- Output — latency histogram (microseconds) per unique kernel stack that initiated the I/O
- Useful for identifying mysterious disk I/O — Eg - ZFS background scrubber, filesystem metadata lookups during `access(2)` syscalls

> [!TIP]
> When disk I/O appears without an obvious application cause, use `biostacks` first — it reveals background kernel activity such as filesystem journal flushes, `kswapd` paging, or storage scrubbers.

## blktrace

- Kernel-level block device I/O event tracer — specialised tracer using `BLKTRACE` ioctl syscalls
- Frontend tools — `blktrace(8)`, `blkparse(1)`, `btrace(8)` (combines both)
- Low-level — shows multiple events per I/O

```bash
btrace /dev/sda                         # equivalent to blktrace + blkparse
blktrace -d /dev/nvme0n1p1 -o out -w 10  # write trace files for 10s
blkparse -i out.blktrace.* -d out.bin    # parse to binary
btt -i out.bin                           # analyse timing statistics
btrace -a issue /dev/sdb                # filter to D (issue) events only
btrace -a read /dev/sdb                 # reads only
btrace -a write /dev/sdb                # writes only
```

### Action Identifiers

| Code | Description |
|------|-------------|
| `A` | I/O remapped to different device |
| `B` | I/O bounced |
| `C` | I/O completion |
| `D` | I/O issued to driver |
| `F` | Front merged with request on queue |
| `G` | Get request |
| `I` | I/O inserted onto request queue |
| `M` | Back merged with request on queue |
| `P` | Plug request |
| `Q` | I/O handled by request queue code |
| `S` | Sleep request |
| `T` | Unplug due to timeout |
| `U` | Unplug request |
| `X` | Split |

### RWBS I/O Type Characters

| Character | Meaning |
|-----------|---------|
| `R` | Read |
| `W` | Write |
| `M` | Metadata |
| `S` | Synchronous |
| `A` | Read-ahead |
| `F` | Flush or force unit access |
| `D` | Discard |
| `E` | Erase |
| `N` | None |

- Characters can be combined — Eg - `WM` = metadata write, `WS` = synchronous write

### btt Timing Statistics

| Metric | Description |
|--------|-------------|
| `Q2C` | Total time from I/O request to completion (block layer time) |
| `D2C` | Device issue to completion (disk I/O latency) |
| `I2D` | Device queue insertion to device issue (request queue time) |
| `M2D` | I/O merge to issue |
| `Q2Q` | Time between consecutive I/O requests |
| `Q2G` | Time from request queue to get request |

## bpftrace

- BPF-based tracer — custom disk analysis with one-liners and scripts

### One-Liners

```bash
# Count block I/O tracepoint events by type
bpftrace -e 'tracepoint:block:* { @[probe] = count(); }'

# Summarise block I/O size as histogram
bpftrace -e 't:block:block_rq_issue { @bytes = hist(args->bytes); }'

# Count block I/O request user stack traces (at queue insertion)
bpftrace -e 't:block:block_rq_insert { @[ustack] = count(); }'

# Count block I/O type flags
bpftrace -e 't:block:block_rq_issue { @[args->rwbs] = count(); }'

# Trace block I/O errors with device and I/O type
bpftrace -e 't:block:block_rq_complete /args->error/ {
    printf("dev %d type %s error %d\n", args->dev, args->rwbs, args->error); }'

# Count SCSI opcodes
bpftrace -e 't:scsi:scsi_dispatch_cmd_start { @opcode[args->opcode] = count(); }'

# Count SCSI result codes
bpftrace -e 't:scsi:scsi_dispatch_cmd_done { @result[args->result] = count(); }'

# Count SCSI driver functions
bpftrace -e 'kprobe:scsi* { @[func] = count(); }'
```

### Disk I/O Size Distribution

```bash
# I/O size histogram by process name
bpftrace -e 't:block:block_rq_issue /args->bytes/ { @[comm] = hist(args->bytes); }'

# I/O size histogram by process name and I/O type
bpftrace -e 't:block:block_rq_insert /args->bytes/ { @[comm, args->rwbs] = hist(args->bytes); }'
```

- Small I/O (4–32 KB) from a single process — indicates unmerged random I/O — consider batching at application level
- Large I/O or merged sequential — appears as 128 KB+ buckets

### Disk I/O Latency (biolatency.bt)

```bash
biolatency.bt    # histogram of disk request time in microseconds
```

```bpftrace
tracepoint:block:block_rq_issue
{
    @start[args->dev, args->sector] = nsecs;
}

tracepoint:block:block_rq_complete
/@start[args->dev, args->sector]/
{
    @usecs = hist((nsecs - @start[args->dev, args->sector]) / 1000);
    delete(@start[args->dev, args->sector]);
}
```

- Key — `[args->dev, args->sector]` — assumes at most one in-flight I/O per sector at a time
- Completion event fires on whatever is currently on CPU — cannot use thread ID as key (unlike VFS tracing)
- Add `args->rwbs` to histogram key to break down by I/O type

### Disk I/O Errors (bioerr.bt)

```bpftrace
tracepoint:block:block_rq_complete
/args->error != 0/
{
    time("%H:%M:%S ");
    printf("device: %d,%d, sector: %d, bytes: %d, flags: %s, error: %d\n",
        args->dev >> 20, args->dev & ((1 << 20) - 1), args->sector,
        args->nr_sector * 512, args->rwbs, args->error);
}
```

## MegaCli

- Tool for LSI disk controllers — observes controller internals not visible to OS
- Shows recent controller events, adapter info, virtual device info, battery status, physical errors

```bash
MegaCli -AdpEventLog -GetLatest 50 -f lsi.log -aALL    # last 50 controller events
more lsi.log    # shows patrol reads, rebuilds, errors with timestamps
```

- Patrol read events visible — explains unexpected disk I/O and latency during scheduled patrol periods
- Cannot explain individual slow I/O at millisecond resolution — use kernel tracing for that

## smartctl

- Reads SMART (Self-Monitoring, Analysis and Reporting Technology) data from disk firmware
- Shows health statistics, error counters, temperature, powered hours

```bash
smartctl --all -d megaraid,0 /dev/sdb    # SMART data for first disk in RAID virtual device
```

### Key SMART Data

| Field | Description |
|-------|-------------|
| `SMART Health Status` | Overall health — OK or FAILED |
| `Elements in grown defect list` | Remapped sectors — increasing count indicates degrading disk |
| `Errors Corrected by ECC fast` | Errors corrected without re-reads |
| `Errors Corrected by rereads/rewrites` | Errors requiring retries — increasing is a warning sign |
| `Total uncorrected errors` | Uncorrected read/write errors — non-zero is serious |
| `Non-medium error count` | Interface/controller errors |
| `Number of hours powered up` | Disk age |

- Corrected errors rate — monitor over time — increasing rate predicts impending failure
- Cannot resolve individual slow I/O events — use kernel tracers for latency investigation

## SCSI Logging

- Built-in Linux facility — logs SCSI events to `dmesg` / system log

```bash
# Enable maximum logging for all event types (warning — may flood system log)
sysctl -w dev.scsi.logging_level=03333333333
echo 03333333333 > /proc/sys/dev/scsi/logging_level

# Using sg3-utils scsi_logging_level tool
scsi_logging_level -s --all 3

# View events
dmesg
```

- Bitfield format — sets logging level 1–7 for 10 different event types (octal)
- Defined in `drivers/scsi/scsi_logging.h`
- Useful for debugging SCSI errors and timeouts — timestamps present but latency calculation difficult without unique I/O identifiers

## Other Tools

| Tool | Description |
|------|-------------|
| `vmstat` | Virtual memory statistics including swapping |
| `swapon` | Swap device usage |
| `seeksize` | Show requested I/O seek distances |
| `biopattern` | Identify random/sequential disk access patterns |
| `bioerr` | Trace disk errors |
| `mdflush` | Trace md flush requests |
| `iosched` | Summarise I/O scheduler latency |
| `scsilatency` | Show SCSI command latency distributions |
| `scsiresult` | Show SCSI command result codes |
| `nvmelatency` | Summarise NVMe driver command latency |
| `/proc/diskstats` | High-level per-disk statistics |
| `seekwatcher` | Visualises disk access patterns |

## Visualizations

### Line Graphs

- Graph IOPS, throughput, utilisation over time — shows time-based patterns and recurring events
- Caveats —
    - Average latency hides multimodal distributions and outliers
    - Cross-device averages hide unbalanced behaviour and single-device outliers
    - Long interval averages hide short-term fluctuations

### Latency Scatter Plots

- x-axis — completion time, y-axis — I/O response time
- Visualises per-event latency — reads and writes plotted differently
- Useful for identifying outliers and correlating them with bursts of other I/O types
- Limitation — merges into unreadable "paint" at millions of events — use heat maps at scale

### Latency Heat Maps

- x-axis — time, y-axis — I/O latency, z-axis — count (darker = more I/O)
- Scales to millions of events — avoids the scatter plot "paint" problem
- Reveals patterns invisible in averages — Eg - bimodal distributions, queueing steps, controller port saturation

### Offset Heat Maps

- x-axis — time, y-axis — disk block address (offset)
- Darker lines — sequential I/O (many I/Os at same offset range)
- Lighter clouds — random I/O
- Tools — `seekwatcher` (Linux), `DTraceTazTool`

### Utilisation Heat Maps

- y-axis — percent utilisation, color — number of disks at that utilisation level
- Identifies hot disks and sloth disks as lines at 100% on the heat map

## Experimentation

### Ad Hoc (dd)

```bash
# Sequential read, 1 MB I/O size, 1 GB total
dd if=/dev/sda1 of=/dev/null bs=1024k count=1k

# Sequential write with direct I/O to filesystem file (safer than raw device)
dd if=/dev/zero of=out1 bs=1024k count=1000 oflag=direct

# Direct I/O read
dd if=inputfile of=/dev/null bs=1024k iflag=direct
```

> [!NOTE]
> Without `oflag=direct`/`iflag=direct`, `dd` measures page cache performance, not disk performance. Always run `iostat` in a separate terminal to verify disk I/O is actually occurring.

### ioping

- Disk micro-benchmark resembling `ping` — 4 KB read per second by default
- Very lightweight — drives disk to ~0.4% utilisation
- Useful for production latency checks where full benchmark tools are unsuitable

```bash
ioping /dev/nvme0n1    # 4 KB read per second, print latency per request
```

### fio

- Flexible I/O tester — supports custom workload patterns
- Use `--direct=true` to bypass filesystem cache and test disk device performance

### blkreplay

- Replays block I/O loads captured with `blktrace`
- Useful for reproducing disk issues that are difficult to generate with synthetic benchmarks

> [!NOTE]
> Replays can be misleading if the target system has changed since capture — Eg - different cache state, disk layout, or controller.

### hdparm

```bash
hdparm -Tt /dev/sdb    # -T = cached reads, -t = disk device reads
```

- Demonstrates dramatic difference between on-disk cache hits and misses
- Various device tunables available — read man page carefully — some options marked DANGEROUS (risk of data loss)

## Tuning

### ionice

- Sets I/O scheduling class and priority for a process

| Class | ID | Description |
|-------|----|-------------|
| None | 0 | Kernel picks default — best effort based on `nice` value |
| Real-time | 1 | Highest priority — can starve other processes if misused |
| Best effort | 2 | Default class — priorities 0 (highest) to 7 |
| Idle | 3 | I/O only allowed after disk idle grace period |

```bash
ionice -c 3 -p 1623    # put PID 1623 in idle class — useful for backup jobs
```

### cgroups blkio Resource Controls

- Proportional weight (share) or fixed limit for processes/process groups
- Can set read and write limits independently
- Limits available for both IOPS and throughput (bytes/s)

### Kernel Tunable Parameters

| Parameter | Description |
|-----------|-------------|
| `/sys/block/*/queue/scheduler` | I/O scheduler — `none`, `mq-deadline`, `bfq`, `kyber` |
| `/sys/block/*/queue/nr_requests` | Number of read or write requests allocatable by block layer |
| `/sys/block/*/queue/read_ahead_kb` | Maximum read-ahead KB for filesystem requests |

- Full list in Linux source — `Documentation/block/queue-sysfs.txt`

### Disk Controller Tunables (Dell PERC example)

| Tunable | Description |
|---------|-------------|
| `Predictive Fail Poll Interval` | How often drives are polled for predictive failure indicators |
| `Interrupt Throttle Active Count` | Interrupt coalescing count threshold |
| `Interrupt Throttle Completion` | Interrupt coalescing time threshold |
| `Rebuild Rate` | Percentage of controller resources allocated to RAID rebuild |
| `Cache Flush Interval` | Seconds between flushing dirty write-back cache to disk — longer reduces I/O via write cancellation but increases read latency during flushes |
| `Max Drives to Spinup at One Time` | Staggers disk spinup to reduce power surge |
| `Load Balance Mode` | Auto load balancing across paths |