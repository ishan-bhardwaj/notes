## Cloud Computing

## Background

### Instance Types

- Hardware instances (IaaS) — hardware virtualisation — each instance is a VM
- OS instances — lightweight OS virtualisation — containers
- Cloud providers — AWS, Azure, GCP

#### AWS Instance Family Examples

| Family | Optimised For |
|--------|--------------|
| m5 | General purpose (balanced) |
| c5 | Compute |
| i3, d2 | Storage |
| r4, x1 | Memory |
| p1, g3, f1 | Accelerated computing (GPUs, FPGAs) |

- Instance type is now a tunable parameter — can be modified as needed
- Fine-grained sizing — Eg - m5.large (2 vCPU, 8 GB) to m5.24xlarge (96 vCPU, 384 GB)

### Scalable Architecture

- Horizontal scalability — spread load across many small systems
- Each environment layer — one or more parallel instances — more added to handle load
- Cloud-native databases — Cassandra, CockroachDB, Amazon Aurora, DynamoDB
- Traditional databases — shard data across multiple instances for parallel scaling

### Capacity Planning

- Cloud instances — inexpensive, created and destroyed almost instantly
- React to real load instead of planning upfront — scale automatically via cloud API
- Key activities — dynamic sizing (auto scaling), scalability testing (short-duration large-scale benchmark)

#### Dynamic Sizing (Auto Scaling)

- container CPU (shares) = all CPUs × container shares / total busy shares on system
- AWS ASG — auto scaling group — scales based on metric (Eg - 60% CPU utilisation)
- Kubernetes HPA — Horizontal Pod Autoscaler — scales Pod replicas
- Risk — overprovisioning — DoS attack or regression triggers expensive instance increases
- Netflix — adds/removes tens of thousands of instances daily to match streams-per-second pattern

### Storage

| Storage Type | Example | Access |
|-------------|---------|--------|
| File store | Amazon EFS | NFS |
| Block store | Amazon EBS | iSCSI |
| Object store | Amazon S3 | HTTP API |

- Instance storage — volatile (ephemeral) — destroyed with instance
- Network storage — higher latency than local disk — mitigated by in-memory caches
- Provisioned IOPS — purchase reliable performance — Eg - Amazon EBS Provisioned IOPS

### Multitenancy

- __Noisy neighbours__ — other tenants consuming resources cause performance issues
- Resource isolation — per-tenant limits or priorities for CPU, memory, disk I/O, network
- Degree of observability depends on virtualisation type

### Orchestration (Kubernetes)

- Kubernetes (k8s) — open source container orchestration — deploys containers as co-located groups (Pods)
- Pod — containers sharing resources and localhost — own IP address
- Kubernetes service — stable abstraction for Pod group endpoints — allows Pods to be treated as disposable
- Node — physical machine in a Kubernetes cluster

#### Kubernetes Networking Performance

- CNI (Container Network Interface) — pluggable networking — Eg - Calico (netfilter/iptables), Cilium (BPF)
- kube-proxy — load balancing — can be replaced with BPF (Cilium) for better performance at scale
- Large iptables rule sets (thousands of services) — first-packet overhead for kube-proxy

## Hardware Virtualisation

### Hypervisor Configurations

#### Config A (Native/Bare-Metal)

- Hypervisor runs directly on processors — Eg - Xen
- Creates domains — schedules guest vCPUs onto real CPUs
- Privileged domain (dom0) administers other domains

#### Config B (Host OS-Based)

- Hypervisor executed by host OS kernel — Eg - KVM
- Host OS schedules VM CPUs alongside other processes
- Uses kernel modules for direct hardware access

### Implementations

| Hypervisor | Type | Notes |
|-----------|------|-------|
| VMware ESX | Bare-metal | Enterprise — microkernel — paravirt optimised over years |
| Xen | Config A | Research origin — paravirt for high performance — PVHVM config |
| Hyper-V | Type 1 | Windows Server 2008 — used by Azure |
| KVM | Config B | Kernel module — paired with QEMU — used by GCP |
| Nitro | Config B-like | AWS — KVM-based + hardware SR-IOV — no QEMU proxy — near bare-metal performance |

### CPU Overhead

#### Virtualisation Types

| Method | Description | Overhead |
|--------|-------------|---------|
| Binary translation | Guest kernel instructions identified and translated at runtime — used before hardware support | High |
| Paravirtualisation | Guest OS modified to use hypercalls — aware it is virtualised | Medium |
| Hardware-assisted (VT-x/AMD-V) | Guest privileged instructions trap to VMM at ring level below 0 | Low |

- Guest exits — vCPU stops executing in guest to handle hypervisor event — key overhead metric
- Events handled in kernel = less overhead — events requiring userspace (QEMU) = more overhead
- KVM exit types — `MSR_READ`, `MSR_WRITE`, `HLT`, `IO_INSTRUCTION`, `EPT_VIOLATION`, `EXTERNAL_INTERRUPT`
- `perf kvm stat live` — shows guest exit types, sample counts, and timing statistics

### Memory Overhead

- Page fault — requires two-step translation: guest virtual → guest physical → host physical
- EPT (Intel) / NPT (AMD) — hardware MMU virtualisation — hardware page walk without hypervisor call
- Shadow page tables — hypervisor maintains guest-virtual-to-host-physical mappings — intercepts CR3 changes
- Memory size — each guest runs own kernel (some overhead) — KVM also runs QEMU process per VM

### I/O Overhead

- Historically largest overhead source — every device I/O translated by hypervisor
- Paravirtualised drivers — coalesce I/O, fewer interrupts — reduce hypervisor overhead
- PCI pass-through — device assigned directly to guest — best performance — reduces flexibility
- SR-IOV (Single Root I/O Virtualisation) — hardware support — direct guest device access with sharing
- Xen — device channel (async shared memory transport) — avoids extra copy of I/O data
- Nitro — full hardware support — eliminates I/O proxies — near bare-metal I/O performance

#### I/O Path Comparison

| Hypervisor | I/O Path | Notes |
|-----------|----------|-------|
| Xen | Guest → dom0 backend → I/O proxy → hardware | Can use isolated driver domains (IDDs) for performance isolation |
| KVM | Guest → host QEMU proxy → hardware | QEMU process per guest |
| Nitro | Guest → hardware (direct via SR-IOV) | No proxy — fewest steps |

### Resource Controls

#### CPU

- vCPUs — coarsely limit CPU resource usage by count
- Xen schedulers — BVT (fair-share), SEDF (real-time guarantees), Credit-based (weights + caps)
- KVM — host OS cgroup CPU bandwidth controls
- Intel CAT — LLC partitioning between guests — prevents cache pollution

#### Memory

- Guest configured with fixed memory limit — guest kernel handles paging within its limit
- Balloon driver (VMware, Xen, KVM) — inflates module inside guest to reclaim memory for hypervisor — can cause performance issues when active

#### Network

- KVM — host kernel cgroups and qdiscs applied to guest network interfaces
- Nitro — I/O limits enforced by external hardware systems (AWS EC2)

### Observability

#### From Privileged Guest/Host

- All physical resources observable with standard OS tools
- Xen — `xentop(1)` shows per-domain CPU%, memory, VCPUS, NETTX/RX, VBD stats
- KVM — guest appears as QEMU process — `pidstat -tp PID` shows per-vCPU threads (`CPU 0/KVM`, `CPU 1/KVM`)
- `perf kvm stat live` — guest exit types, timing, percentages
- KVM tracepoints — `kvm:kvm_exit`, `kvm:kvm_entry` — instrumentable with bpftrace
- `kvmexits.bt` — shows exit reasons as histograms with durations

#### From Guest

- Only virtual devices visible (unless SR-IOV/pass-through)
- `vmstat` `st` column — stolen CPU time — time unavailable to guest (other tenants or hypervisor functions)
- PMCs — may or may not be available depending on hypervisor config — Xen has virtual PMU (vpmu)
- BPF, perf, Ftrace — all work in hardware VM guests — dedicated kernel with root access
- `biosnoop` — shows virtual disk device latency from inside guest

## OS Virtualisation (Containers)

### Origins and Implementation

- Unix `chroot(8)` → FreeBSD jails → Solaris Zones → Linux namespaces + cgroups
- Linux — no kernel notion of "container" — containers are namespaces + cgroups + seccomp-bpf
- Namespaces first added — Linux 2.4.19 (2002) — cgroups first added — Linux 2.6.24 (2008)

### Linux Namespaces

| Namespace | Description |
|-----------|-------------|
| `cgroup` | cgroup visibility |
| `ipc` | Interprocess communication visibility |
| `mnt` | File system mount visibility |
| `net` | Network stack isolation — interfaces, sockets, routes |
| `pid` | Process visibility — filters `/proc` |
| `time` | Separate system clocks per container |
| `user` | User ID isolation |
| `uts` | Host information — `uname(2)` syscall — hostname |

```bash
lsns    # list all namespaces on system
```

### Linux cgroups (v1)

| cgroup | Description |
|--------|-------------|
| `blkio` | Limits block I/O — bytes and IOPS |
| `cpu` | Limits CPU usage based on shares |
| `cpuacct` | Accounting for CPU usage |
| `cpuset` | Assigns CPU and memory nodes |
| `memory` | Limits process memory, kernel memory, and swap |
| `net_cls` | Sets classids on packets for qdiscs/firewalls |
| `net_prio` | Sets network interface priorities |
| `perf_event` | Allows perf to monitor cgroup processes |
| `pids` | Limits number of processes that can be created |

> [!NOTE]
> cgroups v2 — unified hierarchy — released as stable in Linux 4.5 — migration in progress. Fedora 31 switched to v2 in 2019.

### Advantages vs Hardware VMs

- Fast initialisation — milliseconds vs seconds/minutes
- No extra kernel memory per guest
- Unified file system cache — avoids double-caching
- Fine-grained resource sharing (cgroups)
- Host operators — guest processes directly visible
- Can share memory pages for common files — improves page cache efficiency
- CPUs are real CPUs — adaptive mutex assumptions remain valid

### Disadvantages vs Hardware VMs

- Increased contention for kernel resources — locks, caches, buffers, queues
- Guest — reduced performance observability — kernel typically cannot be analysed
- Kernel panic affects all guests
- No custom kernel modules
- No different kernel versions or kernels
- Less secure — shared kernel

### Overhead

- CPU — no direct overhead in user mode — performance degraded primarily by contention
- Memory mapping — no overhead
- I/O — depends on configuration — overlayfs adds layers to file system I/O — bridge networking for network I/O
- Low-IOPS workloads (<1000 IOPS) — overlayfs overhead negligible

#### Multi-Tenant Contention

- CPU caches — lower hit ratio — other tenants consuming and evicting entries
- TLB caches — lower hit ratio — other tenant usage and context switch flushing
- CPU execution — interrupted for other tenant device interrupt service routines
- Kernel contention — buffers, caches, queues, locks — multi-tenant increases load by order of magnitude
- Network I/O — CPU overhead from iptables for container networking

### Resource Controls

| Resource | Priority | Limit |
|----------|---------|-------|
| CPU | CFS shares | cpusets (whole CPUs), CFS bandwidth (fractional) |
| Memory | Soft limits | Hard limits |
| Disk I/O | blkio weights | blkio IOPS/throughput limits |
| Network I/O | net_prio priorities | qdiscs (tbf, fq) |

#### CPU Shares Formula

- container CPU = all CPUs × container shares / total busy shares on system
- Bursting — container can use idle CPU from other containers — up to 100% on idle system
- Bursting problem — end users test on idle system (100% CPU) — later new tenants arrive (10% CPU) — feels like regression
- CFS bandwidth — sets upper limit to bound bursting range — Eg - shares minimum 10% to bandwidth maximum 20%

- Kernel settings: quota of CPU microseconds every period microseconds

Container exposed as: percentage of whole CPUs (Eg - 2.5 = two and a half CPUs)

#### Memory cgroup Settings

| Setting | Description |
|---------|-------------|
| `memory.limit_in_bytes` | Hard limit — hits swap or OOM killer |
| `memory.soft_limit_in_bytes` | Best-effort steering towards limit |
| `memory.kmem.limit_in_bytes` | Kernel memory limit |
| `memory.kmem.tcp.limit_in_bytes` | TCP buffer memory limit |

#### blkio cgroup Settings

| Setting | Description |
|---------|-------------|
| `blkio.weight` | Share weight for BFQ scheduler |
| `blkio.weight_device` | Per-device weight |
| `blkio.throttle.read_bps_device` | Read throughput limit |
| `blkio.throttle.write_bps_device` | Write throughput limit |
| `blkio.throttle.read_iops_device` | Read IOPS limit |
| `blkio.throttle.write_iops_device` | Write IOPS limit |

### Observability

#### Traditional Tools Behaviour

| Tool | From Host | From Container |
|------|-----------|----------------|
| `top` | Shows all host and container processes | Summary shows mixed host/container stats — process table shows container processes |
| `ps` | Shows all processes | Shows container processes |
| `uptime` | Host load averages | Host load averages |
| `mpstat` | Host CPUs and usage | Host CPUs and host CPU usage |
| `vmstat` | Host CPUs, memory, stats | Host CPUs, memory, stats |
| `free` | Host memory | Host memory |
| `iostat` | Host disks | Host disks |
| `sar -n DEV` | Host network interfaces | Container network interfaces and TCP stats |
| `perf` | Can profile everything | Fails to run (permissions), or may profile other tenants |
| `tcpdump` | Can sniff all interfaces | Only container interfaces |
| `dmesg` | Kernel log | Fails to run |

> [!NOTE]
> Traditional tools show host statistics (not container-specific) by default. This is a major observability pitfall — a container may appear busy due to other tenants' activity.

#### From Host

```bash
kubectl top nodes                       # host CPU and memory usage
kubectl top pods                        # per-pod CPU and memory
docker stats                            # per-container CPU, memory, net I/O, block I/O
systemd-cgtop                           # cgroup hierarchy with CPU, memory, I/O
nsenter -t PID -m -p top                # run top inside container's namespaces
```

- PID namespace mapping —

```bash
grep NSpid /proc/4915/status     # host PID → container PID
awk '$1 == "NSpid:" && $3 == 753 { print $2 }' /proc/*/status   # container PID → host PID
ls -lh /proc/4915/ns/pid         # verify namespace ID matches
ls -lh /proc/4915/root/tmp       # browse container's /tmp from host
```

#### cgroup Statistics

```bash
cat /sys/fs/cgroup/cpu,cpuacct/docker/CONTAINER_ID/cpu.stat
# nr_periods — number of enforcement intervals
# nr_throttled — times throttled
# throttled_time — total throttled time (nanoseconds)

cat /sys/fs/cgroup/blkio/blkio.throttle.io_serviced     # per-device I/O operation counts
cat /sys/fs/cgroup/blkio/blkio.throttle.io_service_bytes  # per-device I/O byte counts
```

#### Container CPU Throttling Analysis (Decision Flow)

Is throttled_time increasing?

YES → CPU bandwidth-limited (CFS quota hit)

NO → Are non-voluntary context switches increasing?

YES → Is host CPU idle?

YES → Are cpuset CPUs 100% busy?

YES → cpuset-limited

NO → Bug/unexpected

NO → Share-limited (competing with other tenants)

NO → Not CPU throttled

#### BPF Container Observability

```bash
# Break down forks by container (using UTS namespace nodename)
bpftrace -e '
#include <linux/sched.h>
#include <linux/nsproxy.h>
#include <linux/utsname.h>
tracepoint:syscalls:sys_enter_clone {
    $task = (struct task_struct *)curtask;
    $nodename = $task->nsproxy->uts_ns->name.nodename;
    @new_processes[$nodename] = count();
}'
```

> [!NOTE]
> UTS namespace nodename only works for syscall tracing (process context). For async events (Eg - disk I/O completion interrupts), the originating process may not be on-CPU and `curtask` will not identify the correct container.

#### Tracing Tools in Containers

- `perf`, Ftrace, BPF — typically fail from inside container due to permission requirements (`perf_event_open(2)`, `bpf(2)`)
- Hardware VM guests — all tracing tools work (dedicated kernel, root access)
- Lightweight VM guests — all tracing tools work (dedicated kernel, root access)

## Lightweight Virtualisation

- Best of both worlds — security of hardware VMs + efficiency and fast boot of containers
- Lightweight hypervisor — assumes modern CPU virtualisation features — no BIOS, video, audio, PCI emulation needed
- QEMU — 1.4M+ lines of code — Firecracker — ~50K lines of code

### Implementations

| Implementation | Year | Notes |
|---------------|------|-------|
| Intel Clear Containers | 2015 | Boot time < 45 ms — merged into Kata Containers |
| Kata Containers | 2017 | Intel Clear Containers + Hyper.sh RunV — OpenStack governance |
| Google gVisor | 2018 | User-space kernel in Go — closer to container behaviour |
| Amazon Firecracker | 2019 | KVM + lightweight VMM — boot time ~100 ms — < 5 MB memory overhead |

### Overhead

- Similar to KVM — lower memory footprint — VMM is much smaller
- Firecracker — < 5 MB per VM
- Intel Clear Containers 2.0 — 48–50 MB per container

### Observability

#### From Host

- VM appears as single process (Eg - `firecracker`)
- Cannot see guest internals — cannot see which guest processes consume CPU
- All physical resources observable with standard OS tools

#### From Guest

- Own dedicated kernel — all tracing tools work (BPF, perf, Ftrace)
- `mpstat` shows only guest vCPUs — not host CPUs
- Statistics are guest-only — different from containers where host stats bleed through
- Key advantage vs containers — guests control their own kernel stats

## Comparison

| Attribute | Hardware VMs (KVM) | OS Virtualisation (Containers) | Lightweight (Firecracker) |
|-----------|-------------------|-------------------------------|--------------------------|
| CPU performance | High (hardware support) | High | High (hardware support) |
| CPU allocation | Fixed vCPUs | Flexible (shares + bandwidth) | Fixed vCPUs |
| I/O throughput | High (SR-IOV) | High (no intrinsic overhead) | High (SR-IOV) |
| I/O latency | Low (SR-IOV, no QEMU) | Low | Low (SR-IOV) |
| Memory overhead | Some (EPT/NPT, extra kernels) | None | Some (EPT/NPT, extra kernels) |
| Memory allocation | Fixed (possible double-caching) | Flexible (unused guest memory used for page cache) | Fixed (possible double-caching) |
| Resource controls | Most (kernel + hypervisor) | Many (depends on kernel) | Most (kernel + hypervisor) |
| Host observability | Medium — no guest internals | High — see everything | Medium — no guest internals |
| Guest observability | High — full kernel + device inspection | Medium — user mode only, host stats bleed through | High — full kernel + device inspection |
| Observability favours | End users | Host operators | End users |
| Hypervisor complexity | Highest | Medium (OS) | High (lightweight hypervisor) |
| Different OS guests | Yes | Usually no | Yes |
| Boot time | Seconds | Milliseconds | ~100 ms |

> [!TIP]
> Observability often enables performance wins far greater than minor hypervisor differences. Hardware VMs and lightweight VMs give end users full kernel access — enabling all chapters 13–15 tools. Containers give host operators full visibility but restrict guest kernel analysis. Choose virtualisation type based on who needs the observability: operators (containers) or end users (VMs).

## Other Cloud Types

| Type | Description | Observability |
|------|-------------|--------------|
| FaaS (serverless) | Developer submits function — runs on demand — no server to manage | Limited to application-provided timestamps |
| SaaS | High-level software — no server access for end users | Limited to operators |
| Unikernels | Application + minimal kernel compiled as single binary — no OS | No OS tools — requires custom tooling — hypervisor profiling |
