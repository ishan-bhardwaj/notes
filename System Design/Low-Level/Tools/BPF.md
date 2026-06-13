## BPF

- Extended BPF — kernel execution environment providing programmability to tracers
- Two recommended front ends — BCC (complex tools) and bpftrace (ad hoc programs)
- Key advantage — in-kernel aggregation and custom summaries — avoids dumping all events to user space

## BCC vs bpftrace

| Characteristic | BCC | bpftrace |
|---------------|-----|----------|
| Tool count | >80 (bcc repo), >120 (book repo) | >30 (bpftrace repo) |
| Tool complexity | Complex options (-h, -P PID, etc.) | Typically simple — zero or one argument |
| Programming language | Python/Lua/C/C++ (user-space), C (kernel) | bpftrace language |
| Programming difficulty | Difficult | Easy |
| Per-event output | Anything | Text, JSON |
| Summary types | Anything | Counts, min, max, sum, avg, log2/linear histograms |
| Library support | Yes | No |
| Average program length | ~228 lines | ~28 lines |

- BCC like C programming — bpftrace like shell scripting
- Develop in bpftrace first — port to BCC if needed
- Netflix — BCC for canned tools and dashboards — bpftrace for custom investigation

## BCC

### Installation

```bash
# Search for: bcc-tools, bpfcc-tools, or bcc
# Requires: CONFIG_BPF=y, CONFIG_BPF_SYSCALL=y, CONFIG_BPF_EVENTS=y
# Minimum: Linux 4.4 (some tools), Linux 4.9 (most tools)
```

### Single-Purpose Tools

| Tool | Description |
|------|-------------|
| `biolatency(8)` | Block I/O latency histogram |
| `biotop(8)` | Block I/O by process |
| `biosnoop(8)` | Trace block I/O with latency details |
| `bitesize(8)` | Block I/O size histogram by process |
| `cpudist(8)` | On/off-CPU time per process histogram |
| `drsnoop(8)` | Direct memory reclaim events |
| `execsnoop(8)` | Trace new processes via `execve(2)` |
| `ext4dist(8)` / `xfsdist(8)` / `zfsdist(8)` | FS operation latency histograms |
| `ext4slower(8)` / `xfsslower(8)` / `zfsslower(8)` | Trace slow FS operations |
| `hardirqs(8)` | Hard IRQ event time summary |
| `memleak(8)` | Show outstanding memory allocations |
| `offcputime(8)` | Off-CPU time by stack trace |
| `opensnoop(8)` | Trace `open(2)`-family syscalls |
| `oomkill(8)` | Trace OOM killer |
| `profile(8)` | CPU profiling via timed stack trace sampling |
| `runqlat(8)` | Scheduler run queue latency histogram |
| `runqlen(8)` | Run queue length sampling |
| `syscount(8)` | Syscall counts and latencies |
| `tcplife(8)` | TCP session lifespan tracing |
| `tcpretrans(8)` | TCP retransmit tracing with kernel state |
| `tcptop(8)` | TCP send/recv throughput by host and PID |

### Multi-Purpose Tools

| Tool | Description |
|------|-------------|
| `argdist(8)` | Function parameter values as histogram or count |
| `funccount(8)` | Count kernel or user-level function calls |
| `funcslower(8)` | Trace slow kernel or user-level function calls |
| `funclatency(8)` | Function latency histogram |
| `stackcount(8)` | Count stack traces that led to an event |
| `trace(8)` | Trace arbitrary functions with filters |

### BCC One-Liners

```bash
# funccount(8)
funccount 'vfs_*'                     # count VFS kernel calls
funccount 'tcp_*'                     # count TCP kernel calls
funccount -i 1 'tcp_send*'            # TCP send calls per second
funccount -i 1 't:block:*'            # block I/O events per second
funccount -i 1 c:getaddrinfo          # libc getaddrinfo() per second

# stackcount(8)
stackcount t:block:block_rq_insert    # stack traces creating block I/O
stackcount -P ip_output               # stacks sending IP packets, with PID
stackcount t:sched:sched_switch       # stacks leading to thread blocking

# trace(8)
trace 'do_sys_open "%s", arg2'                    # open with filename
trace 'r::do_sys_open "ret: %d", retval'          # open return value
trace -U 'do_nanosleep "mode: %d", arg2'          # with user-level stacks
trace 'pam:pam_start "%s: %s", arg1, arg2'        # PAM auth requests

# argdist(8)
argdist -H 'r::vfs_read()'                         # VFS read return values
argdist -p 1005 -H 'r:c:read()'                   # libc read() for PID 1005
argdist -C 't:raw_syscalls:sys_enter():int:args->id'  # syscalls by ID
argdist -H 'p::tcp_sendmsg(...):u32:size'         # tcp_sendmsg() size histogram
argdist -C 'r::__vfs_read():u32:$PID:$latency > 100000'  # reads where latency >100μs
```

## bpftrace

- High-level language built upon BPF and BCC
- Inspired by awk and C — similar program structure to awk
- Created by Alastair Robertson

### Installation

```bash
# Search for: bpftrace
# Available: Ubuntu, Fedora, Gentoo, Debian, OpenSUSE, CentOS, RHEL 8.2 (Tech Preview)
# Docker images and standalone binaries also available
# Requires: CONFIG_BPF=y, CONFIG_BPF_SYSCALL=y, CONFIG_BPF_EVENTS=y, Linux 4.9+
```

### Program Structure

probes { actions }

probes /filter/ { actions }

- Multiple probes — comma-separated — execute same actions
- `BEGIN` and `END` — fire at program start and end (like awk)

### Usage

```bash
bpftrace -e 'program'         # run one-liner
bpftrace file.bt              # run from file
sudo ./file.bt                # run as executable (with interpreter line)
```

### bpftrace One-Liners

```bash
# CPUs
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { join(args->argv); }'   # new processes
bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[pid, comm] = count(); }'  # syscalls by process
bpftrace -e 'profile:hz:49 /pid == 189/ { @[ustack] = count(); }'         # user stacks at 49 Hz

# Memory
bpftrace -e 'tracepoint:syscalls:sys_enter_brk { @[ustack, comm] = count(); }'    # heap expansion
bpftrace -e 'tracepoint:exceptions:page_fault_user { @[ustack, comm] = count(); }' # page faults

# File Systems
bpftrace -e 't:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
bpftrace -e 'tracepoint:syscalls:sys_exit_read { @ = hist(args->ret); }'   # read() size distribution
bpftrace -e 'kprobe:vfs_* { @[probe] = count(); }'                         # count VFS calls
bpftrace -e 'tracepoint:ext4:* { @[probe] = count(); }'                    # count ext4 calls

# Disk
bpftrace -e 't:block:block_rq_issue { @bytes = hist(args->bytes); }'      # block I/O size histogram
bpftrace -e 't:block:block_rq_issue { @[ustack] = count(); }'             # user stacks for block I/O
bpftrace -e 't:block:block_rq_issue { @[args->rwbs] = count(); }'         # count I/O type flags

# Networking
bpftrace -e 'k:tcp_sendmsg { @send_bytes = hist(arg2); }'                 # TCP send size histogram
bpftrace -e 'kr:tcp_recvmsg /retval >= 0/ { @recv_bytes = hist(retval); }' # TCP receive histogram
bpftrace -e 'k:udp_sendmsg { @send_bytes = hist(arg2); }'                 # UDP send histogram
bpftrace -e 'kr:sock_sendmsg,kr:sock_recvmsg /retval > 0/ { @[pid, comm] = sum(retval); }'

# Applications
bpftrace -e 'u:/lib/x86_64-linux-gnu/libc-2.27.so:malloc { @[ustack(5)] = sum(arg0); }'  # malloc by stack
bpftrace -e 't:syscalls:sys_enter_kill { printf("%s -> PID %d SIG %d\n", comm, args->pid, args->sig); }'

# Kernel
bpftrace -e 'kprobe:vfs_write { @[arg2] = count(); }'                     # count vfs_write by size arg
bpftrace -e 'k:vfs_read { @ts[tid] = nsecs; } kr:vfs_read /@ts[tid]/ { @ = hist(nsecs - @ts[tid]); delete(@ts[tid]); }'  # vfs_read latency
bpftrace -e 't:sched:sched_switch { @[kstack, ustack, comm] = count(); }' # context switch stacks
bpftrace -e 'profile:hz:99 /pid/ { @[kstack] = count(); }'                # kernel stacks at 99 Hz
```

## bpftrace Programming

### Probe Types

| Type | Shortcut | Description |
|------|---------|-------------|
| `tracepoint` | `t` | Kernel static instrumentation |
| `usdt` | `U` | User-level statically defined tracing |
| `kprobe` | `k` | Kernel dynamic function entry |
| `kretprobe` | `kr` | Kernel dynamic function return |
| `kfunc` | `f` | Kernel dynamic function (BPF-based, low overhead) |
| `kretfunc` | `fr` | Kernel dynamic function return (BPF-based) |
| `uprobe` | `u` | User-level dynamic function entry |
| `uretprobe` | `ur` | User-level dynamic function return |
| `software` | `s` | Kernel software-based events |
| `hardware` | `h` | Hardware counter-based events |
| `watchpoint` | `w` | Memory watchpoint |
| `profile` | `p` | Timed sampling across all CPUs |
| `interval` | `i` | Timed reporting (from one CPU) |
| `BEGIN` | | Start of bpftrace program |
| `END` | | End of bpftrace program |

```bash
# Probe format
type:identifier1[:identifier2[...]]

# Examples
kprobe:vfs_read
uprobe:/bin/bash:readline
tracepoint:syscalls:sys_enter_read

# Multiple probes
probe1,probe2 { actions }

# Wildcards
kprobe:vfs_*                         # all kernel functions starting with vfs_
bpftrace -l 'kprobe:vfs_*'          # list matched probes
```

> [!NOTE]
> Default maximum probes: 512 (set via `BPFTRACE_MAX_PROBES`). More than 512 causes slow start/shutdown as probes are instrumented one by one.

### Probe Arguments

- Tracepoints — use `args->fieldname` (fields from format file)
- kprobes/uprobes — use `arg0`, `arg1`, ..., `argN` (positional) or `args->name` (with BTF/debuginfo)
- kretprobes/uretprobes — use `retval`

```bash
bpftrace -lv 'tracepoint:syscalls:sys_enter_read'   # list tracepoint fields
```

### Flow Control

#### Filters

probe /filter/ { action }

probe /pid == 123/ { action }

probe /pid > 100 && pid < 1000/ { action }

probe /pid/ { action }           # same as /pid != 0/

#### Ternary Operator
abs = $x >= 0 ? $x : -
x;

#### If Statements
if (test) { statements }

if (test) { true_statements } else { false_statements }

#### Loops

while (test) { statements }   # Linux 5.3+

unroll(N) { statements }      # older kernels

### Operators

| Category | Operators |
|----------|-----------|
| Assignment | `=` |
| Arithmetic | `+`, `-`, `*`, `/` |
| Increment/Decrement | `++`, `--` |
| Bitwise | `&`, `|`, `^` |
| Logical not | `!` |
| Shift | `<<`, `>>` |
| Compound | `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `^=`, `<<=`, `>>=` |
| Boolean | `==`, `!=`, `>`, `<`, `>=`, `<=`, `&&`, `||` |

### Variables

#### Built-in Variables

| Variable | Type | Description |
|----------|------|-------------|
| `pid` | integer | Process ID (kernel tgid) |
| `tid` | integer | Thread ID (kernel pid) |
| `uid` | integer | User ID |
| `username` | string | Username |
| `nsecs` | integer | Timestamp in nanoseconds |
| `elapsed` | integer | Nanoseconds since bpftrace start |
| `cpu` | integer | Processor ID |
| `comm` | string | Process name |
| `kstack` | string | Kernel stack trace |
| `ustack` | string | User-level stack trace |
| `arg0..argN` | integer | Arguments for some probe types |
| `args` | struct | Structured arguments for tracepoints |
| `retval` | integer | Return value for kretprobe/uretprobe |
| `func` | string | Name of traced function |
| `probe` | string | Full name of current probe |
| `curtask` | struct | Kernel `task_struct` |
| `cgroup` | integer | Default cgroup v2 ID |
| `$1...$N` | various | Positional parameters to bpftrace program |

#### Scratch Variables (temporary, per-action)

$x = 1;

$y = "hello";

$z = (struct task_struct *)curtask;

#### Map Variables (global storage)
@start[tid] = nsecs;             # hash keyed by tid

@path[pid, $fd] = str(arg0);    # multi-key hash

@count = count();                # global counter

- All maps automatically printed when bpftrace terminates

### Built-in Functions

| Function | Description |
|----------|-------------|
| `printf(fmt [, ...])` | Formatted output (async) |
| `time(fmt)` | Print formatted time (async) |
| `join(char *arr[])` | Print string array joined by space |
| `str(char *s [, len])` | String from pointer with optional length |
| `buf(void *d [, len])` | Hex string from data pointer |
| `strncmp(s1, s2, len)` | Compare strings up to length |
| `sizeof(expr)` | Size of expression or type |
| `kstack([limit])` | Kernel stack up to limit frames |
| `ustack([limit])` | User stack up to limit frames |
| `ksym(void *p)` | Kernel address to symbol string |
| `usym(void *p)` | User-space address to symbol string |
| `kaddr(char *name)` | Kernel symbol name to address |
| `uaddr(char *name)` | User-space symbol name to address |
| `reg(char *name)` | Value in named register |
| `ntop([af,] addr)` | IPv4/IPv6 address to string |
| `cgroupid(char *path)` | cgroup ID for given path |
| `system(fmt [, ...])` | Execute shell command (async) |
| `cat(filename)` | Print file contents (async) |
| `signal(sig)` | Send signal to current task |
| `override(rc)` | Override kprobe return value |
| `exit()` | Exit bpftrace |

> [!NOTE]
> Async functions: `printf()`, `time()`, `cat()`, `join()`, `system()`. Stack/symbol functions record addresses synchronously but translate asynchronously.

### Map Functions

| Function | Description |
|----------|-------------|
| `count()` | Count occurrences — per-CPU map |
| `sum(n)` | Sum values |
| `avg(n)` | Average values |
| `min(n)` | Record minimum value |
| `max(n)` | Record maximum value |
| `stats(n)` | Returns count, average, and total |
| `hist(n)` | Power-of-two histogram |
| `lhist(n, min, max, step)` | Linear histogram |
| `delete(@m[key])` | Delete map key/value pair |
| `print(@m [, top [, div]])` | Print map with optional limits and divisor (async) |
| `clear(@m)` | Delete all keys (async) |
| `zero(@m)` | Set all values to zero (async) |

### Example: Timing vfs_read()

```bash
#!/usr/local/bin/bpftrace

// time vfs_read() and print as microsecond histogram

kprobe:vfs_read
{
    @start[tid] = nsecs;
}

kretprobe:vfs_read
/@start[tid]/
{
    $duration_us = (nsecs - @start[tid]) / 1000;
    @us = hist($duration_us);
    delete(@start[tid]);
}
```

- `kprobe:vfs_read` — saves timestamp keyed by thread ID on entry
- `kretprobe:vfs_read` with filter `/@start[tid]/` — ensures start time was recorded (excludes in-flight calls when tracing began)
- `delete(@start[tid])` — prevents map from growing unboundedly

```bash
# Customisation — per-process breakdown
@us[pid, comm] = hist($duration_us);
```

### Measuring Probe Frequency

```bash
# Count vfs_read() calls for one second then exit
bpftrace -e 'k:vfs_read { @ = count(); } interval:s:1 { exit(); }'
```

- Rough guide — <100k kprobe or tracepoint events/second = low frequency
- Use less-frequent events wherever possible to reduce overhead

## Documentation

### BCC

```bash
funccount -h                           # usage message
man man8/funccount.8                   # man page
# examples/funccount_example.txt       # example output with commentary
```

- `docs/tutorial.md` — end user tutorial
- `docs/tutorial_bcc_python_developer.md` — developer tutorial
- `docs/reference_guide.md` — reference guide

### bpftrace

```bash
bpftrace -l 'kprobe:vfs_*'            # list matched probes
bpftrace -lv 'tracepoint:syscalls:sys_enter_read'  # list probe fields
```

- `docs/reference_guide.md` — full reference
- `docs/tutorial_one_liners.md` — one-liner tutorial
- https://github.com/iovisor/bpftrace — reference guide online

## bpftrace One-Liners Reference

## CPUs

```bash
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { join(args->argv); }'
# Trace new processes with arguments

bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[pid, comm] = count(); }'
# Count syscalls by process

bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'
# Count syscalls by syscall probe name

bpftrace -e 'profile:hz:99 { @[comm] = count(); }'
# Sample running process names at 99 Hz

bpftrace -e 'profile:hz:49 { @[kstack, ustack, comm] = count(); }'
# Sample user and kernel stacks at 49 Hz system-wide with process name

bpftrace -e 'profile:hz:49 /pid == 189/ { @[ustack] = count(); }'
# Sample user-level stacks at 49 Hz for PID 189

bpftrace -e 'profile:hz:49 /pid == 189/ { @[ustack(5)] = count(); }'
# Sample user-level stacks 5 frames deep at 49 Hz for PID 189

bpftrace -e 'profile:hz:49 /comm == "mysqld"/ { @[ustack] = count(); }'
# Sample user-level stacks at 49 Hz for processes named "mysqld"

bpftrace -e 'tracepoint:sched:* { @[probe] = count(); }'
# Count kernel CPU scheduler tracepoints

bpftrace -e 'tracepoint:sched:sched_switch { @[kstack] = count(); }'
# Count off-CPU kernel stacks for context switch events

bpftrace -e 'kprobe:vfs_* { @[func] = count(); }'
# Count kernel function calls beginning with "vfs_"

bpftrace -e 'u:/lib/x86_64-linux-gnu/libpthread-2.27.so:pthread_create {
    printf("%s by %s (%d)\n", probe, comm, pid); }'
# Trace new threads via pthread_create()
```

## Memory

```bash
bpftrace -e 'u:/lib/x86_64-linux-gnu/libc.so.6:malloc {
    @[ustack, comm] = sum(arg0); }'
# Sum malloc() request bytes by user stack and process (high overhead)

bpftrace -e 'u:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 181/ {
    @[ustack] = sum(arg0); }'
# Sum malloc() request bytes by user stack for PID 181 (high overhead)

bpftrace -e 'u:/lib/x86_64-linux-gnu/libc.so.6:malloc /pid == 181/ {
    @[ustack] = hist(arg0); }'
# malloc() request bytes as power-of-2 histogram for PID 181 (high overhead)

bpftrace -e 't:kmem:kmem_cache_alloc { @bytes[kstack] = sum(args->bytes_alloc); }'
# Sum kernel kmem cache allocation bytes by kernel stack trace

bpftrace -e 'tracepoint:syscalls:sys_enter_brk { @[ustack, comm] = count(); }'
# Count process heap expansion (brk(2)) by code path

bpftrace -e 'software:page-fault:1 { @[comm, pid] = count(); }'
# Count page faults by process

bpftrace -e 't:exceptions:page_fault_user { @[ustack, comm] = count(); }'
# Count user page faults by user-level stack trace

bpftrace -e 'tracepoint:vmscan:* { @[probe]++; }'
# Count vmscan operations by tracepoint

bpftrace -e 'kprobe:swap_readpage { @[comm, pid] = count(); }'
# Count swapins by process

bpftrace -e 'tracepoint:migrate:mm_migrate_pages { @ = count(); }'
# Count page migrations

bpftrace -e 't:compaction:mm_compaction_begin { time(); }'
# Trace compaction events

bpftrace -l 'usdt:/lib/x86_64-linux-gnu/libc.so.6:*'
# List USDT probes in libc

bpftrace -l 't:kmem:*'
# List kernel kmem tracepoints

bpftrace -l 't:*:mm_*'
# List all memory subsystem (mm) tracepoints
```

## File Systems

```bash
bpftrace -e 't:syscalls:sys_enter_openat { printf("%s %s\n", comm,
    str(args->filename)); }'
# Trace files opened via openat(2) with process name

bpftrace -e 'tracepoint:syscalls:sys_enter_*read* { @[probe] = count(); }'
# Count read syscalls by syscall type

bpftrace -e 'tracepoint:syscalls:sys_enter_*write* { @[probe] = count(); }'
# Count write syscalls by syscall type

bpftrace -e 'tracepoint:syscalls:sys_enter_read { @ = hist(args->count); }'
# Show distribution of read() syscall request sizes

bpftrace -e 'tracepoint:syscalls:sys_exit_read { @ = hist(args->ret); }'
# Show distribution of read() syscall read bytes (and errors)

bpftrace -e 't:syscalls:sys_exit_read /args->ret < 0/ { @[- args->ret] = count(); }'
# Count read() syscall errors by error code

bpftrace -e 'kprobe:vfs_* { @[probe] = count(); }'
# Count VFS calls

bpftrace -e 'kprobe:vfs_* /pid == 181/ { @[probe] = count(); }'
# Count VFS calls for PID 181

bpftrace -e 'tracepoint:ext4:* { @[probe] = count(); }'
# Count ext4 tracepoints

bpftrace -e 'tracepoint:xfs:* { @[probe] = count(); }'
# Count xfs tracepoints

bpftrace -e 'kprobe:ext4_file_read_iter { @[ustack, comm] = count(); }'
# Count ext4 file reads by process name and user-level stack

bpftrace -e 'kprobe:spa_sync { time("%H:%M:%S ZFS spa_sync()\n"); }'
# Trace ZFS spa_sync() times

bpftrace -e 'kprobe:lookup_fast { @[comm, pid] = count(); }'
# Count dcache references by process name and PID
```

## Disks

```bash
bpftrace -e 'tracepoint:block:* { @[probe] = count(); }'
# Count block I/O tracepoint events

bpftrace -e 't:block:block_rq_issue { @bytes = hist(args->bytes); }'
# Summarize block I/O size as a histogram

bpftrace -e 't:block:block_rq_issue { @[ustack] = count(); }'
# Count block I/O request user stack traces

bpftrace -e 't:block:block_rq_issue { @[args->rwbs] = count(); }'
# Count block I/O type flags

bpftrace -e 't:block:block_rq_complete /args->error/ {
    printf("dev %d type %s error %d\n", args->dev, args->rwbs, args->error); }'
# Trace block I/O errors with device and I/O type

bpftrace -e 't:scsi:scsi_dispatch_cmd_start { @opcode[args->opcode] = count(); }'
# Count SCSI opcodes

bpftrace -e 't:scsi:scsi_dispatch_cmd_done { @result[args->result] = count(); }'
# Count SCSI result codes

bpftrace -e 'kprobe:scsi* { @[func] = count(); }'
# Count SCSI driver function calls
```

## Networking

```bash
bpftrace -e 't:syscalls:sys_enter_accept* { @[pid, comm] = count(); }'
# Count socket accept(2)s by PID and process name

bpftrace -e 't:syscalls:sys_enter_connect { @[pid, comm] = count(); }'
# Count socket connect(2)s by PID and process name

bpftrace -e 't:syscalls:sys_enter_connect { @[ustack, comm] = count(); }'
# Count socket connect(2)s by user stack trace

bpftrace -e 'k:sock_sendmsg,k:sock_recvmsg { @[func, pid, comm] = count(); }'
# Count socket send/receives by direction, on-CPU PID, and process name

bpftrace -e 'kr:sock_sendmsg,kr:sock_recvmsg /(int32)retval > 0/ {
    @[pid, comm] = sum((int32)retval); }'
# Count socket send/receive bytes by on-CPU PID and process name

bpftrace -e 'k:tcp_v*_connect { @[pid, comm] = count(); }'
# Count TCP connects by on-CPU PID and process name

bpftrace -e 'k:inet_csk_accept { @[pid, comm] = count(); }'
# Count TCP accepts by on-CPU PID and process name

bpftrace -e 'k:tcp_sendmsg,k:tcp_recvmsg { @[func, pid, comm] = count(); }'
# Count TCP send/receives by on-CPU PID and process name

bpftrace -e 'k:tcp_sendmsg { @send_bytes = hist(arg2); }'
# TCP send bytes as a histogram

bpftrace -e 'kr:tcp_recvmsg /retval >= 0/ { @recv_bytes = hist(retval); }'
# TCP receive bytes as a histogram

bpftrace -e 't:tcp:tcp_retransmit_* { @[probe, ntop(2, args->saddr)] = count(); }'
# Count TCP retransmits by type and remote host (IPv4)

bpftrace -e 'k:tcp_* { @[func] = count(); }'
# Count all TCP functions (high overhead)

bpftrace -e 'k:udp*_sendmsg,k:udp*_recvmsg { @[func, pid, comm] = count(); }'
# Count UDP send/receives by on-CPU PID and process name

bpftrace -e 'k:udp_sendmsg { @send_bytes = hist(arg2); }'
# UDP send bytes as a histogram

bpftrace -e 'kr:udp_recvmsg /retval >= 0/ { @recv_bytes = hist(retval); }'
# UDP receive bytes as a histogram

bpftrace -e 't:net:net_dev_xmit { @[kstack] = count(); }'
# Count transmit kernel stack traces

bpftrace -e 't:net:netif_receive_skb { @[str(args->name)] = lhist(cpu, 0, 128, 1); }'
# Show receive CPU histogram for each device

bpftrace -e 'k:ieee80211_* { @[func] = count(); }'
# Count ieee80211 layer functions (high overhead)

bpftrace -e 'k:ixgbevf_* { @[func] = count(); }'
# Count all ixgbevf device driver functions (high overhead)

bpftrace -e 't:iwlwifi:*,t:iwlwifi_io:* { @[probe] = count(); }'
# Count all iwl device driver tracepoints (high overhead)
```