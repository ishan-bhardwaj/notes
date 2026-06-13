# Performance Methodologies

## Terminology

| Term | Definition |
|------|------------|
| IOPS | Input/output operations per second |
| Throughput | Rate of work performed — data rate (bytes/s) or operation rate (ops/s) depending on context |
| Response time | Time for an operation to complete — includes wait time, service time, and result transfer |
| Latency | Time an operation spends waiting to be serviced — sometimes used interchangeably with response time |
| Utilization | How busy a resource is based on time actively performing work — or capacity consumed (for storage) |
| Saturation | Degree to which a resource has queued work it cannot service |
| Bottleneck | Resource that limits system performance |
| Workload | Input to the system — Eg - database queries sent by clients |
| Cache | Fast storage area that buffers a limited amount of data to avoid slower storage tiers |

## Models

### System Under Test (SUT)

- Performance affected by perturbations — scheduled system activity, other users, other workloads
- Cloud environments — other tenant activity on physical host may not be observable from guest SUT
- Modern environments composed of multiple networked components — mapping the environment helps reveal overlooked perturbation sources
- Environment can be modelled as a network of queueing systems

### Queueing System

- Components and resources modelled as queueing systems — predicts response time degradation under load
- Studied by queueing theory (see Modelling section)

## Concepts

### Latency

- Time spent waiting before an operation is performed
- Differs from response time — response time = latency + operation time
- Must include qualifying terms to avoid ambiguity — Eg - request latency, TCP connection latency
- Time-based — enables comparison across components and calculation of predicted speedup

### Time Scales

| Event | Latency | Scaled (1 CPU cycle = 1s) |
|-------|---------|--------------------------|
| 1 CPU cycle | 0.3 ns | 1 s |
| L1 cache access | 0.9 ns | 3 s |
| L2 cache access | 3 ns | 10 s |
| L3 cache access | 10 ns | 33 s |
| Main memory (DRAM) | 100 ns | 6 min |
| SSD I/O (flash) | 10–100 μs | 9–90 hours |
| Rotational disk I/O | 1–10 ms | 1–12 months |
| SF to New York (Internet) | 40 ms | 4 years |
| SF to United Kingdom | 81 ms | 8 years |
| SF to Australia | 183 ms | 19 years |
| TCP timer-based retransmit | 1–3 s | 105–317 years |
| SCSI command time-out | 30 s | 3 millennia |
| Physical system reboot | 5 min | 32 millennia |

### Trade-Offs

- Good/fast/cheap — pick two — most IT projects choose on-time and inexpensive, fixing performance later
- CPU vs memory — memory can cache results to reduce CPU — or CPU compresses to reduce memory
- File system record size — small = better for random I/O and cache efficiency — large = better for streaming and backups
- Network buffer size — small = lower memory overhead per connection — large = higher throughput

### Tuning Efforts

- Most effective when done closest to where work is performed — application level first

| Layer | Example Tuning Targets |
|-------|----------------------|
| Application | Application logic, request queue sizes, database queries |
| Database | Table layout, indexes, buffering |
| System calls | Memory-mapped vs read/write, sync vs async I/O flags |
| File system | Record size, cache size, journaling tunables |
| Storage | RAID level, number and type of disks, storage tunables |

- Application-level fix — can yield 20x improvement
- Storage-level fix — typically improves application performance by percentages only (20%)
- Application changes push to production frequently — large performance wins and regressions found regularly in application code changes
- OS performance analysis can identify application-level issues — not just OS issues

### Level of Appropriateness

- ROI for performance expertise determines depth of analysis justified
- Large data centres/cloud environments — performance teams analyse kernel internals and CPU PMCs
- Small startups — superficial checks may be sufficient
- Extreme environments (stock exchanges, HFT) — 6 ms latency reduction justified $300M transatlantic cable

### When to Stop Analysis

- When the bulk of the performance problem is explained — quantify findings before stopping
- When potential ROI is less than cost of analysis — some issues measurable in hundreds of dollars not worth even an hour
- When bigger ROIs exist elsewhere — prioritise by potential impact

### Point-in-Time Recommendations

- Performance characteristics change over time — hardware upgrades, more users, software changes
- Tunable parameter values are valid only at a specific point in time
- Internet-sourced tunable values — verify appropriateness for your system and workload before applying
- Store tunable changes in version control with detailed history

### Load vs Architecture

- Architecture issue — application limited by design (Eg - single-threaded with idle CPUs, single lock contention)
- Load issue — application doing everything right but workload exceeds capacity
- Cloud solution for load issues — scale out with more instances

### Scalability

- Linear scalability — performance increases proportionally with load
- Knee point — boundary where contention begins degrading throughput
- Saturation point — component reaches 100% utilisation — throughput may begin decreasing
- Beyond peak — context switching and coherency overhead causes throughput to decrease even as load increases

### Performance Metrics

| Metric Type | Description |
|-------------|-------------|
| Throughput | Operations or data volume per second |
| IOPS | I/O operations per second |
| Utilisation | Resource busyness as percentage |
| Latency | Operation time — average or percentile |

- Overhead — gathering metrics consumes CPU cycles — affects performance of measured target (observer effect)
- Metrics can be wrong — bugs, stale implementations not updated with new code paths

### Utilisation

- __Time-based__ — `U = B/T` where B = busy time, T = observation period — most commonly available from OS tools
    - 100% time-based utilisation — busy 100% of time but may still accept more work (Eg - storage array with idle disks, elevator accepting passengers)
- __Capacity-based__ — proportion of maximum throughput capacity in use — 100% means no more work accepted
- This book uses time-based (non-idle time) for most discussions
- `100% busy ≠ 100% capacity`

### Saturation

- Begins at 100% capacity-based utilisation — extra work queues
- Any degree of saturation is a performance issue — time spent waiting
- For time-based utilisation — queueing may not begin exactly at 100% depending on parallel work capability

### Profiling

- Sampling the state of the system at timed intervals — builds a picture of target activity
- Coarser view than exact metrics — resolution depends on sample rate
- CPU profiling — samples instruction pointer or stack trace — shows code paths consuming CPU

### Caching

- Hit ratio = hits / (hits + misses) — higher is better
- Non-linear improvement — difference between 98% and 99% much greater than between 10% and 11%
- Miss rate (misses/s) — proportional to performance penalty — easier to compare workloads
- `runtime = (hit rate × hit latency) + (miss rate × miss latency)`

#### Cache Algorithms

| Algorithm | Description |
|-----------|-------------|
| MRU (Most Recently Used) | Retention policy — keep most recently used |
| LRU (Least Recently Used) | Eviction policy — remove least recently used |
| MFU (Most Frequently Used) | Retention based on frequency |
| LFU (Least Frequently Used) | Eviction based on frequency |
| NFU (Not Frequently Used) | Inexpensive but less thorough LRU approximation |

- Cache states — cold (empty/unwanted), warm (useful but low hit ratio), hot (high hit ratio — Eg - >99%)

### Known-Unknowns

| Type | Description |
|------|-------------|
| Known-knowns | Things you know and are currently checking (Eg - CPU at 10%) |
| Known-unknowns | Things you know you could check but haven't yet (Eg - CPU profiling not yet done) |
| Unknown-unknowns | Things you don't know you don't know (Eg - device interrupts as CPU consumers) |

- More you learn about systems → more unknown-unknowns become known-unknowns

## Perspectives

### Resource Analysis

- Bottom-up — starts with system resources (CPUs, memory, disks, network, buses)
- Audience — system administrators responsible for physical resources
- Activities — performance issue investigations, capacity planning
- Best metrics — IOPS, throughput, utilisation, saturation

### Workload Analysis

- Top-down — examines application performance and workload applied
- Audience — application developers and support staff
- Targets — requests (workload applied), latency (response time), completion (errors)
- Best metrics — throughput (transactions/s), latency
- Latency is the most important metric for expressing application performance

## Methodologies

### Anti-Methodologies

#### Streetlight Anti-Method

- Analysing performance using only familiar or random tools — not deliberate
- Named after streetlight effect — looking where the light is, not where the keys were lost
- May find an issue but not the issue — lacks quantification to rule out false positives

#### Random Change Anti-Method

- Randomly changing tunables until performance improves
- Time-consuming — changes not properly understood may cause worse problems under peak load
- Tuning may work around a bug later fixed — leaves unexplained and potentially harmful configuration

#### Blame-Someone-Else Anti-Method

- Finding a component you're not responsible for and redirecting the issue
- Characterised by lack of data supporting the hypothesis
- Counter — ask the accuser for screenshots of tools run and output interpretation

### Problem Statement

- First methodology to use for any new performance issue
- Questions to ask —
    - What makes you think there is a performance problem
    - Has this system ever performed well
    - What changed recently (software, hardware, load)
    - Can the problem be expressed in terms of latency or runtime
    - Does the problem affect other people or applications
    - What is the environment — software, hardware, versions, configuration

### Scientific Method

- Question → Hypothesis → Prediction → Test → Analysis
- Observational test — measure a metric predicting it will differ if hypothesis is correct
- Experimental test — apply a change predicting performance will improve (or worsen for negative tests)
- Change only one factor at a time — revert change if it doesn't fix the issue before trying the next

### Diagnosis Cycle

- Hypothesis → Instrumentation → Data → Hypothesis (repeat)
- Move from hypothesis to data quickly — discard bad theories early

### Tools Method

- List available tools → list useful metrics per tool → list interpretations per metric
- Weakness — relies exclusively on known/available tools — may have incomplete view without knowing it
- Does not guide toward custom tooling (dynamic tracing) for unknown-unknowns
- Time-consuming when many overlapping tools exist

### USE Method

- For every resource, check utilisation, saturation, and errors
- Purpose — identify systemic bottlenecks early in investigation

#### Definitions

| Term | Definition |
|------|------------|
| Resources | Physical server functional components (CPUs, buses, memory, network, storage) — and some software resources |
| Utilisation | Percentage of time resource was busy servicing work over an interval |
| Saturation | Degree of extra work resource can't service — often waiting on a queue (pressure) |
| Errors | Count of error events |

#### Procedure

- Check errors first — quick, objective, easy to interpret
- Check saturation second — any non-zero saturation is a problem
- Check utilisation last — 100% is usually a bottleneck sign but depends on resource type

#### Expressing Metrics

| Metric | Expression |
|--------|------------|
| Utilisation | Percentage over interval — Eg - "CPU at 90% utilisation" |
| Saturation | Wait-queue length — Eg - "average run-queue length of 4" |
| Errors | Count — Eg - "50 disk errors" |

> [!NOTE]
> Short bursts of 100% utilisation can cause saturation even when the overall average is low. 5-minute averages can mask seconds-long periods of 100% CPU utilisation and associated scheduler latency.

#### Resource List

- CPUs — sockets, cores, hardware threads
- Main memory — DRAM
- Network interfaces — Ethernet ports, Infiniband
- Storage devices — disks, storage adapters
- Accelerators — GPUs, TPUs, FPGAs
- Controllers — storage, network
- Interconnects — CPU, memory, I/O
- Caches — usually excluded — they improve performance rather than degrade it under high utilisation

#### USE Method Metrics Reference

| Resource | Type | Metric |
|----------|------|--------|
| CPU | Utilisation | Per-CPU or system-wide average |
| CPU | Saturation | Run queue length, scheduler latency, CPU pressure (PSI) |
| Memory | Utilisation | Available free memory (system-wide) |
| Memory | Saturation | Anonymous paging, page scanning, OOM events, memory pressure (PSI) |
| Network interface | Utilisation | RX throughput/max bandwidth, TX throughput/max bandwidth |
| Storage device I/O | Utilisation | Device busy percent |
| Storage device I/O | Saturation | Wait queue length, I/O pressure (PSI) |
| Storage device I/O | Errors | Device soft/hard errors |

#### Advanced USE Metrics

| Resource | Type | Metric |
|----------|------|--------|
| CPU | Errors | Machine check exceptions, CPU cache errors |
| Memory | Errors | Failed `malloc()` (rare on Linux with overcommit) |
| Network | Saturation | Linux "overruns" |
| Storage controller | Utilisation | Current IOPS/throughput vs maximum |
| CPU interconnect | Utilisation | Per-port throughput/max bandwidth (CPU PMCs) |
| Memory interconnect | Saturation | Memory stall cycles, high CPI (CPU PMCs) |
| I/O interconnect | Utilisation | Bus throughput/max bandwidth (PMCs) |

#### Software Resource USE

| Software Resource | Utilisation | Saturation | Errors |
|------------------|-------------|------------|--------|
| Mutex locks | Time lock was held | Threads queued waiting for lock | N/A |
| Thread pools | Time threads were busy | Requests waiting to be serviced | N/A |
| Process/thread capacity | Current count vs limit | Threads waiting for allocation | Allocation failures |
| File descriptor capacity | In-use FDs vs limit | Threads blocked waiting | `EFILE` "Too many open files" |

#### Suggested Interpretations

- Utilisation at 100% — usually a bottleneck sign — confirm with saturation and application impact
- Utilisation above 60% — can hide short bursts at 100% — may begin to cause queueing delays
- Any saturation (non-zero) — is a performance issue — measured as queue length or wait time
- Non-zero errors — investigate especially when increasing while performance is poor
- Low utilisation, no saturation, no errors — confirms resource is not the problem — narrows investigation

### RED Method

- For every service, check request rate, errors, and duration
- Designed for microservices / cloud services

| Metric | Description |
|--------|-------------|
| Request rate | Number of service requests per second |
| Errors | Number of failed requests |
| Duration | Time for requests to complete (percentiles, not just average) |

- Complementary to USE — USE for machine health, RED for user health
- Rising duration with steady request rate — architecture problem (the service itself)
- Rising duration and rising request rate — load problem — investigate further with workload characterisation

### Workload Characterisation

- Focuses on input to the system rather than resulting performance
- Questions to answer —
    - Who is causing the load — PID, UID, remote IP
    - Why is the load being called — code path, stack trace
    - What are the load characteristics — IOPS, throughput, direction, type, variance
    - How is the load changing over time — daily patterns

- Best performance wins come from eliminating unnecessary work
- Even expected workloads should be verified — surprises happen (Eg - DoS attacks presenting as normal client IPs)
- Identifies load vs architecture issues
- Input for simulation benchmark design — collect distribution and variation, not just averages

### Drill-Down Analysis

- Start at high level — narrow based on findings — dig into interesting areas

#### Three Stages

| Stage | Description | Example Tools |
|-------|-------------|--------------|
| Monitoring | Continuously recording high-level statistics — alerting | Atlas, Prometheus |
| Identification | Narrowing to specific resources or areas — finding possible bottlenecks | vmstat, iostat, mpstat, GUI dashboards |
| Analysis | Deeper examination for root cause — tracing and profiling | perf, BCC, bpftrace, Ftrace |

#### Five Whys

- Ask "why?" five times — each answer becomes the next question
- Persistently drilling down leads to root cause rather than symptoms

### Latency Analysis

- Examines time taken to complete an operation — subdivides into smaller components
- Continues subdividing the highest-latency components — binary search of latency
- Drills down through software stack layers
- Each step divides latency into two parts — proceed with the larger part

### Static Performance Tuning

- Focuses on issues of configured architecture — performed when system is at rest
- Check each component —
    - Does the component make sense (outdated, underpowered)
    - Does the configuration make sense for the intended workload
    - Was it autoconfigured in the best state
    - Has the component experienced an error causing degraded state

- Common finds — NIC negotiating at 1 Gbps instead of 10 Gbps, failed disk in RAID, older firmware, full file system, mismatched record size, debug mode left enabled, IP forwarding accidentally enabled, remote authentication instead of local

### Cache Tuning

- Aim to cache as high in the stack as possible — closer to where work is performed — more metadata available for retention policy
- General steps —
    - Check cache is enabled and working
    - Check hit/miss ratios and miss rate
    - Check current cache size (if dynamic)
    - Tune cache for workload
    - Tune workload for cache — reduce unnecessary consumers
    - Watch for double caching — same data cached twice consuming main memory

### Micro-Benchmarking

- Tests simple and artificial workloads — fewer factors than macro-benchmarks
- Use a micro-benchmark tool or separate load generator + observability tools
- Example targets — syscall time, file system read performance, network throughput
- Average time = runtime / operation count

### Performance Mantras

| Mantra | Example |
|--------|---------|
| Don't do it | Eliminate unnecessary work |
| Do it, but don't do it again | Caching |
| Do it less | Reduce polling/refresh frequency |
| Do it later | Write-back caching |
| Do it when they're not looking | Schedule work during off-peak hours |
| Do it concurrently | Switch from single-threaded to multi-threaded |
| Do it more cheaply | Buy faster hardware |

## Modelling

### Visual Identification of Scalability Profiles

| Profile | Description |
|---------|-------------|
| Linear scalability | Performance increases proportionally — may be early stage of another profile |
| Contention | Shared serial resources reduce effectiveness of scaling |
| Coherence | Data coherency tax outweighs benefits of scaling |
| Knee point | Factor encountered at a specific scale changes the profile |
| Scalability ceiling | Hard limit reached — device bottleneck or software-imposed limit |

### Amdahl's Law of Scalability

- C(N) = N / (1 + α(N - 1))

- C(N) — relative capacity, N — scaling dimension (CPU count, user load), α — degree of seriality (0–1)
- α = 0 → linear scalability — α = 1 → no scalability

### Universal Scalability Law (USL)

C(N) = N / (1 + α(N - 1) + βN(N - 1))

- Adds β — coherence parameter — models performance degradation from coherency delay
- β = 0 → reduces to Amdahl's Law
- Models the coherence scalability profile — throughput can decrease beyond a peak N

### Queueing Theory

- Mathematical study of systems with queues — analyses queue length, wait time (latency), utilisation
- Little's Law — `L = λW` — average number in system = arrival rate × average time in system

#### Kendall's Notation (A/S/m)

| Code | Meaning |
|------|---------|
| A | Arrival process — M (Markovian/exponential), D (deterministic/fixed) |
| S | Service time distribution — M (exponential), D (deterministic), G (general/any) |
| m | Number of service centres |

#### Common Queueing Systems

| Model | Description | Use |
|-------|-------------|-----|
| M/M/1 | Exponential arrivals, exponential service, 1 server | General analysis |
| M/M/c | M/M/1 but multiserver | Parallel resources |
| M/G/1 | Exponential arrivals, general service, 1 server | Rotational hard disks |
| M/D/1 | Exponential arrivals, deterministic service, 1 server | Disk performance modelling |

#### M/D/1 Response Time

- r = s(2 - ρ) / 2(1 - ρ)
- r = response time, s = service time, ρ = utilisation
- Beyond 60% utilisation — average response time doubles
- At 80% — response time has tripled
- Disk I/O latency often the bounding resource — doubling latency significantly hurts application performance
- This is why disk utilisation becomes a problem well before 100% — unlike CPUs where preemption allows higher-priority work

## Capacity Planning

### Resource Limits
- CPU% per request = total CPU% / requests

max requests/s = 100% × CPU count / CPU% per request

- Measure current request rate and resource utilisation
- Express requests in terms of resource usage
- Extrapolate to known resource limits
- Resources to monitor — CPU, memory, disk IOPS, disk throughput, disk capacity, network throughput, virtual memory, processes/threads, file descriptors

### Factor Analysis

- Test maximum configuration first
- Change one factor at a time — measure performance drop
- Attribute percentage performance drop and cost saving to each factor
- Choose factors to save cost while maintaining required performance
- Retest final configuration

### Scaling Solutions

| Strategy | Description |
|----------|-------------|
| Vertical scaling | Larger systems — more CPU, memory |
| Horizontal scaling | Spread load across multiple systems behind load balancer |
| Cloud auto-scaling | Automatically add/remove instances based on metric (Eg - AWS ASG, Kubernetes HPA) |
| Sharding | Split database data by logical key — spreads load across multiple database instances |

> [!TIP]
> Netflix targets 60% CPU utilisation for ASGs — scales up and down with load to maintain that target. This accounts for the M/D/1 response time degradation beyond 60%.

## Statistics

### Quantifying Performance Gains

#### Observation-Based

- Estimated gain: original latency → (original - component latency + new component latency)

Eg - 10 ms → (10 - 9 + 0.01) = 1.01 ms ≈ 9x gain

- Ensure latency is a synchronous component of the application request — async events don't directly affect performance

#### Experimentation-Based

- Apply fix → measure before vs after → calculate ratio

### Averages

| Average Type | Description | Use Case |
|-------------|-------------|----------|
| Arithmetic mean | Sum / count | General purpose |
| Geometric mean | nth root of product | Multiplicative effects (Eg - layered performance improvements) |
| Harmonic mean | Count / sum of reciprocals | Averaging rates (Eg - transfer rates) |

- Averages hide distribution details — always ask "what is the distribution?"
- Monitoring tools using 5-minute averages mask seconds-long spikes at 100% utilisation

### Standard Deviation, Percentiles, Median

- Standard deviation — measure of variance from the mean — larger = greater spread
- Percentiles (90th, 95th, 99th, 99.9th) — quantify slowest requests — used in SLAs
- Median (50th percentile) — where bulk of data is — less affected by outliers than mean

### Coefficient of Variation (CoV)

- CoV = standard deviation / mean
- Expresses variance as a single metric — lower = less variance

### Multimodal Distributions

- System performance often bimodal — Eg - cache hits vs misses, sequential vs random I/O
- Average of bimodal distribution is not the index of central tendency — can be seriously misleading
- Always inspect full distribution — use histograms, heat maps
- Eg - disk I/O with cache hits <1 ms and cache misses ~7 ms — average 3.3 ms falls in the gap between modes

### Outliers

- Very small number of extreme values — Eg - occasional disk I/O >1000 ms when majority is 0–10 ms
- Difficult to identify from averages — may shift mean slightly but not median
- 99th percentile may identify them depending on frequency
- Best approach — inspect full distribution via histogram or heat map

## Monitoring

### Time-Based Patterns

| Pattern | Cause |
|---------|-------|
| Every 5–60 minutes | Monitoring and reporting tasks |
| Daily | Work hours activity, nightly log rotation, backups |
| Weekly | Weekday vs weekend usage |
| Quarterly | Financial reporting |
| Yearly | School schedules, vacations |

- Irregular spikes — new content releases, sales events (Black Friday)
- Irregular drops — power outages, sports finals

### Monitoring Architecture

- Agents/exporters — run on each system — collect and publish metrics
- Centralised monitoring — aggregate across hundreds or thousands of instances
- Eg - Netflix Atlas — 200,000+ instances — open source cloud-wide monitoring

### Summary-Since-Boot

- If monitoring not available — OS may provide summary-since-boot values from kernel counters
- Coarse but better than no baseline

## Visualisations

### Line Chart

- Shows metric over time — passage of time on x-axis
- Multiple lines — show related data on same axes (Eg - per-disk latency)
- Statistical lines — add median, standard deviation, percentiles to show distribution

### Scatter Plot

- Each event drawn as a point — completion time on x-axis, latency on y-axis
- Shows all data — reveals outliers invisible in averages
- Scalability limit — points overlap at high density — becomes unreadable ("wall of paint")

### Heat Map

- Quantises x and y ranges into buckets (columns) — coloured by event count
- Solves scatter plot scalability — works for millions of events from one or thousands of systems
- High-latency outliers — appear as light-coloured blocks high in the heat map
- Reveals distribution patterns impossible to see in scatter plots at scale
- Invented for latency, utilisation, and offset metrics — now standard in monitoring products (Grafana)

### Timeline Chart (Waterfall Chart)

- Activities as bars on a timeline — shows timing and duration
- Front-end performance — network requests in browser (Firefox, Chrome devtools)
- Back-end performance — thread or CPU timelines (KernelShark, Trace Compass)
- With dependency arrows — becomes a Gantt chart

### Surface Plot (Wireframe)

- Three-dimensional representation — works best with smoothly varying third dimension
- Eg - per-CPU utilisation across data centre — 16 CPUs × 60 seconds per server
- Color and hue can add fourth and fifth dimensions of data