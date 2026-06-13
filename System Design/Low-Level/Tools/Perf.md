## perf

- Official Linux profiler — in kernel source under `tools/perf`
- Front-end to `perf_events` (Performance Counters for Linux / Linux Performance Events)
- Capabilities — CPU profiling, tracing, PMC analysis, scheduler analysis, scripting

## Subcommands Overview

| Command | Description |
|---------|-------------|
| `annotate` | Display annotated code from perf.data |
| `bench` | System micro-benchmarks |
| `c2c` | Cache-to-cache and cache line false sharing analysis |
| `diff` | Differential profile between two perf.data files |
| `ftrace` | Interface to Ftrace tracer |
| `kmem` | Kernel memory (slab) analysis |
| `kvm` | KVM guest analysis |
| `list` | List event types |
| `lock` | Lock analysis |
| `mem` | Memory access analysis |
| `probe` | Define new dynamic tracepoints |
| `record` | Record events to perf.data |
| `report` | Summarise perf.data |
| `sched` | Scheduler latency and statistics |
| `script` | Print perf.data events or run trace scripts |
| `stat` | Count events — PMC statistics |
| `timechart` | Visualise system behaviour during workload |
| `top` | Real-time profiling (screen updates) |
| `trace` | Live event tracer (syscalls by default) |

## One-Liners

### Listing Events

```bash
perf list                      # all known events
perf list 'sched:*'            # sched tracepoints
perf list block                # events containing "block"
perf probe -l                  # currently available dynamic probes
```

### Counting Events

```bash
perf stat command                                          # PMC stats for command
perf stat -p PID                                          # PMC stats for PID until Ctrl-C
perf stat -a sleep 5                                      # system-wide 5 seconds
perf stat -e LLC-loads,LLC-load-misses,LLC-stores command # LLC cache stats
perf stat -e r003c -a sleep 5                             # raw PMC (unhalted core cycles, Intel)
perf stat -e raw_syscalls:sys_enter -I 1000 -a            # syscalls per second
perf stat -e 'syscalls:sys_enter_*' -p PID                # syscall types for PID
perf stat -e 'block:*' -a sleep 10                        # block device events 10s
```

### Profiling

```bash
perf record -F 99 command                                  # on-CPU functions at 99 Hz
perf record -F 99 -a -g sleep 10                           # system-wide with stacks 10s
perf record -F 99 -p PID --call-graph dwarf sleep 10       # per-process DWARF stack unwinding
perf record -F 99 -a --call-graph lbr sleep 10             # Intel LBR stack walking
perf record -e LLC-load-misses -c 100 -ag sleep 5          # once per 100 LLC misses
perf record -e cycles:up -a sleep 5                        # precise user instruction sampling (PEBS)
perf top -F 49 -ns comm,dso                                # live top process names and segments
```

### Static Tracing

```bash
perf record -e sched:sched_process_exec -a                 # new processes
perf record -e context-switches -a -g sleep 1             # sampled context switches with stacks
perf record -e sched:sched_switch -a -g sleep 1           # all context switches with stacks
perf record -e sched:sched_switch/max-stack=5/ -a sleep 1  # context switches 5-level deep
perf record -e syscalls:sys_enter_connect -a -g            # connect(2) calls with stacks
perf record -F 100 -e block:block_rq_issue -a              # sample block requests at 100 Hz
perf record -e block:block_rq_issue,block:block_rq_complete -a  # all block I/O with timestamps
perf record -e block:block_rq_issue --filter 'bytes >= 65536'   # block I/O >= 64 KB
perf record -e 'ext4:*' -o /tmp/perf.data -a              # all ext4 calls (output to non-ext4)
perf record -e sdt_node:http__server__request -a           # Node.js USDT event
perf trace -e block:block_rq_issue                         # live block I/O (no perf.data)
perf trace                                                 # live syscall trace system-wide
```

### Dynamic Tracing

```bash
perf probe --add tcp_sendmsg                               # add kprobe for tcp_sendmsg()
perf probe --del tcp_sendmsg                               # remove kprobe
perf probe -V tcp_sendmsg --externs                        # list variables (needs debuginfo)
perf probe -L tcp_sendmsg                                  # list available line probes (needs debuginfo)
perf probe 'tcp_sendmsg %ax %dx %cx'                       # probe with register arguments
perf probe 'tcp_sendmsg bytes=%cx'                         # probe with register alias
perf record -e probe:tcp_sendmsg --filter 'bytes > 100'    # trace when bytes > 100
perf probe 'tcp_sendmsg%return $retval'                    # capture return value
perf probe 'tcp_sendmsg size sk->__sk_common.skc_state'    # probe with struct member (debuginfo)
perf probe 'do_sys_open filename:string'                   # filename as string (debuginfo)
perf probe -x /lib/x86_64-linux-gnu/libc.so.6 --add fopen # user-level fopen() uprobe
```

### Reporting

```bash
perf report                                                # TUI browser
perf report -n --stdio                                     # text report with sample counts
perf script --header                                       # all events with metadata header
perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso  # specific fields
perf script report flamegraph                              # flame graph (Linux 5.8+)
perf annotate --stdio                                      # annotated instructions with percentages
```

## Events

### Event Types

| Type | Description |
|------|-------------|
| Hardware event | Processor events — PMC-based (Eg - `branch-instructions`, `cache-misses`) |
| Software event | Kernel counter events (Eg - `context-switches`, `cpu-clock`) |
| Hardware cache event | Processor cache events — PMC-based (Eg - `L1-dcache-load-misses`) |
| Kernel PMU event | Performance Monitoring Unit events (Eg - `cpu/branch-instructions/`) |
| Processor vendor events | Named with descriptions — Intel/AMD/ARM specific |
| Raw hardware event | PMC via raw codes — `rNNN` or `cpu/t1=v1,t2=v2/modifier` |
| Hardware breakpoint | Processor breakpoint — `mem:<addr>[/len][:access]` |
| Tracepoint event | Kernel static instrumentation — also shows initialised kprobes/uprobes |
| SDT event | User-level USDT static instrumentation |
| pfm-events | libpfm events (Linux 5.8+) |

### Hardware Events

- PMC-based — processor-specific register codes (event select + umask)
- Human-readable mappings available — Eg - `branch-instructions` maps to branch instructions PMC
- Raw codes needed for deeper PMCs without mappings — Eg - `r00c4` for Intel branch instructions
- Intel events available as JSON files — processor manuals document full event set

#### Frequency Sampling

- Default sample frequency — 4000 events per second per CPU
- `stat` counts all events — `record` uses frequency sampling by default
- `-F 99` — target 99 Hz sample rate
- `-c N` — overflow sampling — one-in-every-N events (period sampling)
- Tracepoints — default to period=1 (every event) unlike software/hardware events

```bash
sysctl kernel.perf_event_max_sample_rate   # check max sample rate (Eg - 15500 Hz)
sysctl kernel.perf_cpu_time_max_percent    # max CPU % allowed by perf PMU interrupt (Eg - 25%)
```

### Software Events

- Map to hardware events but instrumented in software
- Default to frequency sampling (4000 Hz) unlike tracepoints which capture every event
- Use `-c 1` to capture every context switch software event

### Tracepoints

- Static kernel instrumentation
- Default to period=1 (every event)
- Format and arguments in `/sys/kernel/debug/tracing/events/SUBSYSTEM/EVENT/format`
- Filter using Boolean expressions on event arguments

```bash
perf record -e block:block_rq_issue --filter 'bytes > 65536' -a sleep 10
```

### Probe Events (Dynamic)

- Must be initialised before tracing — not in `perf list` by default
- After initialisation — appear as "Tracepoint event"
- Includes kprobes, uprobes, and USDT probes

#### kprobes

```bash
perf probe --add do_nanosleep              # create kprobe at function entry
perf probe --add do_nanosleep%return       # create kretprobe at function return
perf probe 'do_nanosleep%return $retval'   # capture return value
perf probe --del probe:do_nanosleep        # remove kprobe
```

- With debuginfo — list variables using `perf probe --vars FUNCTION`
- Without debuginfo — use register locations from ABI (Eg - `mode=%si:x32` for x86_64)
- Dry run technique — install debuginfo on reference system, use `-nv` to find register locations

#### uprobes

```bash
perf probe -x /path/to/binary --add function                    # create uprobe
perf probe -x /path/to/binary --add function%return             # create uretprobe
perf probe -x /path/to/binary --vars function                   # list variables (needs debuginfo)
perf probe -x /lib/.../libc.so.6 --add 'fopen filename mode'   # with arguments
perf probe --del probe_libc:fopen                               # remove uprobe
```

- Without debuginfo — use ABI register locations manually
- x86_64 argument registers — `%di` (arg1), `%si` (arg2), `%dx` (arg3), etc. (AMD64 ABI)
- Dereference pointer — `+0(%di)` — print as string with `:string`
- Print as integer — `:u8`, `:x32`, etc.
- Return value — `$retval` in uretprobe

#### USDT

```bash
perf buildid-cache --add $(which node)             # register binary with USDT probes
perf list | grep sdt_node                          # list USDT probes (SDT events)
perf probe sdt_node:http__server__request          # initialise USDT as tracepoint
perf record -e sdt_node:http__server__request -a  # trace USDT event
```

- Must increment semaphore for some USDT probes — fixed in Linux 4.20
- Arguments printed as pointer addresses by default — cast using probe definition

## perf stat

- Counts events — efficient via kernel context and PMC registers
- Use before `perf record` to check event frequency and overhead cost

```bash
perf stat -e sched:sched_switch -a -- sleep 1      # count context switches for 1s
perf stat -e cycles,instructions -a               # IPC shadow statistic
```

### Options

| Option | Description |
|--------|-------------|
| `-a` | All CPUs (default since Linux 4.11) |
| `-e event` | Specify event(s) — wildcards supported |
| `--filter filter` | Boolean filter for tracepoint event arguments |
| `-p PID` | Specific PID |
| `-t TID` | Specific thread ID |
| `-G cgroup` | Specific cgroup (containers) |
| `-A` | Per-CPU counts |
| `-I ms` | Print per-interval output |
| `--per-socket` | Socket-level aggregation |
| `--per-core` | Core-level aggregation |

### Interval Statistics

```bash
perf stat -e sched:sched_switch -a -I 1000    # print count every 1000 ms
```

### Shadow Statistics

- Automatically computed when certain event combinations are measured
- IPC printed when both `cycles` and `instructions` are measured
- Other examples — branch miss ratio, LLC hit ratio

## perf record

- Records events to `perf.data` for later analysis
- Passed from kernel to user via per-CPU ring buffers

### Options

| Option | Description |
|--------|-------------|
| `-a` | All CPUs |
| `-e event` | Specify event(s) |
| `--filter filter` | Boolean filter expression |
| `-p PID` | Specific PID |
| `-g` | Record stack traces |
| `--call-graph mode` | Stack walking method: `fp`, `dwarf`, or `lbr` |
| `-F Hz` | Sample frequency |
| `-c N` | Sample period (one-in-N) |
| `-o file` | Output file |

### CPU Profiling

- Default event selection priority when no `-e` specified —
cycles:ppp → cycles:pp → cycles:p → cycles → cpu-clock
- `:ppp`, `:pp`, `:p` — precise event modes (Intel PEBS, AMD IBS)

### Stack Walking Methods

| Method | Description |
|--------|-------------|
| `fp` (default) | Frame pointer — fast but requires `-fno-omit-frame-pointer` |
| `dwarf` | Debug info — works without frame pointers — requires debuginfo package |
| `lbr` | Intel Last Branch Record — no overhead — limited to 16–32 frames |

```bash
# max-stack per event
perf record -e sched:sched_switch/max-stack=5/,sched:sched_wakeup/max-stack=1/ -a -- sleep 1
```

## perf report

- Summarises perf.data — TUI (default) or `--stdio` text report

### Options

| Option | Description |
|--------|-------------|
| `--tui` | Interactive TUI (default) |
| `--stdio` | Text report |
| `-n` | Include sample count column |
| `-g callee` | Callee ordering (event function on left) — was default before Linux 4.4 |
| `-g caller` | Caller ordering (root function on left) — current default since Linux 4.4 |
| `-i file` | Input file |

## perf script

- Prints each sample from perf.data individually
- Used for flame graph generation and custom post-processing

### Default Output Fields

process_name  thread_id  [cpu_id]  timestamp:  period  event_name:  ip  sym+offset  (dso)

### Recommended Usage

```bash
perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso,trace
```

- `--header` — includes system metadata, hostname, kernel version, perf command used
- `-F` — specifies exact fields for consistent post-processing

### Flame Graphs

```bash
# Linux 5.8+ built-in
perf record -F 99 -a -g -- sleep 10
perf script report flamegraph

# Original flamegraph.pl approach
perf script --header > out.stacks
./stackcollapse-perf.pl < out.stacks | ./flamegraph.pl --hash > out.svg
```

### Trace Scripts

```bash
perf script -l                        # list available trace scripts
perf script stackcollapse             # produce callgraphs in folded format
perf script syscall-counts-by-pid     # system-wide syscall counts by PID
perf script failed-syscalls-by-pid    # failed syscalls by PID
```

- Custom scripts in Perl or Python
- `stackcollapse` script — compatible input for flamegraph.pl

## perf trace

- Live event tracer — syscalls by default — no perf.data file
- Lower overhead than strace — can trace system-wide
- Adds "beautification" — prints flags/constants as strings (Eg - `prot: READ` not `prot: 1`)

```bash
perf trace                                                              # live syscall trace
perf trace -e block:block_rq_issue,block:block_rq_complete             # live block I/O
perf trace -e syscalls:*enter_mmap --filter='flags==SHARED'            # filtered mmap calls
```

> [!NOTE]
> Prior to Linux 4.19 — also traced all syscalls by default alongside specified events — use `--no-syscalls` to disable (now the default). `-a` system-wide default since Linux 3.8. `--filter` added in Linux 5.5.

## Other Commands

| Command | Description |
|---------|-------------|
| `perf c2c` | Cache-to-cache and false sharing analysis (Linux 4.10+) |
| `perf kmem` | Kernel memory allocation analysis |
| `perf kvm` | KVM guest exit analysis |
| `perf lock` | Lock contention analysis |
| `perf mem` | Memory access analysis |
| `perf sched` | Scheduler latency statistics — `latency`, `timehist`, `map` |
| Intel PT | Per-instruction tracing — `--insn-trace` — use with XED for assembly |

### Intel Processor Trace

```bash
perf record -e intel_pt/cyc/u date        # record user-mode cycles
perf script --insn-trace                  # print as machine code
perf script --insn-trace --xed            # print as assembly (requires Intel XED)
```

## Documentation

- Man pages — `perf-SUBCOMMAND(1)` — Eg - `perf-record(1)`, `perf-probe(1)`
- Linux source — `tools/perf/Documentation`
- Tutorial — `wiki.kernel.org` [Perf 15]
- One-liners and examples — Brendan Gregg's perf examples page
