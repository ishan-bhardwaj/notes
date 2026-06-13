# Approach to Performance Testing

## Test a Real Application

### Microbenchmarks

- __Microbenchmark__ — measures a small unit of performance to compare alternate implementations
- Hard to write correctly in Java due to JIT compilation and garbage collection

#### Common Pitfalls

- __Dead code elimination__ — compiler omits code whose result is never read; use `volatile` instance variables or `Blackhole.consume()` to prevent this
- __Single input__ — testing only one input value allows the JIT to detect redundancy and skip iterations; always test a range of inputs
- __Including measurement overhead__ — generating random inputs inside the timed loop adds noise; pre-calculate inputs before the measurement loop:

```java
int[] input = new int[nLoops];
for (int i = 0; i < nLoops; i++) input[i] = random.nextInt();
long then = System.currentTimeMillis();
for (int i = 0; i < nLoops; i++) { l = fibImpl1(input[i]); }
```

- __Unrepresentative input range__ — out-of-range inputs can make a slower implementation appear faster (early throws); test with the realistic input distribution
- __Different compiler profile in production__ — JIT optimises based on call depth, argument types, and frequency; these differ between a microbenchmark and the full application
- __Different GC profile in production__ — multiple threads fill the young generation faster, promoting short-lived objects to old gen and triggering expensive full GCs

#### Threaded Microbenchmarks

- With small code, even 2 threads frequently contend on a `synchronized` block
- Results measure JVM contention handling rather than the intended behaviour
- Avoid unless threading is exactly what is being measured

#### Warm-Up Period

- JIT compiles and optimises code progressively — early executions are slower
- All benchmarks must include a warm-up phase where results are discarded
- Without warm-up, the benchmark measures compilation time rather than steady-state performance

### Macrobenchmarks

- __Macrobenchmark__ — the actual application under its real workload and full environment (real LDAP, real DB, real network)
- Only way to identify the true bottleneck across the full system
- Bottleneck example: doubling throughput of one module yields no improvement if a downstream module is already saturated
- Mocking out subsystems (DB, LDAP) changes heap usage, code compilation profiles, CPU cache behaviour, and network saturation

> [!NOTE]
> Multiple JVMs on the same host affect each other. A single JVM's GC can spike CPU to 100% on all cores. Isolation tests miss this interaction entirely.

### Mesobenchmarks

- __Mesobenchmark__ — exercises a meaningful subset of the application (e.g. a single REST endpoint without auth or session management)
- Fewer dead-code pitfalls than microbenchmarks
- More realistic threading than microbenchmarks
- Still may not reflect full production behaviour (missing auth overhead, caching, DB pools, etc.)
- Good for automated module-level regression testing
- Always verify that the metric being measured matches what matters in production (e.g. simple REST vs REST with authorisation)

---

## Throughput, Batching, and Response Time

### Elapsed Time (Batch)

- Measure total time to complete a fixed set of work
- Simple and direct — no client/server complexity
- Warm-up may or may not matter depending on whether the total duration is what the user experiences
- Caches (JPA, OS page cache) warm up during execution — second reads are faster; relevant to choosing where warm-up matters

### Throughput

- __Throughput__ — amount of work completed per unit of time (OPS / RPS / TPS)
- Zero think time: clients send the next request immediately after receiving a response
- Client must have enough CPU and threads to saturate the server; otherwise the measurement reflects client capacity
- Throughput tests use fewer client threads than response-time tests
- Average response time is reported alongside throughput but only meaningful if throughput is equal between compared runs

### Response Time

- __Response time__ — elapsed time from sending a request to receiving the response
- Clients include __think time__ (sleep between requests) to simulate real user behaviour
- Think time makes throughput approximately fixed; response time becomes the key metric

Think time vs cycle time:
- __Think time__ — sleep for a fixed interval; throughput varies slightly with response time
- __Cycle time__ — sleep for `(fixed interval − response time)`; throughput is constant regardless of response time

#### Reporting Response Times

- __Average__ — includes all requests; sensitive to outliers
- __Percentile (90th, 95th, 99th)__ — better choice; shows what most users experience; not skewed by a few extreme outliers
- GC pauses are a common source of large outliers in Java applications
- Best practice: report both average and at least one percentile; use percentile as the primary metric

---

## Variability

- Every test run produces different results due to background processes, network jitter, and random input
- Cannot determine from one run whether a difference is a real regression or random variation

### Student's t-Test

- Compare baseline and specimen by running each multiple times and computing averages
- __p-value__ — probability that the null hypothesis (baseline = specimen) is true
- __α-value__ — threshold for statistical significance; commonly 0.1 (90%), 0.05 (95%), or 0.01 (99%)
- Result is statistically significant when `p-value < α-value`
- A non-significant result is __inconclusive__, not proof of no difference
- A statistically significant result proves the values differ — it does not quantify the exact improvement with certainty
- Statistical significance ≠ statistical importance: a 1% regression with p=0.01 may matter less than a 10% regression with p=0.20

### Increasing Confidence

- More data points lower the p-value and increase confidence
- Run at least 5–10 iterations per test; more for closely matched implementations
- There is no fixed rule — balance confidence level against available time

---

## Test Early, Test Often

### Challenges

- Performance characteristics change as code evolves — early measurements may not reflect the final state
- Developers under deadline pressure may defer fixes
- Architecture regressions are cheaper to fix early before other code depends on the new implementation

### Guidelines for Automated Testing

- __Automate everything__ — scripted install, configure, run, repeat, t-test, and report; ensure the machine is in a known state before each run
- __Measure everything__ — CPU, memory, disk, network, GC logs, JFR recordings, thread stacks, heap histograms, DB AWR reports; more data = more clues for diagnosis
- __Run on the target system__ — JVM defaults are tuned to the hardware; caches, synchronisation, and GC behave differently on a 72-core server vs a laptop; extrapolations from smaller hardware are predictions, not measurements

---

## JMH (Java Microbenchmark Harness)

### Setup

```bash
mvn archetype:generate \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=net.sdo -DartifactId=my-benchmark -Dversion=1.0
mvn package
java -jar target/benchmarks.jar
```

### Basic Benchmark

```java
@Benchmark
public void testIntern(Blackhole bh) {
    for (int i = 0; i < nStrings; i++) {
        String s = new String("String to intern " + i);
        String t = s.intern();
        bh.consume(t);  // prevent dead code elimination
    }
}
```

- `@Benchmark` — marks a method for measurement
- `Blackhole.consume(value)` — prevents the compiler from eliminating a computation whose result appears unused
- jmh runs warm-up iterations (discarded) then measurement iterations (reported)
- Multiple forked JVMs (trials) test repeatability

### Parameters

```java
@Param({"1", "10000"})
private int nStrings;
```

- jmh runs the benchmark for each parameter value and reports results separately
- Override at runtime: `java -jar benchmarks.jar -p nStrings=1,1000000`

### Setup and Teardown

```java
@Setup(Level.Iteration)
public void setup() {
    strings = new String[nStrings];
    for (int i = 0; i < nStrings; i++) strings[i] = makeRandomString();
    map = new ConcurrentHashMap<>();
}
```

`Level` values:
- `Level.Trial` — once per forked JVM
- `Level.Iteration` — before each measurement/warm-up iteration
- `Level.Invocation` — before each call to the benchmark method (use rarely; distorts nanosecond-level results)

### Key CLI Options

| Option | Default | Description |
|---|---|---|
| `-f 5` | 5 | Number of forked trials |
| `-wi 5` | 5 | Warm-up iterations per trial |
| `-i 5` | 5 | Measurement iterations per trial |
| `-r 10` | 10s | Minimum length of each iteration |
| `-p key=v1,v2` | — | Parameter values |
| `-jvmArg -XX:...` | — | Extra JVM arguments for forked JVMs |

### What jmh Provides

- Warm-up isolation — results from warm-up iterations are discarded automatically
- Multiple trials in separate JVMs — tests repeatability and avoids cross-contamination
- Statistical summary — average, standard deviation, and confidence interval at 99.9%
- Prevents dead code elimination (with correct use of `Blackhole`)
- Does NOT protect against poorly designed benchmarks — still requires careful thought about what is being measured

> [!TIP]
> Pre-create input data with `@Setup` to avoid measuring input generation. Ensure both implementations under comparison have the same cache hit rate and equivalent conditions.

## OS-Level Tools

### CPU Usage

- __User time__ — CPU executing application code
- __System time__ — CPU executing kernel code (I/O, system calls, etc.)
- Goal: drive CPU usage as high as possible for as short a time as possible — more CPU = more work done in less time
- Low CPU usage means the application is idle due to:
    - Blocked on a synchronization primitive (lock contention)
    - Waiting for an external resource (DB, network)
    - No work to do (acceptable for server applications between requests)

```bash
vmstat 1   # Linux: shows CPU (us/sy/id/wa), memory, I/O, run queue each second
typeperf -si 1 "\Processor(_Total)\% Processor Time"   # Windows
```

- CPU usage is always an average over an interval — a 45% average can mean 100% busy for 450ms then 0% for 550ms
- Driving CPU lower only makes sense when a fixed load arrives and the goal is reducing resource consumption — but this usually just frees capacity for more load

### CPU Run Queue

```bash
vmstat 1   # Linux: first column 'r' = run queue (threads running OR runnable)
typeperf -si 1 "\System\Processor Queue Length"   # Windows (excludes currently running threads)
```

- Linux run queue includes all currently running threads; Windows excludes them
- Target: run queue ≤ number of CPUs on Linux; processor queue = 0 on Windows
- Sustained high run queue = machine is overloaded

### Disk Usage

```bash
iostat -xm 5   # Linux: per-device I/O stats every 5s
```

Key `iostat` fields:
- `w/s` — writes per second
- `wMB/s` — write throughput in MB/s
- `w_await` — average time (ms) to service a write
- `%util` — disk utilisation (100% = saturated)
- `%iowait` (in CPU section) — time CPUs waited for I/O

Two failure modes:
- Too few writes per MB (e.g. 24 w/s but only 0.14 MB/s) — application writing inefficiently (unbuffered)
- Disk at 100% utilisation with high `w_await` — writing more than disk can handle

- Also watch for swapping: `vmstat` columns `si` (swap in) and `so` (swap out); disk activity when no app I/O expected = likely swapping
- Swapping causes severe performance degradation; configure systems to never swap

### Network Usage

```bash
nicstat 5   # Unix: per-interface bandwidth and utilisation
netstat     # basic packet/byte counts
typeperf -si 1 "\Network Interface(*)\Bytes Total/sec"   # Windows
```

- Standard tools report bytes/packets; calculate utilisation manually: `(rKB/s + wKB/s) / interface_bandwidth_KBps`
- Network bandwidth in bits/sec; tools report bytes/sec — multiply KB/s × 8 to get Kbps
- Ethernet LAN: sustained utilisation > 40% indicates saturation
- Two failure modes: insufficient throughput (writing inefficiently) or too much throughput (saturating the interface)

---

## JDK Tools

| Tool | Description | Scripting |
|---|---|---|
| `jcmd` | Multi-purpose: VM info, GC control, thread dumps, JFR | Yes |
| `jstack` | Thread stack dump | Yes |
| `jstat` | GC and class-loading statistics | Yes |
| `jmap` | Heap dumps and heap summaries | Yes |
| `jinfo` | View and dynamically change JVM flags | Yes |
| `jconsole` | GUI: threads, classes, GC, MBeans | No |
| `jvisualvm` | GUI: monitoring, profiling, heap dump analysis | No |

> [!NOTE]
> jconsole and jvisualvm consume significant resources. Connect them to remote JVMs rather than running them on production systems. All other tools can be used inside Docker via `docker exec`.

### Basic VM Information

```bash
jcmd <pid> VM.uptime
jcmd <pid> VM.version
jcmd <pid> VM.command_line
jcmd <pid> VM.system_properties
jcmd <pid> VM.flags [-all]         # flags in effect (includes ergonomic settings)
jinfo -flags <pid>                  # all flags
jinfo -flag PrintGCDetails <pid>    # single flag value
jinfo -flag -PrintGCDetails <pid>   # disable a manageable flag at runtime

# Print all flag defaults for a given JVM config
java [other_options] -XX:+PrintFlagsFinal -version
```

Flag output format:
- `:=` — non-default value (set on command line, set ergonomically, or changed by another flag)
- `=` — default value for this JVM version
- `{product}` — uniform default across platforms
- `{pd product}` — platform-dependent default
- `{manageable}` — can be changed at runtime via `jinfo`
- `{C2 diagnostic}` — diagnostic output for compiler engineers

> [!NOTE]
> jinfo can set any flag in JDK 8 but the JVM may ignore it. Only `{manageable}` flags actually change behaviour at runtime. JDK 11 reports an error for non-manageable flags.

### Thread Information

```bash
jstack <pid>                  # thread stacks
jcmd <pid> Thread.print       # same via jcmd
```

### GC / Class Analysis

```bash
jstat -gc <pid> 1000          # GC stats every 1s
jstat -class <pid>            # class loading stats
jcmd <pid> GC.run             # trigger GC
jmap -heap <pid>              # heap summary
jmap -dump:format=b,file=heap.hprof <pid>   # heap dump
```

---

## Profiling Tools

### Sampling Profilers

- Fire a timer periodically; snapshot each thread's current method; charge that method with CPU time
- Lower overhead than instrumented profilers — better for production-like measurements
- Common errors:
    - __Timer bias__ — if timer always fires when method B is running, method A's time is missed (Figure 3-1)
    - __Safepoint bias__ — many Java profilers can only sample at JVM safepoints (lock, I/O, monitor wait, JNI); methods that never reach safepoints are under-counted

- __Async profilers__ — use `AsyncGetCallTrace` (public since Java 8) to sample at any point, not just safepoints; much less safepoint bias
    - Examples: async-profiler (open source), Oracle Developer Studio

```bash
# async-profiler (Linux)
./profiler.sh -d 30 -f profile.html <pid>   # 30s recording → flame graph
```

Interpreting profiles:
- Top methods indicate __where to look__, not necessarily what to optimise — JDK internals cannot be changed; look for how to reduce calls to them
- Flame graph — bottom-up; wider = more CPU time; interactive; produced by async-profiler
- Call tree — top-down; drill into method call chains; useful for seeing cumulative cost

### Instrumented Profilers

- Alter bytecodes at class load time to count invocations and measure time
- Higher overhead; may change inlining decisions and other JIT optimisations
- Best used for __second-level analysis__ after a sampling profiler narrows the scope
- Provide invocation counts — critical for understanding if reducing call frequency is more valuable than speeding up individual calls
- Small methods may not appear (inlining removes them); results can differ significantly from sampling profilers

### Blocking Methods and Thread Timelines

- Blocking calls (`select()`, `wait()`, `read()`) consume wall-clock time but not CPU time
- Most profilers exclude blocked time; some can be configured to include it
- Thread timeline view (e.g. Oracle Developer Studio) — shows per-thread execution and idle periods over time; reveals which threads are blocked and when

### Native Profilers

- Profile both Java code and native/JVM code in the same view
- Useful for:
    - Diagnosing unexpected GC dominance (visible as native GC threads on flame graph)
    - Spotting native memory allocation bottlenecks
    - Seeing JIT compilation thread CPU usage

---

## Java Flight Recorder (JFR)

### Overview

- Built into the JVM — lowest possible overhead (< 1% in default configuration)
- Circular event buffer — captures the most recent events up to a configured size/duration
- Available open source from JDK 11; Oracle-licensed only in JDK 8
- Events fired when: periodic timer, event duration exceeds threshold, or specific JVM condition occurs

### Enabling JFR

```bash
# Enable the JFR feature (JDK 8 Oracle also needs -XX:+UnlockCommercialFeatures)
java -XX:+FlightRecorder MyApp

# Start recording immediately at launch
java -XX:+FlightRecorder -XX:+StartFlightRecording=duration=60s,filename=rec.jfr MyApp

# Control via jcmd on a running process
jcmd <pid> JFR.start duration=60s filename=rec.jfr
jcmd <pid> JFR.dump name=1 filename=dump.jfr
jcmd <pid> JFR.check [verbose]
jcmd <pid> JFR.stop name=1
```

Recording modes:
- __Fixed duration__ — best for proactive analysis (load testing after warm-up)
- __Continuous (circular buffer)__ — best for reactive analysis (dump on unexpected event); configure with `maxage` and `maxsize`

### JFR Event Templates

- Default template: < 1% overhead; time thresholds high; profiling every 20ms
- Profile template: ~2% overhead; thresholds at 10ms; more detailed sampling
- Configurable per-event: enable/disable, threshold (ms), stack trace capture
- Templates stored as XML in `$HOME/.jmc/<release>/` or `$JAVA_HOME/jre/lib/jfr/`

### Key JFR Event Types

| Category | Basic info | JFR-specific visibility |
|---|---|---|
| Classloading | Classes loaded/unloaded | Which classloader, time per class |
| Thread statistics | Thread count, thread dumps | Threads blocked on specific locks |
| Throwables | Exception classes used | Stack trace at creation, total count |
| TLAB allocation | Heap allocations, TLAB sizes | Specific objects allocated + stack trace |
| File/socket I/O | Time in I/O | Per-call duration, specific slow files/sockets |
| Monitor blocked | Threads waiting for monitors | Specific threads, specific locks, duration |
| Code cache | Cache size and fill level | Methods removed from cache, configuration |
| Code compilation | Methods compiled, OSR | Unifies data from multiple sources |
| GC | Times, phases, generation sizes | All GC phases and configuration unified |
| Profiling | Sample-based profile | Lower fidelity than dedicated profiler; good overview |

### Java Mission Control (jmc)

- GUI for analysing JFR recordings and live JVM monitoring
- JDK 8: bundled as `jmc` with Oracle JVM; JDK 11: separate download from OpenJDK project
- Key views:
    - __Memory panel__ — heap fluctuation, GC event list with phases, TLAB allocation, promotion failures
    - __Code panel__ — hot methods, call tree, exception counts, code cache, compiler activity
    - __Event panel__ — raw event log filterable by type; most powerful view for diagnosing specific issues
    - __Thread timeline__ — per-thread execution and blocking periods

---

## Tool Selection Summary

| Goal | Recommended Tool(s) |
|---|---|
| First look at CPU / disk / network | `vmstat`, `iostat`, `nicstat` / `typeperf` |
| JVM flags in effect | `jcmd VM.flags -all` or `-XX:+PrintFlagsFinal` |
| Thread stacks / deadlocks | `jstack` or `jcmd Thread.print` |
| GC activity (live) | `jstat`, `jconsole`, `jmc` |
| Heap dump | `jmap` or `jcmd GC.heap_dump` |
| CPU hot spots | async-profiler, Oracle Developer Studio |
| Invocation counts | Instrumented profiler (jvisualvm, commercial) |
| Lock contention | JFR monitor-blocked events, native profiler |
| Allocation hot spots | JFR TLAB events, async-profiler allocation mode |
| End-to-end JVM visibility | JFR + jmc |
