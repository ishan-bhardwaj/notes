## Benchmarking

## Background

### Reasons for Benchmarking

| Reason | Description |
|--------|-------------|
| System design | Comparing systems, components, or applications — price/performance ratio for purchase decisions |
| Proofs of concept | Test performance under load before purchasing or production deployment |
| Tuning | Test tunable parameters and configuration options |
| Development | Non-regression testing and limit investigations during product development |
| Capacity planning | Determine system and application limits — data for modelling or direct capacity limits |
| Troubleshooting | Verify components can still operate at maximum performance |
| Marketing (benchmarketing) | Determine maximum product performance for marketing use |

- Cloud — large-scale environment created in minutes, tested, destroyed — very low cost
- Netflix — automated software for new instance types — micro-benchmarks + system stats + CPU profiles for comparison

### Effective Benchmarking

A good benchmark must be —
- Repeatable — facilitates comparisons
- Observable — performance can be analyzed and understood
- Portable — allows benchmarking across competitors and different product releases
- Easily presented — results understandable to everyone
- Realistic — reflects customer-experienced realities
- Runnable — developers can quickly test changes

- For purchase comparisons — also include price/performance ratio (5-year capital cost)
- Analyse while running — not after — engage performance experts during benchmark, not post-hoc
- Develop custom benchmarks when possible — shorter, easier to analyse

### Benchmarking Failures Checklist

| # | Failure | Description |
|---|---------|-------------|
| 1 | Casual benchmarking | You benchmark A, measure B, conclude C — requires rigour to avoid |
| 2 | Blind faith | Popularity ≠ validity (argumentum ad populum) — analyse tools regardless of reputation |
| 3 | Numbers without analysis | Bare results with no limit description or analysis — assume wrong until verified — spending < 1 week studying a result means it's probably wrong |
| 4 | Complex benchmark tools | Benchmark software itself becomes the bottleneck — prefer short, open-source, C language tools |
| 5 | Testing the wrong thing | Benchmark doesn't match production workload — use workload characterisation to select relevant benchmarks |
| 6 | Ignoring the environment | Test environment doesn't match production — check tunable parameters, file system, configuration |
| 7 | Ignoring errors | Benchmark produces results even with all requests erroring — always check error rate |
| 8 | Ignoring variance | Steady average-based workload ignores bursts and variance — consider Markov model for realistic variance |
| 9 | Ignoring perturbations | System backups, monitoring agents, noisy cloud neighbours skew results — use longer runs, multiple trials, check standard deviation |
| 10 | Changing multiple factors | Both systems must be identical except for the tested variable — cloud instances especially tricky |
| 11 | Benchmark paradox | 50% win probability → 12.5% chance of winning all three benchmarks — winning consistently requires actual superiority |
| 12 | Benchmarking competition | Untuned competitor results are unrealistic — spend serious time tuning competitor product before comparison |
| 13 | Friendly fire | Misconfiguration undersells own product — share results with engineering team before publication |
| 14 | Misleading benchmarks | Technically correct but misrepresented — omitting price, testing non-standard configuration |
| 15 | Benchmark specials | Engineering product specifically to score well on benchmark — not real-world performance |
| 16 | Cheating | Fake results — rare or nonexistent in practice |

## Benchmark Types

### Micro-Benchmarking

- Tests a particular type of operation — single file system I/O type, database query, CPU instruction, syscall
- Advantages — simple, easy to study, repeatable, quick to run across systems, not confused with real workload simulations
- Must be mapped to target workload — only relevant dimensions should be used

#### Example Tools by Resource

| Resource | Tool |
|----------|------|
| CPU | SysBench |
| Memory I/O | lmbench |
| File system | fio |
| Disk | hdparm, dd with direct I/O, fio with direct I/O |
| Network | iperf |

#### File System Micro-Benchmark Design Example

| Test | Intent |
|------|--------|
| Sequential 512-byte reads | Maximum realistic IOPS |
| Sequential 1 MB reads | Maximum read throughput |
| Sequential 1 MB writes | Maximum write throughput |
| Random 512-byte reads | Effect of random I/O |
| Random 512-byte writes | Effect of rewrites |

- Working set size — smaller than memory (test FS cache software) vs larger than memory (test disk I/O)
- Thread count — single-threaded (clock speed bound) vs multithreaded (maximum system performance)
- Sunny day testing — tests top speeds
- Cloudy/rainy day testing — tests contention, perturbations, workload variance

### Simulation (Macro-Benchmarking)

- Simulates customer application workloads — based on production workload characterisation
- Includes complex system interactions missed by micro-benchmarks
- Stateless simulation — each request independent — operations chosen by measured probability
- Stateful simulation — requests dependent on client state — Markov model for realistic state transitions
- Problem — ignores variance — customer usage patterns change over time

#### HTTP Load Tools

- `wrk`, `siege`, `hey` — HTTP load generation tools for evaluating software/hardware changes

### Replay

- Replays captured trace log to target — sounds ideal but problematic
- When server characteristics change — captured client workload does not respond naturally
- Risk — tests wrong level (Eg - disk I/O replay misses file system cache benefits of new system)

### Industry Standards

#### TPC (Transaction Processing Performance Council)

| Benchmark | Description |
|-----------|-------------|
| TPC-C | Simulation of complete computing environment — users executing database transactions |
| TPC-DS | Decision support system — queries and data maintenance |
| TPC-E | OLTP workload — brokerage firm database |
| TPC-H | Decision support — ad hoc queries and concurrent data modifications |
| TPC-VMS | Virtual machine database workloads |
| TPCx-HS | Big data — Hadoop |

- Results shared online — include price/performance

#### SPEC (Standard Performance Evaluation Corporation)

| Benchmark | Description |
|-----------|-------------|
| SPEC Cloud IaaS 2018 | Provisioning, compute, storage, network — multi-instance workloads |
| SPEC CPU 2017 | Compute-intensive — integer and floating point |
| SPECjEnterprise 2018 | Java EE Web Profile — full-system performance |
| SPECsfs2014 | NFS/CIFS file access workload simulation |
| SPECvirt_sc2013 | Virtualised environments — end-to-end performance |

- Results shared online — include tuning details and component list — not usually price

## Methodology

### Passive Benchmarking (Anti-Method)

- Fire-and-forget — execute benchmark, ignore until complete, collect data
- Problems — results may be invalid (software bugs), limited by benchmark itself (single-threaded), limited by unrelated component, misconfigured, perturbed, testing wrong thing entirely

### Active Benchmarking

- Analyse performance while benchmark is running — use observability tools in parallel
- Confirm what the benchmark is actually testing — identify true limiters
- Include limit details with benchmark results
- Run in steady state for hours or days to allow thorough analysis

#### Active Benchmarking Case Study (bonnie++ analysis)

```bash
# bonnie++ claims to test "hard drive performance"
# First test: Sequential Output, Per Chr — reports 739 KB/s

# Check if disk I/O is actually happening
iostat -sxz 1          # no disk I/O reported

# Verify at block layer
bpftrace -e 'tracepoint:block:* { @[probe] = count(); }'
# no block_rq_issue or block_rq_complete — confirms no disk I/O

# Check file system cache
cachestat 1            # shows "dirties" — writes to FS cache, not disk

# Check VFS level
bpftrace -e 'kprobe:vfs_* /comm == "bonnie++"/ { @[probe] = count(); }'
# shows 1,176,936 vfs_write() calls

# Check write size
bpftrace -e 'k:vfs_write /comm == "bonnie++"/ { @bytes = hist(arg2); }'
# size = 1 byte (668,839 writes)
```

Conclusion — bonnie++ Per Chr test writes 1-byte file system writes to the FS cache — does not test disk I/O despite claiming to test "hard drive performance"

### CPU Profiling

- Profile both benchmark target and benchmark software — quick way to discover unexpected behaviour
- Often reveals hidden resource controls or unexpected code paths
- Eg - ZFS I/O throttle (`zfs_zone_io_throttle()`) found consuming 62% of CPU during disk benchmark via flame graph — was artificially throttling the result

### USE Method

- Apply USE method during benchmarking — ensure a limit is found
- Either some component reaches 100% utilisation OR the system has not been driven to its limit

### Custom Benchmarks

- Keep as short as possible — avoids complexity that hinders analysis
- C language preferred for micro-benchmarks — maps closely to execution
- Be aware of compiler optimisations — may elide benchmark routines if output is unused
- Disassemble compiled binary to verify what will actually execute
- Custom load generators — generate load only, leave measurements to other tools

### Ramping Load

- Simple method for determining maximum throughput
- Increment load in small steps — measure delivered throughput — plot scalability profile
- Stop when limit is reached — profile shows knee point and ceiling

```perl
#!/usr/bin/perl
# Example load generator — 8 KB random reads
my $IOSIZE = 8192;
while (1) {
    seek(FILE, int(rand($span / $IOSIZE)) * $IOSIZE, 0);
    sysread(FILE, $junk, $IOSIZE);
}
```

- Measure latency distribution as well as throughput — queueing delays increase at saturation
- Choose maximum result where latency is still acceptable — not the absolute peak IOPS with 10x latency

### Sanity Check

- Check whether any component would have needed to exceed its known limits
- Simple example — 50,000 NFS IOPS × 8 KB = 400 MB/s = 3.2 Gbits/s — over 1 Gbit/s interface limit → result is bogus

#### Common Bogus Results Over Single 1 Gbit/s Interface

| Claimed Throughput | Actual Gbits/s | Status |
|-------------------|----------------|--------|
| 120 MB/s | 0.96 Gbits/s | Plausible |
| 200 MB/s | 1.6 Gbits/s | Bogus (unidirectional) |
| 350 MB/s | 2.8 Gbits/s | Clearly bogus |
| 800 MB/s | 6.4 Gbits/s | Clearly bogus |

### Statistical Analysis

- Three phases — select benchmark + metrics, execute and collect large dataset, interpret with statistics
- Used when access to max-config system is time-limited and expensive
- Collect variation, full distributions, error margins alongside primary metrics
- Collect all available system statistics (/proc, sar, monitoring) for forensic analysis
- Statistical tools — scalability analysis (Amdahl's Law, USL), queueing theory

### Benchmarking Checklist

| Question | What It Catches |
|----------|----------------|
| Why not double? | Forces identification of the true limiter |
| Did it break limits? | Sanity check — detects cached/short-circuited results |
| Did it error? | Errors skew results — all-error tests report timeout latency as "performance" |
| Does it reproduce? | Identifies perturbations and variance |
| Does it matter? | Confirms benchmark relates to production workload |
| Did it even happen? | Firewall blocking, caching substituting, workload never reaching target |

## Benchmark Questions

### General

- Does the benchmark relate to my production workload
- What was the configuration of the system under test
- Single system or cluster
- Cost of system under test
- Duration of test and number of results collected
- Average or peak — what is the average, standard deviation, percentiles
- What was the limiting factor
- Success/fail ratio of operations
- Were operation attributes chosen to simulate a workload and how
- Does the benchmark simulate variance or average workload
- Was the result confirmed using other analysis tools (screenshots)
- Is the result reproducible — what is the error margin

### CPU and Memory Benchmarks

- Processor type and count
- Were processors overclocked or using custom cooling
- Memory module count, type, attachment to sockets
- Were any CPUs disabled
- System-wide CPU utilisation during test (low utilisation → higher turbo boost)
- Cores or hyperthreads
- Main memory size and type
- Custom BIOS settings

### Storage Benchmarks

- Storage device configuration — count, type, protocol, RAID, cache size, write-back vs write-through
- File system configuration — type, journaling, tuning
- Working set size — how much cached and where
- Number of files accessed

### Network Benchmarks

- Network interface configuration — count, type, configuration
- Network topology
- Protocols and socket options used
- Network stack tunables — TCP/UDP settings