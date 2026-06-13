## CPUs

## Terminology

| Term | Definition |
|------|------------|
| Processor | Physical chip containing one or more CPUs implemented as cores or hardware threads |
| Core | Independent CPU instance on a multicore processor — chip-level multiprocessing (CMP) |
| Hardware thread | CPU architecture supporting multiple threads in parallel on a single core (Eg - Intel Hyper-Threading) — simultaneous multithreading (SMT) |
| CPU instruction | Single CPU operation from its instruction set |
| Logical CPU (virtual processor) | OS CPU instance — schedulable CPU entity — may be a hardware thread, core, or single-core processor |
| Scheduler | Kernel subsystem that assigns threads to run on CPUs |
| Run queue | Queue of runnable threads waiting to be serviced by CPUs — modern kernels may use other data structures (Eg - red-black tree) |

## Models

### CPU Architecture

- Each hardware thread addressable as a logical CPU — Eg - single processor with 4 cores and 8 hardware threads appears as 8 CPUs to OS
- OS may have additional knowledge of topology — which CPUs share a core, how CPU caches are shared — improves scheduling decisions

### CPU Memory Caches

- Cache sizes — smaller and faster the closer to CPU — larger and slower further away
- Trade-off between size and speed at each cache level

### CPU Run Queues

- Per-CPU run queues — threads more likely to stay on same CPU — CPU cache warmth (CPU affinity)
- NUMA systems — per-CPU run queues improve memory locality
- Avoids thread synchronisation overhead of a global shared queue — improves scalability
- Scheduler latency — time spent waiting on run queue — also called run-queue latency or dispatcher-queue latency

## Concepts

### Clock Rate

- Digital signal driving all processor logic — measured in GHz — Eg - 4 GHz = 4 billion cycles/second
- Variable clock rate — processors can increase for performance (turbo boost) or decrease to save power
- Faster clock rate does not always improve performance — if CPUs are mostly stalled on memory, faster clock means more stall cycles not more instructions

### Instructions

- Steps per instruction — instruction fetch, decode, execute, memory access (optional), register write-back (optional)
- Memory access is slowest step — can take dozens of cycles — called stall cycles
- CPU caching dramatically reduces stall cycles

### Instruction Pipeline

- Executes multiple instructions in parallel using different functional units simultaneously
- Ideal — complete one instruction per clock cycle
- Branch prediction — processor guesses outcome of conditional branches — begins processing outcome instructions — wrong guess discards pipeline progress (hurts performance)
- `likely()` and `unlikely()` macros in Linux kernel improve branch prediction accuracy

### Instruction Width (Superscalar)

- Multiple functional units of same type — even more instructions per cycle
- Modern processors — 3-wide or 4-wide — up to 3 or 4 instructions per cycle

### SMT (Simultaneous Multithreading)

- Uses superscalar architecture and hardware multithreading — core runs more than one thread
- Switches between threads during instruction stalls (Eg - memory I/O)
- Kernel presents hardware threads as virtual CPUs — schedules normally
- Intel Hyper-Threading — typically 2 hardware threads per core
- POWER8 — 8 hardware threads per core
- Stall-heavy workloads (low IPC) benefit more from SMT — stall cycles reduce core contention

### IPC and CPI

- __IPC (Instructions Per Cycle)__ — high-level metric for CPU cycle efficiency
- __CPI (Cycles Per Instruction)__ — inverse of IPC
- Low IPC — CPUs often stalled (memory-bound) — Eg - < 0.2 on Netflix cloud
- High IPC — CPUs rarely stalled, high instruction throughput — Eg - > 1.0 (possible due to pipelining)
- Netflix range — 0.2 (slow) to 1.5 (good)
- IPC shows efficiency of instruction processing — not efficiency of the instructions themselves

### Utilization

- Time CPU is not in idle thread — running user/kernel threads or processing interrupts
- High utilisation not necessarily a problem — may indicate system doing work (good ROI)
- Performance does not degrade steeply under high utilisation — kernel supports priorities, preemption, time sharing
- Includes memory stall cycles — CPU can be highly utilised but mostly stalled on memory (Netflix case)

### User Time / Kernel Time

- __User time__ — executing user-level software
- __Kernel time__ — system calls, kernel threads, interrupts

| Workload Type | Typical User/Kernel Ratio |
|--------------|--------------------------|
| Computation-intensive (ML, genomics, image processing) | ~99/1 |
| I/O-intensive (web server, network I/O) | ~70/30 |

### Saturation

- CPU at 100% utilisation — threads encounter scheduler latency waiting on run queue
- CPU resource controls — imposed limit reached before 100% physical utilisation — threads still wait
- Less severe than other resource types — higher-priority work can preempt current thread

### Priority Inversion

- Lower-priority thread holds a resource — blocks higher-priority thread
- Example — thread A (low priority, monitoring) holds lock — thread B (medium) preempts A — thread C (high priority) needs the lock A holds — C blocks — B runs — C effectively blocked by B
- Solution — priority inheritance — thread A inherits thread C's high priority until lock is released
- Linux 2.6.18+ — user-level mutex with priority inheritance for real-time workloads

### Multiprocess vs Multithreading

| Attribute | Multiprocess | Multithreading |
|-----------|-------------|----------------|
| Development | Easier — `fork(2)`/`clone(2)` | Threads API (pthreads) |
| Memory overhead | Separate address space per process | Small — extra stack, registers, thread-local data |
| CPU overhead | `fork(2)`/`clone(2)`/`exit(2)` — MMU work | Small — API calls |
| Communication | IPC — context switch overhead | Direct shared memory — fastest |
| Crash resilience | High — processes are independent | Low — any bug can crash entire application |

- Multithreading generally superior — more efficient and faster communication
- Application must create enough threads to span desired CPU count for maximum parallelism

## Architecture

### Hardware

#### Processor Components

- Control unit — instruction fetch, decode, managing execution, storing results
- P-cache — prefetch cache (per core)
- W-cache — write cache (per core)
- Timestamp counter — high-resolution time
- Microcode ROM — quickly converts instructions to circuit signals
- Temperature sensors — input for dynamic overclocking (Intel Turbo Boost)

#### P-States and C-States

| State Type | Description |
|-----------|-------------|
| P-states | Performance states — vary CPU frequency — P0 = highest (turbo boost), P1..N = lower |
| C0 | Normal execution — CPU fully on |
| C1 | Halt via `hlt` instruction — caches maintained — lowest wakeup latency |
| C1E | Enhanced halt — lower power |
| C2 | Deeper sleep — higher wakeup latency |
| C3 | Deeper sleep — caches may stop snooping — OS defers coherency |

#### CPU Caches

| Cache | Notes |
|-------|-------|
| L1 instruction cache (I$) | Per core — fastest and smallest |
| L1 data cache (D$) | Per core |
| TLB | Caches address translations |
| L2 cache (E$) | Embedded cache — per core |
| L3 cache | Shared across cores — last-level cache (LLC) |

- Cache line size — 64 bytes on x86 — unit of storage and transfer
- Cache latency — L1: few cycles, L2: ~dozen cycles, DRAM: ~60 ns (~240 cycles at 4 GHz)
- Cache associativity — fully associative (expensive), direct mapped (poor hit rates), set associative (balance)

#### Cache Coherency

- When one CPU modifies memory — all caches with cached copy must discard stale data
- LLC hit latency examples (rough guide) —
    - Unshared line: ~40 cycles
    - Shared in another core: ~65 cycles
    - Modified in another core: ~75 cycles

#### MMU

- Responsible for virtual-to-physical address translation
- TLB — on-chip cache for address translations — misses satisfied by page tables in DRAM
- Modern processors — hardware TLB miss handling (greatly reduces cost vs software)

#### Interconnects

| Interconnect | Transfer Rate | Bandwidth |
|-------------|--------------|-----------|
| Intel FSB (2007) | 1.6 GT/s | 12.8 GB/s |
| Intel QPI (2008) | 6.4 GT/s | 25.6 GB/s |
| Intel UPI (2017) | 10.4 GT/s | 41.6 GB/s |

- Bottleneck in interconnect — causes stall cycles for remote memory I/O — key indicator is IPC drop
- AMD HyperTransport (HT), ARM CoreLink, IBM CAPI — other interconnect examples
- QPI cache coherency modes — Home Snoop (bandwidth), Early Snoop (latency), Directory Snoop (scalability — UPI default)

#### Performance Monitoring Counters (PMCs)

- Processor registers to count low-level CPU activity
- Typically 2–8 registers per CPU — programmable via event-select MSRs
- Intel Skylake — 3 fixed counters per hardware thread + 8 programmable counters per core

| PMC Category | Examples |
|-------------|---------|
| CPU cycles | Stall cycles, types of stall cycles |
| CPU instructions | Retired (executed) |
| Cache | L1/L2/L3 hits and misses |
| FPU | Operations |
| Memory I/O | Reads, writes, stall cycles |
| Branches | Retired, mispredicted |

- PEBS (Precise Event-Based Sampling) — Intel solution for accurate instruction attribution during overflow sampling (avoids skid)

#### GPUs

| Attribute | CPU | GPU |
|-----------|-----|-----|
| Cores | 2–64 typical | Hundreds/thousands of streaming processors (SPs) |
| Threads per core | 2 hardware threads (SMT) | Each SP executes one thread — grouped into thread blocks |
| Clock | High (~3.4 GHz) | Lower (~1.0 GHz) |
| Caches | L1/L2 per core, shared L3 | Cache per SM, shared L2 |
| Connection | Direct to CPU bus/interconnect | Expansion bus (PCIe) |

- CUDA (Nvidia) — widely adopted for general-purpose GPU compute
- GPU profiling differs from CPU — no stack traces — instrument API and memory transfer calls

### Software

#### Kernel CPU Scheduler

- Time sharing — multitasking between runnable threads — highest priority first
- Preemption — higher-priority threads can interrupt currently running thread immediately
- Load balancing — moves runnable threads to idle or less-busy CPUs
- CFS uses red-black tree keyed by task CPU time — not traditional run queues
- CPU affinity preference — avoids migrations when cost exceeds benefit (`task_hot()` check)

#### Scheduling Classes (Linux)

| Class | Priority Range | Description |
|-------|---------------|-------------|
| RT | 0–99 | Fixed high priorities for real-time — kernel and user preemption |
| O(1) | User (deprecated) | O(1) algorithm — dynamic I/O-bound priority boost |
| CFS | User (default) | Completely fair scheduling — red-black tree — favours low CPU consumers |
| Deadline | — | Earliest deadline first (EDF) — runtime/period/deadline parameters |
| Idle | Lowest | Placeholder when no work |

#### Scheduling Policies

| Policy | Class | Description |
|--------|-------|-------------|
| `SCHED_RR` | RT | Round-robin — moves to end of queue after time quantum |
| `SCHED_FIFO` | RT | First-in first-out — runs until voluntary exit or higher priority arrives |
| `SCHED_NORMAL` | CFS | Default time-sharing — dynamic priority adjustment |
| `SCHED_BATCH` | CFS | CPU-bound expectation — does not interrupt I/O-bound work |
| `SCHED_IDLE` | Idle | Lowest possible priority |
| `SCHED_DEADLINE` | Deadline | EDF with runtime/period/deadline parameters |

#### NUMA Grouping

- Scheduling domains — NUMA-aware topology from root domain
- Estimates cost of any memory access based on topology
- Manual grouping — bind processes to CPUs or create exclusive CPU sets

## Methodology

### Tools Method

```bash
uptime/top       # check load averages — increasing or decreasing
vmstat 1         # system-wide CPU utilisation — us+sy near 100% → scheduler latency likely
mpstat -P ALL 1  # per-CPU stats — identify hot CPUs (scalability problem indicator)
top              # top CPU-consuming processes and users
pidstat 1        # per-process user/system time breakdown
perf/profile     # profile CPU stack traces for user and kernel time
perf stat -a     # measure IPC — indicator of cycle-based inefficiencies
showboost/turboboost  # check current CPU clock rates
dmesg            # check for "cpu clock throttled" messages
```

### USE Method

- Per CPU —
    - __Utilisation__ — time CPU was busy (not in idle thread)
    - __Saturation__ — degree to which runnable threads queue waiting for CPU
    - __Errors__ — correctable ECC errors — some kernels offline a CPU on increase as precaution

- Check errors first — quick and objective
- Check utilisation per CPU — single hot CPU while others idle = thread scalability problem
- For cloud environments — also measure utilisation vs CPU quota limit (saturation may occur before 100% physical)
- Saturation commonly provided as load averages — system-wide measure of overload

### Workload Characterisation

- Basic attributes —
    - CPU load averages (utilisation + saturation)
    - User-time to system-time ratio
    - Syscall rate
    - Voluntary context switch rate
    - Interrupt rate
- High user-time — compute-intensive — application doing its own work
- High system-time — I/O-intensive — more kernel involvement
- I/O-bound workloads — higher system time, higher syscalls, higher voluntary context switches

### Profiling

- Timer-based sampling — 99 Hz typical (not 100 Hz to avoid lock-step sampling) — low overhead — preferred for production
- Function tracing — all function calls — 10%+ overhead — not for production
- Profile both user and kernel stacks for complete CPU profile
- Typical Netflix profile — 49 Hz × 32 CPUs × 30s = 47,040 samples

#### CPU Flame Graph Interpretation

- Look top-down (leaf to root) for large plateaus — single function on-CPU many samples — quick wins
- Look bottom-up to understand code hierarchy — may reveal even bigger wins higher up
- Look for scattered common patterns — Eg - lock contention showing as many small frames
- Search for code areas — `tcp_` for TCP, `blk_` for block I/O, `utex` for lock contention, `ext4` for filesystem
- Icicle graphs (inverted flame graphs) — help reveal common leaf functions scattered across many stacks

### Cycle Analysis

- Use PMCs to understand CPU utilisation at cycle level
- Begin with IPC — low IPC → investigate stall cycle types — high IPC → look to reduce instructions
- PMC-based profiling — interrupt on counter overflow (Eg - every 10,000 LLC misses) — collect stack trace — builds profile of LLC miss code paths without measuring every miss
- Useful for identifying which code paths cause L1/L2/L3 cache misses, TLB misses, memory stalls
- Tools — Intel vTune, AMD uprof — save time vs manual PMC analysis with command-line tools

### Static Performance Tuning

- Configuration checklist —
    - Number and type of CPUs — cores, hardware threads
    - GPUs or accelerators available and in use
    - CPU cache sizes and sharing
    - Clock speed — dynamic features (Turbo Boost, SpeedStep) enabled in BIOS
    - Other BIOS settings — turboboost, power saving, bus settings
    - Processor errata — known performance bugs
    - Microcode version — Spectre/Meltdown mitigation performance impact
    - BIOS firmware performance bugs
    - Software-imposed CPU limits (resource controls) — especially in cloud

### Priority Tuning

- Use `nice` for low-priority background work — monitoring agents, scheduled backups
- Positive nice values (1–19) — lower priority — can be set by any user
- Negative nice values (-1 to -20) — higher priority — requires root (or `RLIMIT_NICE` capability)
- Verify tuning is effective — check scheduler latency remains low for high-priority work
- Real-time scheduling class (`SCHED_RR`/`SCHED_FIFO`) — eliminates scheduler latency for RT work — risk: infinite loop RT thread starves all other work
- Linux 2.6.25+ — `RLIMIT_RTTIME` limits CPU time for real-time threads before forcing a blocking syscall

### CPU Binding

- CPU binding — process runs on single CPU or defined set — improves cache warmth and memory locality
- Exclusive CPU sets — other processes cannot use bound CPUs — further improves cache warmth — reduces available CPU for rest of system
- Linux — `taskset(1)` for CPU binding — `cpusets` for exclusive CPU sets
- NUMA — `numactl(8)` for CPU and memory node binding together

## Observability Tools

## uptime

```bash
uptime    # prints 1-, 5-, 15-minute load averages
```

- Load averages — demand for system resources — CPUs plus disks and other resources (Linux since 1993)
- Compare 1 vs 15 minute — load increasing (issue getting worse) or decreasing (may have missed it)
- Load = utilisation (busy) + saturation (queued) — Eg - 64-CPU system, load average 128 = 1 thread running + 1 queued per CPU

#### Load Averages vs PSI

| Attribute | Load Averages | PSI |
|-----------|--------------|-----|
| Resources | System-wide (CPU + disk + memory) | cpu, memory, io separately |
| Metric | Number of busy and queued tasks | Percent of time stalled (saturation only) |
| Times | 1 min, 5 min, 15 min | 10 s, 60 s, 300 s |

```bash
cat /proc/pressure/cpu
# some avg10=50.00 avg60=50.00 avg300=49.70 total=...
```

- `some` line — at least one task stalled — `full` line (memory/io only) — all runnable tasks stalled

#### PSI vs Load Average Examples (2 CPUs)

| Scenario | Load Average | PSI CPU |
|----------|-------------|---------|
| 1 busy thread | 1.0 | 0.0 |
| 2 busy threads | 2.0 | 0.0 |
| 3 busy threads | 3.0 | 50.0 |
| 4 busy threads | 4.0 | 100.0 |

## vmstat

```bash
vmstat 1    # 1-second interval
```

| Column | Description |
|--------|-------------|
| `r` | Run-queue length — total runnable threads (waiting + running) |
| `us` | User-time percent |
| `sy` | System-time (kernel) percent |
| `id` | Idle percent |
| `wa` | Wait I/O percent — CPU idle with threads blocked on disk I/O |
| `st` | Stolen percent — time servicing other VM tenants |

- All values system-wide averages across all CPUs except `r` (total count)

## mpstat

```bash
mpstat -P ALL 1    # per-CPU statistics, 1-second interval
```

| Column | Description |
|--------|-------------|
| `%usr` | User-time (excluding nice) |
| `%sys` | System-time (kernel) |
| `%iowait` | I/O wait |
| `%irq` | Hardware interrupt CPU usage |
| `%soft` | Software interrupt CPU usage |
| `%steal` | Time servicing other VM tenants |
| `%idle` | Idle |

- Key columns — `%usr`, `%sys`, `%idle` — shows user/kernel ratio and identifies hot CPUs

## sar

```bash
sar -P ALL 1    # per-CPU (same as mpstat -P ALL)
sar -u 1        # system-wide CPU average
sar -q 1        # run-queue size and load averages
```

## ps

```bash
ps aux         # BSD-style — all users, extended, including no-terminal processes
ps -ef         # SVR4-style — every process, full details
```

- `TIME` column — total CPU time (user + system) since creation — `HH:MM:SS`
- `%CPU` on Linux — average over process lifetime, NOT normalised by CPU count — 200% for 2-thread CPU-bound process

## top

```bash
top                # default — sorted by CPU usage, updates every 3s
top -d 1           # 1-second refresh
```

- `%CPU` — total CPU utilisation for current update interval — NOT normalised (Irix mode default)
- Press `I` to toggle Solaris mode — normalises by CPU count (max 100%)
- `TIME+` — hundredths of a second resolution — Eg - `1:36.53` = 1 min 36.53 sec on-CPU
- `atop` — uses process accounting to catch short-lived processes missed by top snapshots

## pidstat

```bash
pidstat 1         # rolling output of active processes, 1-second interval
pidstat -p ALL 1  # include idle processes
pidstat -t 1      # per-thread statistics
```

| Column | Description |
|--------|-------------|
| `%usr` | User-time |
| `%system` | System-time (kernel) |
| `%guest` | Time in guest VMs |
| `%CPU` | Total CPU (usr + system + guest) |
| `CPU` | Logical CPU ID the process last ran on |

## time

```bash
time command           # shell built-in — real, user, sys
/usr/bin/time -v command  # verbose — includes context switches, page faults, max RSS
```

- `real` — wall clock time
- `user` — user-mode CPU time
- `sys` — kernel-mode CPU time
- Missing time (real - user - sys) — likely time blocked on I/O

## turbostat

```bash
turbostat    # MSR-based tool — CPU state, clock rates, C-states, power, temperature
```

- Shows `Avg_MHz`, `Busy%`, `Bzy_MHz` (actual frequency when busy), `TSC_MHz`, C-state percentages, temperatures, power
- Verbose output — may be 300+ characters wide

## showboost

```bash
showboost    # current CPU clock rate with turbo boost status per interval
```

- Shows base MHz, set MHz, turbo MHz steps, current utilisation ratio, and actual MHz per interval

## pmcarch

```bash
pmcarch    # high-level PMC-based CPU cycle breakdown
```

| Column | Description |
|--------|-------------|
| `K_CYCLES` | CPU cycles × 1000 |
| `K_INSTR` | CPU instructions × 1000 |
| `IPC` | Instructions per cycle |
| `BMR%` | Branch misprediction ratio (%) |
| `LLC%` | Last level cache hit ratio (%) |

- Uses Intel architectural PMC set — available on some cloud environments (Eg - AWS EC2)

## tlbstat

```bash
tlbstat -C0 1    # TLB statistics for CPU 0, 1-second interval
```

| Column | Description |
|--------|-------------|
| `DTLB_WALKS` | Data TLB walks count |
| `ITLB_WALKS` | Instruction TLB walks count |
| `K_DTLBCYC` | Cycles with active data TLB walks × 1000 |
| `K_ITLBCYC` | Cycles with active instruction TLB walks × 1000 |
| `DTLB%` | Data TLB active cycles as ratio of total |
| `ITLB%` | Instruction TLB active cycles as ratio of total |

- KPTI Meltdown mitigation flushes TLBs on syscalls — can cause 50%+ of cycles in TLB walks

## perf

- Official Linux profiler — multi-tool

### CPU Profiling One-Liners

```bash
perf record -F 99 command                                        # sample on-CPU functions at 99 Hz
perf record -F 99 -a -g -- sleep 10                              # system-wide with stacks
perf record -F 99 -p PID --call-graph dwarf -- sleep 10          # per-process with DWARF stack unwinding
perf record -e migrations -a -- sleep 10                         # CPU migrations
perf record -e sched:sched_switch -a -g -- sleep 1               # context switches with stacks
perf report --stdio                                              # text report with counts and percentages
perf report -g callee --stdio                                    # callee ordering (leaf first)
perf script --header                                             # list all events with header
perf stat -a -- sleep 5                                          # system-wide PMC statistics
perf stat -e LLC-loads,LLC-load-misses,LLC-stores command        # LLC cache statistics
perf stat -e L1-dcache-load-misses,LLC-load-misses,dTLB-load-misses command  # cache hierarchy
perf stat -e sched:sched_switch -a -I 1000                       # context switch rate per second
perf sched record -- sleep 10                                    # record scheduler profile
perf sched latency                                               # per-process scheduler latency summary
perf sched timehist                                              # per-event scheduler latency details
```

### CPU Flame Graphs

```bash
# Linux 5.8+ built-in flame graph
perf record -F 99 -a -g -- sleep 10
perf script report flamegraph

# Original approach (older kernels)
perf record -F 99 -a -g -- sleep 10
perf script --header > out.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-perf.pl < ../out.stacks | ./flamegraph.pl --hash > out.svg
```

- flamegraph.pl options — `--title`, `--width`, `--colors` (hot/mem/io/java/js), `--bgcolors` (yellow/blue/green/grey), `--hash` (consistent colors), `--reverse` (leaf-to-root merge), `--inverted` (icicle graph)

### perf stat PMC Output

94,817,146,941      cycles                    #    3.757 GHz

152,114,038,783      instructions             #    1.60  insn per cycle

28,974,755,679      branches                  # 1148.167 M/sec

1,020,287,443      branch-misses             #    3.52% of all branches

- IPC 1.6 = good — IPC 0.7 = likely memory-bound

### Scheduler Latency (perf sched)

```bash
perf sched record -- sleep 10    # record — high overhead, large perf.data file
perf sched latency               # avg and max scheduler latency per process
perf sched timehist              # per-event: wait time, sch delay, run time
```

> [!NOTE]
> `perf sched` generates very large perf.data files and high overhead due to frequent scheduler events. Use cautiously in production.

## profile

- BCC tool — timer-based CPU profiler — aggregates stacks in kernel (lower overhead than perf)
- Only emits unique stacks and counts to user space — no intermediate profile file

```bash
profile                  # 49 Hz, all threads, user + kernel stacks
profile -F 99 10         # 99 Hz, 10 seconds
profile -U               # user-level stacks only
profile -K               # kernel-level stacks only
profile -p PID           # single process
profile -af 10 > out.stacks  # folded format for flame graph generation
./flamegraph.pl --hash < out.stacks > out.svg
```

- Output sorted from least to most frequent — last stack = most frequent
- `WARNING: N stack traces could not be displayed` — increase `--stack-storage-size`

## cpudist

- BCC tool — distribution of on-CPU time per thread wakeup

```bash
cpudist 10 1      # 10-second interval, print once
cpudist -m        # millisecond output
cpudist -O        # off-CPU time instead of on-CPU
cpudist -P        # per-process histograms
cpudist -p PID    # single process
```

## runqlat

- BCC and bpftrace tool — measures CPU scheduler latency (time from wakeup to running)

```bash
runqlat 10 1      # 10-second interval, print once
runqlat -m        # milliseconds
runqlat -P        # per-process histograms
runqlat -p PID    # single process
```

> [!NOTE]
> `runqlat` instruments every context switch — can exceed 1 million events/second on busy systems — significant overhead. Consider `runqlen` as a lighter alternative.

## runqlen

- BCC and bpftrace tool — samples run queue length at 99 Hz — lightweight (no per-event tracing)

```bash
runqlen 10 1    # 10-second interval, print once
runqlen -C      # per-CPU histograms
runqlen -O      # run queue occupancy (% of time with waiting threads)
```

- Occupancy — single metric for monitoring, alerting, and graphing

## softirqs

- BCC tool — time spent servicing soft interrupts by type

```bash
softirqs 10 1    # 10-second interval
softirqs -d      # show IRQ time as histograms
```

- Shows totals in microseconds per soft IRQ type (Eg - `net_rx`, `timer`, `sched`, `rcu`, `tasklet`)
- Reveals CPU consumers not visible in typical CPU profiles (softirqs are non-interruptible by samplers)

## hardirqs

- BCC tool — time spent servicing hard interrupts by type

```bash
hardirqs 10 1    # 10-second interval
hardirqs -d      # show IRQ time as histograms
```

## bpftrace

### One-Liners

```bash
# Trace new processes with arguments
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { join(args->argv); }'

# Count syscalls by process
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[pid, comm] = count(); }'

# Sample running process names at 99 Hz
bpftrace -e 'profile:hz:99 { @[comm] = count(); }'

# Sample user and kernel stacks at 49 Hz system-wide
bpftrace -e 'profile:hz:49 { @[kstack, ustack, comm] = count(); }'

# Sample user stacks for PID 189 at 49 Hz
bpftrace -e 'profile:hz:49 /pid == 189/ { @[ustack] = count(); }'

# Sample user stacks 5 frames deep at 49 Hz for PID 189
bpftrace -e 'profile:hz:49 /pid == 189/ { @[ustack(5)] = count(); }'

# Count off-CPU kernel stacks at context switches
bpftrace -e 'tracepoint:sched:sched_switch { @[kstack] = count(); }'

# Count kernel scheduler tracepoints
bpftrace -e 'tracepoint:sched:* { @[probe] = count(); }'

# Trace new threads via pthread_create()
bpftrace -e 'u:/lib/x86_64-linux-gnu/libpthread-2.27.so:pthread_create {
    printf("%s by %s (%d)\n", probe, comm, pid); }'
```

### Scheduler Tracepoints

```bash
bpftrace -l 'tracepoint:sched:*'    # 24 tracepoints on kernel 5.3
bpftrace -lv 'kprobe:sched*'        # 104 kprobe targets beginning with "sched"
```

> [!NOTE]
> Scheduler events can be very frequent — use maps to summarise rather than per-event output — trace fewest possible events.

## Other Tools

| Tool | Description |
|------|-------------|
| `offcputime` | Off-CPU profiling via scheduler tracing |
| `execsnoop` | New process execution tracing |
| `syscount` | Syscall counts by type and process |
| `runqslower` | Prints run queue waits above a threshold |
| `llcstat` | LLC hit ratio by process |
| `lstopo` | Hardware topology visualisation (hwloc package) |
| `cpupower` | Processor power states and C-state statistics |
| `getdelays.c` | CPU scheduler latency via delay accounting |
| `/proc/cpuinfo` | Processor details, clock speed, feature flags |
| `lscpu` | CPU architecture information |
| `oprofile` | Original Linux CPU profiling tool |
| `atop` | Catches short-lived processes via process accounting |

## Visualisations

### Utilisation Heat Map

- Per-CPU utilisation vs time — saturation (darkness) shows number of CPUs at that utilisation level
- Scales to any CPU count — identifies both idle CPUs and CPUs at 100% simultaneously
- Dark line at top = multiple CPUs consistently at 100%

### Subsecond-Offset Heat Map

- Y-axis = subsecond offset within each second — reveals within-second activity patterns
- Identifies issues invisible in per-second averages — Eg - 100–500 ms gaps where no database threads are on-CPU → discovered locking issue blocking entire database

### CPU Flame Graphs

- Each box = stack frame — y-axis = stack depth — x-axis = alphabetical sort (not time)
- Width = proportion of profile — look for widest towers and plateaus first
- Top edge = functions directly on-CPU
- Interactive — mouse over for details, click to zoom, Ctrl-F to search (shows cumulative %)

#### Flame Graph Color Schemes

| Scheme | Description |
|--------|-------------|
| Default (random warm) | Helps visually distinguish adjacent towers |
| Hue by code type | Red = native user-level, orange = kernel, green = interpreted, aqua = inlined |
| Saturation hashed from name | Consistent colors per function — easier to compare multiple flame graphs |
| Background color | Yellow = CPU, blue = off-CPU/I/O, green = memory |
| IPC gradient | Blue = low IPC → white → red = high IPC |

### FlameScope

- Combines subsecond-offset heat map + flame graphs — Netflix open source tool
- Select time ranges (including subsecond) in heat map — generates flame graph for that range
- Identifies CPU perturbations too small to see in full profile — 100 ms perturbation in 30s profile = 0.3% width in flame graph vs visible vertical stripe in FlameScope

## Tuning

### Compiler Options

- Compile for 64-bit — more registers, efficient register calling convention
- Optimisation levels (`-O2`, `-O3`) — significant performance improvement
- See Chapter 5 Applications for details

### Scheduling Priority

```bash
nice -n 19 command          # run at lowest nice priority (19)
nice -n -20 command         # run at highest nice priority (-20, root only)
renice -n 10 -p PID         # adjust priority of running process
chrt -b command             # run with SCHED_BATCH
chrt -r -p 99 PID           # set process to SCHED_RR priority 99
```

### Linux Scheduler CONFIG Options

| Option | Default | Description |
|--------|---------|-------------|
| `CONFIG_SCHED_SMT` | y | Hyperthreading support |
| `CONFIG_SCHED_MC` | y | Multicore support |
| `CONFIG_HZ` | 250 | Kernel clock rate |
| `CONFIG_NO_HZ` | y | Tickless kernel |
| `CONFIG_PREEMPT_VOLUNTARY` | y | Preemption at voluntary kernel code points |
| `CONFIG_PREEMPT` | n | Full kernel preemption |

### Linux Scheduler sysctl Tunables

| Tunable | Default | Description |
|---------|---------|-------------|
| `kernel.sched_latency_ns` | 12000000 | Targeted preemption latency — increase for longer time slices |
| `kernel.sched_migration_cost_ns` | 500000 | Task migration cost for affinity — tasks run more recently = cache hot |
| `kernel.sched_nr_migrate` | 32 | Max tasks migrated at once during load balancing |
| `kernel.sched_schedstats` | 0 | Enable additional scheduler statistics and tracepoints |

### CPU Scaling Governors

```bash
cat /sys/devices/system/cpu/cpufreq/policy0/scaling_available_governors
echo performance > /sys/devices/system/cpu/cpufreq/policy0/scaling_governor
# Must be done for all CPUs (policy0..N)
```

- `powersave` — uses lower frequencies — default on many systems (untuned)
- `performance` — always uses maximum frequency — better for latency-sensitive workloads

> [!TIP]
> Consider environmental cost before setting all CPUs to `performance`. Quantify improvement before making permanent — use MSRs to measure power with and without.

### Power States

```bash
cpupower idle-info          # list C-states with exit latency and usage statistics
cpupower idle-set -d 4      # disable C-state 4 (index)
cpupower idle-set -D 100    # disable all C-states with exit latency > 100 μs
```

### CPU Binding

```bash
taskset -pc 7-10 PID        # bind PID to CPUs 7-10
numactl --cpubind=0 command  # bind to NUMA node 0 CPUs
```

### Exclusive CPU Sets (cpusets)

```bash
mkdir /sys/fs/cgroup/cpuset/prodset
echo 7-10 > /sys/fs/cgroup/cpuset/prodset/cpuset.cpus
echo 1 > /sys/fs/cgroup/cpuset/prodset/cpuset.cpu_exclusive
echo PID > /sys/fs/cgroup/cpuset/prodset/tasks
```

### Resource Controls (cgroups)

- CPU shares — flexible allocation — idle cycles shared based on share value
- CFS bandwidth — fixed limits — microseconds of CPU per interval
- Use `taskset` and `cgroups` in concert for cloud multi-tenant workloads

### Security Boot Options

- Meltdown/Spectre mitigations reduce performance — disable only when security not required
- Options — `nospectre_v1`, `nospectre_v2` — documented in `Documentation/admin-guide/kernel-parameters.txt`
- Reference — https://make-linux-fast-again.com (lacks safety warnings from kernel docs)

### BIOS/Processor Options

- Intel Turbo Boost — usually enabled by default — disable only for consistent benchmark clock rates
- Verify settings for: power saving modes, bus settings, turbo boost
- Default settings usually provide maximum performance