## Application Performance

## Application Basics

### Key Questions Before Analysis

- Function — what is the role (database, web server, load balancer, file server, object store)
- Operation — what requests does it serve and at what rate
- Performance requirements — SLO (Eg - 99.9% of requests < 100 ms)
- CPU mode — user-level process or kernel-level (Eg - NFS, BPF programs)
- Configuration — tunable parameters changed from defaults (buffer sizes, cache sizes, parallelism)
- Host — CPUs, memory topology, storage, limits
- Metrics — operation rate, provided via tools, API, or logs
- Logs — what is available, what can be enabled, latency details (Eg - MySQL slow query log)
- Version — latest version, performance fixes in release notes
- Bugs — known performance bugs in bug database for current version
- Source code — open source allows code path study from profilers and tracers
- Community — forums, blogs, IRC, Slack, meetups, conferences — shared performance findings
- Experts — recognised performance experts for the application

### Objectives

- Latency — low or consistent application response time
- Throughput — high operation rate or data transfer rate
- Resource utilisation — efficiency for a given workload
- Price — improving performance/price ratio

#### Quantified Examples

| Metric | Example Target |
|--------|---------------|
| Average request latency | 5 ms |
| 95th percentile | 100 ms or less |
| Outlier elimination | Zero requests beyond 1000 ms |
| Maximum throughput | 10,000 requests/s per server |
| Disk utilisation | Under 50% at 10,000 requests/s |

#### Apdex

- Apdex = (satisfactory + 0.5 × tolerable + 0 × frustrating) / total events
- Range — 0 (no satisfied customers) to 1 (all satisfied)

### Optimise the Common Case

- Find the most common code path for production workload — improve that first
- CPU-bound — focus on frequently on-CPU code paths
- I/O-bound — focus on code paths that frequently lead to I/O
- Use stack traces and flame graphs to identify the common case

### Observability

- Biggest performance wins often come from eliminating unnecessary work
- Application with rich observability tools will outperform opaque application in the long run — even if initially 10% slower
- Observability enables finding and eliminating unnecessary work and better tuning active work
- Prefer mature languages (Java, C) with many observability tools over new languages with limited tooling

### Big O Notation

| Notation | Examples |
|----------|---------|
| O(1) | Boolean test |
| O(log n) | Binary search of sorted array |
| O(n) | Linear search of linked list |
| O(n log n) | Quick sort (average case) |
| O(n²) | Bubble sort (average case) |
| O(2^n) | Factoring numbers |
| O(n!) | Brute force travelling salesman |

- Used to estimate algorithm speedup — Eg - linear vs binary search on 100 items = 21x difference (100 / log(100))
- O(n²) algorithms become pathological at scale — common source of production issues
- Fix — use more efficient algorithm or partition the population differently
- Constant costs ignored by Big O — may dominate when n is small

## Application Performance Techniques

### Selecting I/O Size

- I/O has fixed "initialisation tax" per call — syscall, mode switch, kernel metadata, privilege checks, address mapping
- Larger I/O sizes amortise fixed cost — far more efficient to do 1 × 128 KB than 128 × 1 KB
- Rotational disk I/O historically had high per-I/O cost due to seek time
- Downside — unnecessarily large I/O wastes bandwidth and cache space
- Eg - database doing 8 KB random reads should not use 128 KB disk I/O size — 120 KB wasted per I/O

### Caching

- Store results of expensive operations in a local cache for reuse
- Eg - database buffer cache stores results of common queries
- Configure cache sizes to suit the system during deployment

### Buffering

- Coalesce writes in a buffer before sending to next level — increases effective I/O size
- May increase write latency — first write waits for subsequent writes before being sent
- __Ring buffer (circular buffer)__ — fixed-size buffer for continuous async transfer between components — implemented with start/end pointers

### Polling

- Checks event status in a loop with pauses — CPU overhead and latency between event occurrence and next check
- Alternative — event-based listening — application notified immediately
- `poll(2)` — event-based, supports multiple FDs as array — scanning is O(n) — scalability issue
- `epoll(2)` (Linux) / `kqueue(2)` (BSD) — avoids scan — O(1) — preferred for high connection counts

### Concurrency and Parallelism

- Concurrency — multiple programs in flight simultaneously — does not require multiple CPUs
- Parallelism — executing on multiple CPUs simultaneously — requires multiple processes or threads
- Multiple threads preferred over multiple processes — same address space, more efficient

#### User-Mode Scheduling Mechanisms

| Mechanism | Description |
|-----------|-------------|
| Fibers | User-mode lightweight threads — application controls scheduling — less overhead than OS threads |
| Co-routines | Lighter than fibers — user-mode schedulable subroutines — still requires kernel for I/O |
| Event-based concurrency | Program broken into event handlers — Eg - Node.js single event worker thread |

#### Multithreaded Programming Models

| Model | Description |
|-------|-------------|
| Service thread pool | Pool of threads — each thread services one client connection at a time |
| CPU thread pool | One thread per CPU — long-duration batch processing (Eg - video encoding) |
| SEDA (Staged Event-Driven Architecture) | Application requests decomposed into stages — each processed by pools of threads |

#### Synchronization Primitives

| Primitive | Description |
|-----------|-------------|
| Mutex (MUTually EXclusive) lock | Only holder operates — others block and wait off-CPU |
| Spin lock | Holder operates — others spin on-CPU in tight loop checking for release — low latency but wastes CPU |
| RW lock | Multiple readers OR one writer — no readers during write |
| Semaphore | Counting (N parallel operations) or binary (effectively mutex) |

- Linux adaptive mutex — three paths depending on lock state —
    - fastpath — `cmpxchg` instruction — succeeds if lock not held
    - midpath (optimistic spinning) — spins while lock holder is running on another CPU
    - slowpath — blocks and deschedules — woken when lock available
- RCU (Read-Copy-Update) — read operations without acquiring lock — writes create a copy — replace original when no more readers — high performance for read-heavy kernel code

#### Hash Tables of Locks

- Problem with single global lock — all concurrent access serialises
- Problem with per-struct lock — creation/destruction overhead per data structure
- Hash table of locks — fixed number of locks — hashing selects which lock per data structure
- Number of buckets — ideally >= CPU count for maximum parallelism
- Hash collision — chain of data structures under same bucket — long chains hurt performance — walk is serialised under one lock
- False sharing — multiple locks in same cache line — two CPUs updating different locks cause cache coherency overhead — fix with padding

### Non-Blocking I/O

- Blocking model — each I/O consumes a thread while waiting — many concurrent I/O requires many threads
- High-frequency short-lived I/O — context switching overhead consumes CPU and adds latency
- Non-blocking I/O — issues I/O asynchronously — current thread continues other work

| Mechanism | Description |
|-----------|-------------|
| `open(2)` with `O_ASYNC` | Process notified via signal when I/O becomes possible |
| `io_submit(2)` | Linux AIO — asynchronous I/O submission |
| `sendfile(2)` | Copies data between FDs in kernel — avoids user-level I/O overhead |
| `io_uring_enter(2)` | Ring buffer shared between user and kernel space — added Linux 5.1 |

### Processor Binding

- NUMA environments — keeping thread on same CPU improves memory locality
- OS already implements CPU affinity — application can force via explicit binding
- Risk — binding conflicts with other CPU bindings (Eg - device interrupt mappings)
- Container risk — application sees all CPUs and binds to subset — multiple tenants may bind to same CPUs causing contention even when other CPUs are idle
- Risk over time — host changes, stale bindings bind across sockets unnecessarily

## Programming Languages

### Compiled Languages

- Machine instructions generated in advance — stored as ELF (Linux/Unix) or PE (Windows) binaries
- High-performing — no further translation needed at runtime
- Symbol table maps addresses to function/object names — enables CPU profiling and stack trace translation
- Eg - Linux kernel written mostly in C with critical paths in assembly

#### Compiler Optimisations

- gcc optimisation levels — `-O0` (least) through `-O3` (most), `-Os` (size), `-Og` (debug), `-Ofast` (all + non-standard)
- `-fomit-frame-pointer` — enabled at higher optimisation levels — reuses RBP as general-purpose register — breaks stack trace profilers

> [!NOTE]
> Compile with `-fno-omit-frame-pointer` to preserve frame pointers for stack profiling. The performance cost is typically <1% but the observability benefit is enormous — lost performance wins from broken stacks far outweigh the initial gain.

> [!TIP]
> Also compile with `-g` to include debuginfo. Can be stripped later for distribution while maintaining a separate debuginfo package.

### Interpreted Languages

- Program translated to actions at runtime — execution overhead — not selected for high performance
- Performance analysis difficulty — CPU profiling shows interpreter internals, not original program function names
- Program context may be available via dynamic instrumentation of interpreter function arguments
- Alternative — examine process memory with knowledge of program layout (`process_vm_readv(2)`)

### Virtual Machines

- Bytecode compiled from source — executed by VM — enables portability
- Java HotSpot — supports interpretation and JIT compilation — JIT compiles bytecode to machine code
- Most difficult to observe — multiple compilation/interpretation stages between source and CPU execution
- Analysis focuses on — VM-provided toolset, USDT probes, third-party tools

### Garbage Collection

- Automatic memory management — no explicit `free()` needed
- Disadvantages —

| Issue | Description |
|-------|-------------|
| Memory growth | Less control of application memory — may grow until hitting limits or causing swapping |
| CPU cost | GC runs intermittently — searches/scans objects — may consume entire CPU as memory grows |
| Latency outliers | Application paused during stop-the-world GC — intermittent high-latency responses |

- GC types — stop-the-world, incremental, concurrent — different latency impact
- Tuning — Java VM provides GC type, thread count, max heap size, target heap free ratio
- If tuning insufficient — application creating too much garbage or leaking references — use allocation tracing tools to find elimination targets

## Methodology

### CPU Profiling

- Essential activity — use kernel-based profilers (`perf(1)`, `profile(8)`) not user-based profilers
- User-based profilers — cannot show kernel CPU usage — may have skewed notion of CPU time
- Sample-based — typical Netflix profile: 49 Hz × 32 CPUs × 30s = 47,040 samples

#### CPU Flame Graphs

- Rectangle = stack frame — y-axis = code flow (top = current function, down = ancestry)
- Width = proportion of profile — x-axis = alphabetical sort (no time ordering)
- Look for wide plateaus and towers — biggest CPU consumers
- Search terms for footprints of off-CPU activity —

| Search Term | Indicates |
|-------------|-----------|
| `ext4`, `btrfs`, `xfs`, `zfs` | File system operations |
| `blk` | Block I/O |
| `tcp` | Network I/O |
| `utex` | Lock contention (`mutex` or `futex`) |
| `alloc`, `object` | Memory allocation code paths |

```bash
# Generate CPU flame graph
perf record -F 49 -a -g -- sleep 10; perf script --header > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-perf.pl < ../out.stacks | ./flamegraph.pl --hash > out.svg
```

### Off-CPU Analysis

- Study of threads not running on CPU — blocked on disk I/O, network I/O, locks, sleeps, scheduler preemption
- Scheduler tracing — instruments kernel CPU scheduler to time off-CPU duration — records stack trace once per blocking event
- Stack trace read once — thread not running so stack does not change

#### Off-CPU Profiling Challenges

- Sampling all threads — system may have 10,000 threads vs 8 CPUs — 1000x more overhead than CPU profiling
- Scheduler events — same system may have 100,000+ scheduler events/second
- Optimisation — record only off-CPU events exceeding minimum duration (1 μs default) — aggregate stacks in kernel via BPF

#### Off-CPU Time Flame Graphs

- Background colour blue — visual distinction from CPU flame graphs (yellow)
- Dominated by wait time (threads waiting for work) — filter by application request handling function
- Linux filter — match on `TASK_UNINTERRUPTIBLE` to focus on interesting off-CPU events

```bash
offcputime -f 5 | ./flamegraph.pl --bgcolors=blue \
    --title="Off-CPU Time Flame Graph" > out.svg
```

- Use `-M 900000` to exclude durations longer than 900 ms — filters out uninteresting idle waits
- Combine with CPU flame graph for complete view of all thread time

### Syscall Analysis

- Targets —

| Target | Tool | Description |
|--------|------|-------------|
| New process tracing | `execsnoop(8)` | Trace `execve(2)` — find short-lived processes consuming resources |
| I/O profiling | `bpftrace` | Trace read/write/send/recv — identify suboptimal I/O (Eg - small I/O) |
| Kernel time analysis | `syscount(8)` | Locate cause of high `%sys` CPU time |

- Syscalls called synchronously with application — stack traces show responsible code path
- Well-documented API (man pages) — easy event source to study

### USE Method

- Apply to software resources as well as hardware
- Eg - worker thread pool —
    - Utilisation — average threads busy as percentage of total
    - Saturation — average request queue length
    - Errors — requests denied or failed
- Eg - file descriptors —
    - Utilisation — in-use FDs as percentage of limit
    - Saturation — threads blocked waiting for FDs
    - Errors — `EFILE` "Too many open files"

### Thread State Analysis

#### Nine-State Model

| State | Description |
|-------|-------------|
| User | On-CPU in user mode |
| Kernel | On-CPU in kernel mode |
| Runnable | Off-CPU waiting for turn on CPU |
| Swapping | Runnable but blocked for anonymous page-ins |
| Disk I/O | Waiting for block device I/O — reads/writes, data/text page-ins |
| Net I/O | Waiting for network I/O — socket reads/writes |
| Sleeping | Voluntary sleep |
| Lock | Waiting to acquire synchronisation lock |
| Idle | Waiting for work |

- Performance improved by reducing time in all states except idle

#### Follow-Up Per State

| State | Investigation |
|-------|--------------|
| User / Kernel | CPU profiling — identify hot code paths including spin locks |
| Runnable | Check system CPU load and CPU resource controls |
| Swapping | Check system memory usage and memory limits |
| Disk | Syscall analysis, file system chapter, disk chapter — check file names, I/O sizes, types |
| Network | Syscall analysis, bpftrace, network chapter — check hostnames, protocols, throughput |
| Sleeping | Analyse reason (code path) and sleep duration |
| Lock | Identify lock, holder, reason holder held lock so long |
| Idle | Often misidentified — network I/O wait for next request or condition variable wait |

#### Linux Thread State Measurement

| State | Tool / Source |
|-------|--------------|
| User | `pidstat(1)` `%usr`, `/proc/PID/stat`, `getrusage(2)` |
| Kernel | `pidstat(1)` `%system`, `/proc/PID/stat` |
| Runnable | `/proc/PID/schedstat` — `perf sched`, `runqlat(8)` BCC |
| Swapping | Delay accounting — `getdelays.c` |
| Disk | `pidstat -d` `iodelay` — `biotop(8)` BCC |
| Network | `tcptop(8)` BCC, `bpftrace` |
| Sleeping | `syscalls:sys_enter_nanosleep` tracepoint — `naptime.bt` |
| Lock | `klockstat(8)` BCC — `pmlock.bt`, `pmheld.bt` (pthread) — `mlock.bt`, `mheld.bt` (kernel) |

> [!NOTE]
> Linux kernel thread state only provides three states via `ps`/`top`: R (`TASK_RUNNING`), D (`TASK_UNINTERRUPTIBLE`), S (`TASK_INTERRUPTIBLE`). The nine-state model requires additional tools.

### Lock Analysis

- Multithreaded — locks can become bottleneck inhibiting parallelism
- Single-threaded — can still be inhibited by kernel locks (Eg - file system locks)
- Check for — contention (current problem) and excessive hold times (future scalability risk)
- Identify — lock name (if it exists) and code path that led to using it
- Spin locks — contention shows up as CPU usage — identify via CPU profiling
- Adaptive mutex — some spinning shows in CPU profile — but blocks/sleeps are invisible to CPU profile

### Static Performance Tuning

- Configuration checklist —
    - Application version and dependencies — newer versions with performance improvements
    - Known performance issues in bug database
    - How application is configured — defaults changed and why
    - Cache object sizing
    - Concurrency configuration — thread pool sizing
    - Debug mode vs release build
    - System library versions
    - Memory allocator in use
    - Large pages configured for heap
    - Compiler, version, options, 64-bit
    - Native code advanced instructions (SIMD/SSE)
    - Error state causing degraded mode
    - System-imposed limits — CPU, memory, file system, disk, network (common in cloud)

### Distributed Tracing

- Traces requests across multiple services — breaks down latency by service
- Collected information — external request ID, dependency hierarchy location, start/end times, error status
- Head-based sampling — sampling decision at start of request — Eg - 1 in 10,000 — sufficient for bulk analysis, misses intermittent errors
- Tail-based sampling — capture all events, decide what to keep based on latency and errors
- Once problematic service identified — use other methodologies for deeper analysis

## Observability Tools

## perf

- Standard Linux profiler — multi-tool

### CPU Profiling

```bash
# Sample stack traces at 49 Hz across all CPUs for 30 seconds
perf record -F 49 -a -g -- sleep 30
perf script    # print each stack sample

# CPU flame graph
perf record -F 49 -a -g -- sleep 10; perf script --header > out.stacks
./stackcollapse-perf.pl < ../out.stacks | ./flamegraph.pl --hash > out.svg
# --color=java for Java applications
```

### Syscall Tracing (perf trace)

```bash
perf trace -p $(pgrep mysqld)         # trace all syscalls for process
perf trace -s -p $(pgrep mysqld)      # summary: counts and timing per syscall
perf trace -e sendto -p $(pgrep mysqld)  # filter to single syscall type
```

- Advantages over `strace` — per-CPU buffers (lower overhead), system-wide tracing, traces non-syscall events
- Disadvantage vs `strace` — fewer syscall argument translations (work in progress via BPF)

## profile

- BCC tool — timer-based CPU profiler — aggregates stacks in kernel via BPF — only unique stacks and counts sent to user space

```bash
profile -F 49 10        # 49 Hz, 10 seconds, all CPUs
profile -F 49 -p PID 10 # single process
```

- Output — unique stack traces with on-CPU sample counts
- Lowest overhead CPU profiler for BPF-based tooling

## offcputime

- BCC and bpftrace tool — summarises time spent blocked off-CPU with stack traces
- Counterpart to `profile(8)` — together they cover all thread time

```bash
offcputime 5                    # all threads, 5 seconds
offcputime -m 1000 5            # minimum off-CPU duration 1000 μs
offcputime -M 900000 5          # exclude durations > 900 ms (filters idle waits)
offcputime -p PID 5             # single process

# Off-CPU flame graph
offcputime -f 5 | ./flamegraph.pl --bgcolors=blue \
    --title="Off-CPU Time Flame Graph" > out.svg
```

- Aggregates in kernel — emits only unique stacks — threshold filtering via `-m`
- Caution — production overhead can be significant — evaluate in test environment first

## strace

- Linux system call tracer — ptrace-based

```bash
strace -ttt -T -p PID       # trace syscalls with timestamps and durations
strace -c command            # summary: syscall counts and timing
strace -f -p PID             # follow child threads
strace -o file.txt -p PID    # write output to file
```

| Option | Description |
|--------|-------------|
| `-ttt` | Time-since-epoch with microsecond resolution |
| `-T` | Duration of each syscall (rightmost column) |
| `-p PID` | Attach to process ID |
| `-c` | Summary — count and time syscalls |
| `-f` | Follow child threads |
| `-e syscall` | Filter to specific syscall |

> [!NOTE]
> strace uses ptrace — sets breakpoints for every syscall entry and return. High-syscall-rate applications may be slowed by 73x or more. Use only for short durations when other methods are unavailable. Prefer `perf trace`, BCC, or bpftrace for production use.

## execsnoop

- BCC and bpftrace tool — traces new process execution system-wide via `execve(2)`

```bash
execsnoop       # trace all new process executions
execsnoop -t    # include timestamps
```

| Column | Description |
|--------|-------------|
| `PCOMM` | Parent process name |
| `PID` | New process ID |
| `PPID` | Parent process ID |
| `RET` | Return value (0 = success) |
| `ARGS` | Full command and arguments |

- Use cases — find short-lived processes consuming resources, debug application start scripts, discover unexpected process execution

## syscount

- BCC and bpftrace tool — counts system calls system-wide

```bash
syscount           # top 10 syscalls by count
syscount -P        # count by process
syscount -i 5      # interval mode, 5 seconds
```

- Use to identify most frequent syscalls — then drill into specific syscall with `perf trace -e` or `bpftrace`

## bpftrace

### Signal Tracing

```bash
# Trace kill(2) signals — source PID/name, destination PID, signal number
bpftrace -e 't:syscalls:sys_enter_kill { time("%H:%M:%S ");
    printf("%s (PID %d) send a SIG %d to PID %d\n",
    comm, pid, args->sig, args->pid); }'
```

- Use for debugging early terminations, unexpected application behaviour

### I/O Profiling

```bash
# recvfrom(2) request buffer size histogram
bpftrace -e 't:syscalls:sys_enter_recvfrom { @bytes = hist(args->size); }'

# recvfrom(2) actual bytes received histogram
bpftrace -e 't:syscalls:sys_exit_recvfrom { @bytes = hist(args->ret); }'

# recvfrom(2) latency histogram
bpftrace -e 't:syscalls:sys_enter_recvfrom { @ts[tid] = nsecs; }
    t:syscalls:sys_exit_recvfrom /@ts[tid]/ {
    @usecs = hist((nsecs - @ts[tid]) / 1000); delete(@ts[tid]); }'

# Filter to specific process
bpftrace -e 't:syscalls:sys_enter_recvfrom /comm == "mysqld"/ { ... }'
```

- Break down by return value — `@usecs[args->ret]` — confirms if larger I/O causes higher latency
- Break down by user stack — `@usecs[ustack]` — shows code path per latency class

### Lock Tracing

```bash
# pthread mutex lock acquisition latency by stack
pmlock.bt $(pgrep mysqld)

# pthread mutex hold time by stack
pmheld.bt $(pgrep mysqld)
```

- `pmlock.bt` — traces `pthread_mutex_lock()` duration — shows acquisition latency per code path
- `pmheld.bt` — traces lock to unlock duration — shows holder code path and hold time
- Lock address printed (or symbol name if available) — identify lock from stack trace in source code

> [!NOTE]
> Lock tracing adds overhead — lock events can be very frequent. CPU profiling is often a lower-overhead alternative — spin lock contention shows as CPU usage, adaptive mutex partially shows spinning.

### Application Internals

```bash
# Check for USDT probes
bpftrace -l 'usdt:/path/to/binary:*'

# List uprobe targets
bpftrace -l 'uprobe:/path/to/binary:*'
```

- USDT probes — stable interface, prefer over uprobes when available
- uprobes — instrument any function in binary or library — C++ code, JVM runtime, OS libraries
- Dynamic USDT — necessary for insight into JIT-compiled software (Eg - Java methods)

## Gotchas

### Missing Symbols

#### ELF Binaries (C, C++, ...)

- Cause — binary stripped of symbol table to reduce file size
- Fix — adjust build to avoid stripping, or use debuginfo / BTF as alternate symbol source
- `perf(1)`, BCC, bpftrace all support debuginfo symbols

#### JIT Runtimes (Java, Node.js, ...)

- Cause — JIT compiler has its own runtime symbol table not in pre-compiled ELF symbols
- Fix — supplemental symbol tables in `/tmp/perf-<PID>.map` — read by `perf(1)` and BCC
- Java fix — `perf-map-agent` dumps live Java process symbols — use `jmaps` immediately after profile

```bash
# Java CPU flame graph with symbols
perf record -F 49 -a -g -- sleep 10; jmaps
perf script --header > out.stacks

# bpftrace with jmaps
bpftrace --unsafe -e 'profile:hz:49 { @[ustack] = count(); }
    interval:s:10 { exit(); } END { system("jmaps"); }'
```

- Symbol churn — JIT symbol table changes between profile sample and dump — run `jmaps` immediately after recording to minimise
- async-profiler — alternative approach — calls into JVM's own stack walker

### Missing Stacks

- Cause — frame pointer-based stack walking + binary compiled without frame pointer (`-fomit-frame-pointer` default at `-O2+`)
- RBP register reused as general-purpose — profiler reads random value — gets `[unknown]` or wrong symbol
- Libc compiled without frame pointers — stacks broken through any libc path

#### Fixes

```bash
# C/C++ — recompile with frame pointer
gcc -fno-omit-frame-pointer ...

# Java — enable frame pointer preservation
java -XX:+PreserveFramePointer ...
```

- Performance cost — typically <1% — far outweighed by observability gains
- Alternative stack walkers for `perf(1)` — DWARF-based, ORC, LBR
- DWARF and LBR not yet available from BPF — ORC not yet available for user-level software