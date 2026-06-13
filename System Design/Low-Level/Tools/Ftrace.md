## Ftrace

- Official Linux tracer — created by Steven Rostedt — first added Linux 2.6.27 (2008)
- Multi-tool composed of different tracing utilities
- Uses tracefs file system interface — no additional user-level front-end required
- Profilers — statistical summaries (counts, histograms)
- Tracers — per-event details

## Capabilities Overview

### Ftrace Profilers

| Profiler | Description |
|---------|-------------|
| `function` | Kernel function call statistics — counts, timing, stddev |
| kprobe profiler | Hit counts for enabled kprobes |
| uprobe profiler | Hit counts for enabled uprobes |
| hist triggers | Custom histograms on any event (Linux 4.7+) |

### Ftrace and Event Tracers

| Tracer | Description |
|--------|-------------|
| `function` | Per-event kernel function call tracer |
| `tracepoints` | Kernel static instrumentation |
| `kprobes` | Kernel dynamic instrumentation |
| `uprobes` | User-level dynamic instrumentation |
| `function_graph` | Hierarchical graph of child calls with timing |
| `wakeup` | Max CPU scheduler latency |
| `wakeup_rt` | Max CPU scheduler latency for RT tasks |
| `irqsoff` | IRQs-off code location and latency |
| `preemptoff` | Preemption disabled latency |
| `preemptirqsoff` | Combined irqsoff and preemptoff |
| `blk` | Block I/O tracer (used by blktrace) |
| `hwlat` | Hardware latency — detects SMI/hypervisor perturbations |
| `nop` | Disables other tracers |

```bash
cat /sys/kernel/debug/tracing/available_tracers    # list available tracers
```

## tracefs Interface

```bash
mount -t tracefs tracefs /sys/kernel/tracing
mount -t debugfs,tracefs    # show both mount points
```

- Ubuntu — typically mounted at `/sys/kernel/debug/tracing`

### Key tracefs Files

| File | Access | Description |
|------|--------|-------------|
| `available_tracers` | r | Lists available tracers |
| `current_tracer` | r/w | Current enabled tracer (`nop` = none) |
| `function_profile_enabled` | r/w | Enables function profiler |
| `available_filter_functions` | r | Functions available to trace |
| `set_ftrace_filter` | r/w | Select functions to trace |
| `tracing_on` | r/w | Enable/disable output ring buffer |
| `trace` | r/w | Ring buffer output — read shows contents, write clears |
| `trace_pipe` | r | Streaming output — consumes events, blocks for input |
| `trace_options` | r/w | Customise trace output |
| `trace_stat/` | r/w | Function profiler output |
| `kprobe_events` | r/w | kprobe configuration |
| `uprobe_events` | r/w | uprobe configuration |
| `events/` | r/w | Tracepoint/kprobe/uprobe control files |
| `instances/` | r/w | Ftrace instances for concurrent users |
| `set_graph_function` | r/w | Function to trace for function_graph |

> [!NOTE]
> Ftrace originally did not support concurrent users. Use the `instances/` directory to create separate instances with independent `current_tracer` and output files.

## Ftrace Function Profiler

- Provides counts, total time, average time, and stddev per kernel function
- Uses compiled-in profiling hooks (`__fentry__()`) — replaced with nops when not in use — low overhead
- Requires `CONFIG_FUNCTION_PROFILER=y`

```bash
cd /sys/kernel/debug/tracing
echo 'tcp*' > set_ftrace_filter          # filter to tcp* functions only
echo 1 > function_profile_enabled        # enable profiler
sleep 10                                  # collect for 10 seconds
echo 0 > function_profile_enabled        # disable
echo > set_ftrace_filter                  # reset filter

# Read results per CPU
head trace_stat/function*                 # columns: Function, Hit, Time, Avg, s^2
```

> [!TIP]
> Use `set_ftrace_filter` to limit scope and reduce overhead. Without a filter, all kernel functions are profiled.

## Ftrace Function Tracing

- Prints per-event details for kernel function calls — sequence, timestamps, on-CPU process name/PID
- Higher overhead than profiling — suited for infrequent functions (<1000 calls/s)
- Use function profiler first to check call rates

### Using trace (file capture)

```bash
cd /sys/kernel/debug/tracing
echo 1 > tracing_on
echo '*sleep' > set_ftrace_filter
echo function > current_tracer
sleep 10
cat trace > /tmp/out.trace.txt
echo nop > current_tracer              # disables tracer AND clears buffer
echo > set_ftrace_filter               # reset filter
```

- Writing `nop` to `current_tracer` clears the trace buffer
- Writing a newline to `trace` clears the buffer manually (`> trace`)

### Output Format

TASK-PID   CPU#  IRQFLAG  TIMESTAMP  FUNCTION <- PARENT

multipathd-348   [001] ....  332762.532877: do_nanosleep <-hrtimer_nanosleep

- Flags column (`....`) — irqs-off, need-resched, hardirq/softirq, preempt-depth

### Using trace_pipe (live streaming)

```bash
echo '*sleep' > set_ftrace_filter
echo function > current_tracer
cat trace_pipe                         # streams events live — consumes them
# Ctrl-C to stop
echo nop > current_tracer
echo > set_ftrace_filter
```

### Options

```bash
ls options/                            # list available options
echo 0 > options/irq-info             # disable flags column
echo 1 > options/irq-info             # re-enable
echo 1 > options/stacktrace           # append kernel stacks to output
echo 1 > options/userstacktrace       # append user stacks to output
```

## Tracepoints

```bash
cd /sys/kernel/debug/tracing
echo 1 > events/block/block_rq_issue/enable     # enable tracepoint
cat trace_pipe                                   # stream events
echo 0 > events/block/block_rq_issue/enable     # disable
```

- Per-event files — `enable`, `filter`, `format`, `hist`, `id`, `trigger`

### Filter

```bash
echo 'bytes > 65536' > events/block/block_rq_insert/filter   # only trace large I/O
echo 0 > events/block/block_rq_insert/filter                  # reset filter
```

- Field operators — numbers: `==`, `!=`, `<`, `<=`, `>`, `>=`, `&` — strings: `==`, `!=`, `~`
- Glob-style wildcards for strings — `*`, `?`, `[]`
- Combine with `&&`, `||`, parentheses

### Trigger

```bash
cat events/block/block_rq_issue/trigger    # list available trigger commands
# Available: traceon traceoff snapshot stacktrace enable_event disable_event enable_hist disable_hist hist
```

- `traceoff` — disable tracing when event fires (preserve buffer of prior events)
- `snapshot` — snapshot trace buffer when event fires
- `stacktrace` — print stack trace when event fires
- Combine with filter using `if` keyword —

```bash
echo 'traceoff if bytes > 65536' > events/block/block_rq_insert/trigger
```

## kprobes

### Event Tracing

```bash
echo 'p:brendan do_nanosleep' >> kprobe_events          # create kprobe named "brendan"
echo 1 > events/kprobes/brendan/enable                   # enable
cat trace_pipe                                           # stream events
echo 0 > events/kprobes/brendan/enable                   # disable
echo '-:brendan' >> kprobe_events                        # delete kprobe
```

#### kprobe_events Syntax

p[:[GRP/]EVENT] [MOD:]SYM[+offs]|MEMADDR [FETCHARGS]    # entry probe

r[MAXACTIVE][:[GRP/]EVENT] [MOD:]SYM[+0] [FETCHARGS]    # return probe

-:[GRP/]EVENT                                            # delete probe

### Arguments (Linux 4.20+)

```bash
echo 'p:brendan do_nanosleep hrtimer_sleeper=$arg1 hrtimer_mode=$arg2' >> kprobe_events
```

- Pre-4.20 — use register names (Eg - `%di`, `%si` for x86_64 args 1 and 2)

```bash
echo 'p:brendan do_nanosleep hrtimer_sleeper=%di hrtimer_mode=%si' >> kprobe_events
```

- x86_64 AMD64 ABI — arg1=`rdi`, arg2=`rsi`, arg3=`rdx`, arg4=`r10`, arg5=`r8`, arg6=`r9`

### Return Values

```bash
echo 'r:brendan do_nanosleep ret=$retval' >> kprobe_events    # kretprobe
```

### Filters and Triggers

- Same interface as tracepoints — use `events/kprobes/PROBE_NAME/filter` and `events/kprobes/PROBE_NAME/trigger`
- Format file shows custom fields from argument definitions

### kprobe Profiling

```bash
cat /sys/kernel/debug/tracing/kprobe_profile
# columns: probe_name, hit_count, miss_hits
```

## uprobes

### Event Tracing

```bash
readelf -s /bin/bash | grep -w readline    # find symbol offset
echo 'p:brendan /bin/bash:0xb61e0' >> uprobe_events
echo 1 > events/uprobes/brendan/enable
cat trace_pipe
echo 0 > events/uprobes/brendan/enable
echo '-:brendan' >> uprobe_events
```

> [!NOTE]
> Use a higher-level tracer (BCC or bpftrace) instead of the raw uprobe_events interface — a wrong offset in a shared library can corrupt all processes that use it.

#### uprobe_events Syntax

p[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS]    # entry uprobe

r[:[GRP/]EVENT] PATH:OFFSET [FETCHARGS]    # uretprobe

-:[GRP/]EVENT                              # delete

### uprobe Profiling

```bash
cat /sys/kernel/debug/tracing/uprobe_profile
# columns: path, probe_name, hit_count
```

## Ftrace function_graph

- Shows child function calls and code flow with per-function timing
- Latency symbols — `$` >1s, `@` >100ms, `*` >10ms, `#` >1ms, `!` >100μs, `+` >10μs

```bash
cd /sys/kernel/debug/tracing
echo do_nanosleep > set_graph_function
echo function_graph > current_tracer
cat trace_pipe
echo nop > current_tracer
echo > set_graph_function
```

- Without `set_ftrace_filter` — all child calls traced — overhead inflates reported durations
- With `set_ftrace_filter` — limits which child functions add instrumentation overhead

```bash
# Reduce overhead by also filtering child functions
echo do_nanosleep > set_ftrace_filter
cat trace_pipe                    # reported durations now more accurate
```

### Options

```bash
ls options/funcgraph-*
# funcgraph-cpu, funcgraph-proc, funcgraph-duration, funcgraph-overhead, funcgraph-irqs, etc.
```

## Ftrace hwlat

- Detects external hardware latency — SMI events, hypervisor perturbations
- Runs CPU loop with interrupts disabled — measures iteration timing
- Reports slowest iterations exceeding threshold (10 μs default)

```bash
echo hwlat > current_tracer
cat trace_pipe
# Output: #seq inner/outer(us): NNN/NNN  ts:...
echo nop > current_tracer
```

- Configure via `/sys/kernel/debug/tracing/hwlat_detector/width` and `window` (microseconds)

> [!NOTE]
> hwlat is a microbenchmark tool — it makes one CPU busy with interrupts disabled for the `width` duration. This will perturb the system.

## Ftrace Hist Triggers

- Custom histograms on any event — Linux 4.7+
- Created by writing `hist:expression` to an event's `trigger` file

### Workflow

```bash
echo 'hist:keys=FIELD' > events/SYSTEM/EVENT/trigger    # create
sleep 10                                                  # populate
cat events/SYSTEM/EVENT/hist                             # print
echo '!hist:keys=FIELD' > events/SYSTEM/EVENT/trigger   # remove
```

### Hist Expression Syntax

hist:keys=<field1[,field2,...]>[:values=<field1[,field2,...]>]

[:sort=<field1[,field2,...]>][:size=#entries][:pause][:continue]

[:clear][:name=histname1][:<handler>.<action>] [if <filter>]

### Single Key

```bash
echo 'hist:key=common_pid' > events/raw_syscalls/sys_enter/trigger
# Output: { common_pid: 32396 } hitcount: 882604
```

### Modifiers

```bash
echo 'hist:key=common_pid.execname' > events/raw_syscalls/sys_enter/trigger
# Output: { common_pid: dd [32396] } hitcount: 869024

echo 'hist:key=id.syscall' > events/raw_syscalls/sys_enter/trigger
# Output: { id: sys_write [1] } hitcount: 106425
```

- `.execname` — adds process name next to PID
- `.syscall` — adds syscall name next to syscall ID
- `.log2` — bucket into log2 ranges (reduces entries for latency histograms)

### PID Filter

```bash
echo 'hist:key=id.syscall if common_pid==32396' > events/raw_syscalls/sys_enter/trigger
```

### Multiple Keys

```bash
echo 'hist:key=common_pid.execname,id' > events/raw_syscalls/sys_enter/trigger
# Output: { common_pid: dd [14325], id: 0 } hitcount: 9195176
```

### Stack Trace Key

```bash
echo 'hist:key=stacktrace' > events/block/block_rq_issue/trigger
# Output: { stacktrace: nvme_queue_rq+... __blk_mq_try_issue_directly+... ... } hitcount: 266
```

### Synthetic Events (Custom Latency)

```bash
# Create synthetic event definition
echo 'syscall_latency u64 lat_us; long id' >> synthetic_events

# On sys_enter — save timestamp keyed by PID
echo 'hist:keys=common_pid:ts0=common_timestamp.usecs' >> \
    events/raw_syscalls/sys_enter/trigger

# On sys_exit — calculate latency, trigger synthetic event when PID matches
echo 'hist:keys=common_pid:lat_us=common_timestamp.usecs-$ts0:'\
    'onmatch(raw_syscalls.sys_enter).trace(syscall_latency,$lat_us,id)' >>\
    events/raw_syscalls/sys_exit/trigger

# Create histogram of the synthetic event
echo 'hist:keys=lat_us,id.syscall:sort=lat_us' >> \
    events/synthetic/syscall_latency/trigger

sleep 10
cat events/synthetic/syscall_latency/hist
```

- Synthetic events — combine arguments from multiple events — enable custom latency calculations
- Uses hashes to store values from earlier events (timestamps) for retrieval by later events

## trace-cmd

- Open-source Ftrace front end — Steven Rostedt
- Safer than raw /sys — handles cleanup of tracing state
- Binary output format (`trace.dat`)

### Selected Subcommands

| Command | Description |
|---------|-------------|
| `record` | Trace and record to trace.dat |
| `report` | Read and display trace.dat |
| `stream` | Trace and print to stdout |
| `list` | List available events |
| `stat` | Show kernel tracing subsystem status |
| `profile` | Trace and generate custom report |
| `listen` | Accept network tracing requests |

### Key One-Liners

```bash
# Listing
trace-cmd list -t                                         # Ftrace tracers
trace-cmd list -e                                         # tracepoints, kprobes, uprobes
trace-cmd list -e syscalls:                              # syscall tracepoints

# Function tracing
trace-cmd record -p function -l 'tcp_*' sleep 10         # trace tcp* functions 10s
trace-cmd record -p function -l 'vfs_*' -F ls            # trace vfs* for ls command
trace-cmd record -p function -l 'vfs_*' -F -c bash       # trace for bash and children
trace-cmd record -p function -l 'vfs_*' -P 21124         # trace for specific PID

# Function graph
trace-cmd record -p function_graph -g do_nanosleep sleep 10

# Event tracing
trace-cmd record -e sched_process_exec                   # new processes
trace-cmd record -e block_rq_issue -T                    # block I/O with stacks
trace-cmd record -e block                                # all block tracepoints
trace-cmd record -e syscalls -F ls                       # all syscalls for ls

# Reporting
trace-cmd report
trace-cmd report --cpu 0                                 # CPU 0 only

# Network
trace-cmd listen -p 8081
trace-cmd record ... -N addr:port
```

### trace-cmd vs perf(1)

| Attribute | perf(1) | trace-cmd |
|-----------|---------|----------|
| Binary output | perf.data | trace.dat |
| Tracepoints | Yes | Yes |
| kprobes | Yes | Partial (must be pre-created) |
| PMCs | Yes | No |
| Timed sampling | Yes | No |
| function tracing | Partial (via ftrace subcommand) | Yes |
| function_graph | Partial | Yes |
| Network client/server | No | Yes |
| Output overhead | Low | Very low |
| Front ends | Various | KernelShark |

### KernelShark

- Visual interface for trace.dat — created by Steven Rostedt — rewritten in Qt by Yordan Karadzhov
- Per-CPU timeline with colored tasks + event table
- Interactive — click and drag right to zoom in, left to zoom out
- Identifies performance issues from thread interactions

```bash
trace-cmd record -e 'sched:*'
kernelshark                            # reads trace.dat automatically
```

## perf ftrace

- `perf(1)` ftrace subcommand — wrapper for function and function_graph tracers
- Does not write to perf.data — prints to stdout

```bash
perf ftrace -T do_nanosleep -a sleep 10          # function tracer
perf ftrace -G do_nanosleep -a sleep 10          # function_graph tracer
```

## perf-tools

- Open-source collection — Ftrace and perf(1)-based — Brendan Gregg — used at Netflix
- Mostly shell scripts — few dependencies (shell + awk)
- Single-purpose tools follow Unix philosophy — one job, concise default output

### Single-Purpose Tools

| Tool | Backend | Description |
|------|---------|-------------|
| `bitesize(8)` | perf | Disk I/O size histogram |
| `cachestat(8)` | Ftrace | Page cache hit/miss statistics |
| `execsnoop(8)` | Ftrace | Trace new processes via `execve(2)` with arguments |
| `iolatency(8)` | Ftrace | Disk I/O latency histogram |
| `iosnoop(8)` | Ftrace | Trace disk I/O with latency details |
| `killsnoop(8)` | Ftrace | Trace `kill(2)` signals |
| `opensnoop(8)` | Ftrace | Trace `open(2)`-family syscalls with filenames |
| `tcpretrans(8)` | Ftrace | Trace TCP retransmits with addresses and kernel state |

### Multi-Purpose Tools

| Tool | Description |
|------|-------------|
| `funccount(8)` | Count kernel function calls |
| `funcgraph(8)` | Trace kernel functions with child call flow |
| `functrace(8)` | Trace kernel functions |
| `funcslower(8)` | Trace kernel functions slower than threshold |
| `kprobe(8)` | Dynamic tracing of kernel functions |
| `uprobe(8)` | Dynamic tracing of user-level functions |
| `tpoint(8)` | Trace tracepoints |
| `syscount(8)` | Summarise syscalls |
| `perf-stat-hist(8)` | Custom power-of aggregations for tracepoint arguments |

### Key One-Liners

```bash
# Ftrace profiling
funccount 'tcp_*'                                              # count all tcp* functions
funccount -t 10 -i 1 'vfs*'                                   # top 10 every 1 second

# Ftrace tracing
funcgraph do_nanosleep                                         # show all child calls
funcgraph -m 3 do_nanosleep                                    # 3 levels deep
funcslower vfs_read 10000                                      # calls slower than 10ms

# kprobes
kprobe p:do_sys_open                                           # trace entry
kprobe 'r:do_sys_open $retval'                                 # trace return value
kprobe 'p:do_sys_open mode=$arg3:u16'                         # trace argument (Linux 4.20+)
kprobe 'p:do_sys_open filename=+0($arg2):string'              # trace string argument
kprobe 'p:do_sys_open file=+0($arg2):string' 'file ~ "*stat"' # with filter
kprobe -s p:tcp_retransmit_skb                                 # with kernel stack traces

# uprobes
uprobe p:bash:readline                                         # trace all bash readline calls
uprobe 'r:bash:readline +0($retval):string'                   # return value as string
uprobe 'p:/bin/bash:readline prompt=+0(%di):string'           # entry argument (x86_64)
uprobe -p 1234 p:libc:gettimeofday                             # filter to PID 1234
uprobe 'r:libc:fopen file=$retval' 'file == 0'                # trace failed fopen calls

# tracepoints
tpoint -l                                                      # list all tracepoints
tpoint -s block:block_rq_issue                                 # trace with stack traces
```

### CPU Register Reference (x86_64 AMD64 ABI)

| Argument | Register |
|----------|---------|
| arg1 | `rdi` / `%di` |
| arg2 | `rsi` / `%si` |
| arg3 | `rdx` / `%dx` |
| arg4 | `r10` |
| arg5 | `r8` |
| arg6 | `r9` |

- `$argN` aliases available since Linux 4.20 — preferred over register names

### perf-tools vs BCC/BPF

| Advantage of perf-tools | Reason |
|------------------------|--------|
| `funccount(8)` | Uses Ftrace function profiling — more efficient than kprobe-based BCC version |
| `funcgraph(8)` | No BCC equivalent — uses Ftrace function_graph |
| Hist triggers | Future tools using hist triggers will be more efficient than kprobe-based BPF |
| Dependencies | Only requires shell and awk — suited for embedded Linux |

## Documentation

- Ftrace — `Documentation/trace/ftrace.rst`
- kprobes — `Documentation/trace/kprobetrace.rst`
- uprobes — `Documentation/trace/uprobetracer.rst`
- Trace events — `Documentation/trace/events.rst`
- Hist triggers — `Documentation/trace/histogram.rst`
- trace-cmd — https://trace-cmd.org
- perf-tools — https://github.com/brendangregg/perf-tools