# JIT Compiler

## JIT Compilation Overview

- Java source → bytecodes (via `javac`) → machine code (via JIT at runtime)
- Gives portability of interpreted languages + performance of compiled code
- Compilation is deferred until code is "hot enough" — two reasons:
    - Code executed only once is faster to interpret than compile
    - More executions = more profiling data = better optimisations (type specialisation, inlining, escape analysis, etc.)

### Register vs Memory Optimisation

- JIT keeps frequently used values in CPU registers rather than main memory
- Main memory access = multiple cycles; register access = one cycle
- Instance variables like `sum` in a loop are loaded once into a register and written back at the end
- This is why thread synchronisation is critical: other threads cannot see register values

---

## Tiered Compilation

### Compilation Levels

| Level | Name | Description |
|---|---|---|
| 0 | Interpreted | Bytecode interpreted directly |
| 1 | Simple C1 | C1 compiled, no profiling |
| 2 | Limited C1 | C1 compiled with invocation/back-edge counters |
| 3 | Full C1 | C1 compiled with full profile feedback |
| 4 | C2 | C2 compiled with aggressive optimisations |

- Typical path: 0 → 3 → 4
- If C2 queue is full: 0 → 2 → 3 → 4 (C1 compiles quickly, later re-compiles with profile, then C2)
- If C1 queue is full: 0 → 2 → 4 (skip full C1)
- Trivial methods: may stop at level 1
- C2 fails to compile: falls back to level 1

### C1 vs C2

- __C1 (client compiler)__ — compiles sooner with less information; faster startup
- __C2 (server compiler)__ — waits longer, gathers more profile data, produces faster code
- `-client` / `-server` flags are no-ops since JDK 8; `-d64` removed in JDK 11
- Both are always active in tiered compilation; disable with `-XX:-TieredCompilation` (default: true)

### Tiered Compilation Trade-offs

When to disable tiered compilation (`-XX:-TieredCompilation`):
- Memory-constrained environments (Docker containers, VMs with tight limits)
- Reduces code cache usage by ~4× (compiled ~4× fewer classes in example)
- Long-running programs reach similar throughput after warm-up
- Startup time increases (68.5s vs 50.1s in example)

---

## Code Cache

- Fixed-size region holding compiled machine code — once full, no more compilation
- Warning when full: `CodeCache is full. Compiler has been disabled.`
- Default max size: 240 MB (`-XX:ReservedCodeCacheSize=N`)
- Initial size: 2.5 MB (`-XX:InitialCodeCacheSize=N`)

In JDK 11, code cache split into three segments:
- Nonmethod code — sized by compiler thread count (~5.5 MB on 4-CPU machine)
- Profiled code — equally shares remaining space (~117 MB)
- Nonprofiled code — equally shares remaining space (~117 MB)

Individual segment flags: `-XX:NonNMethodCodeHeapSize=N`, `-XX:ProfiledCodeHeapSize=N`, `-XX:NonProfiledCodeHeapSize=N`

Monitor with: `jconsole` Memory panel → Code Cache chart, or Native Memory Tracking (Ch8)

> [!NOTE]
> Reserving a large code cache is safe on 64-bit machines — memory is reserved but not committed until needed. On 32-bit JVMs the 4 GB process limit applies.

---

## Inspecting the Compiler

### PrintCompilation Output

Enable with: `-XX:+PrintCompilation` (default: false)

Format:
```
timestamp  compilation_id  attributes  [tiered_level]  class::method  size  [deopt_message]
```

Attribute characters (5-char field):
- `%` — OSR (on-stack replacement) compilation
- `s` — method is `synchronized`
- `!` — method has exception handler
- `b` — compiled in blocking mode (not seen in normal operation)
- `n` — native method wrapper

- __OSR (on-stack replacement)__ — loop compiled while it is still running; JVM replaces the loop body mid-execution on the next iteration
- Compilation IDs may appear out of order due to multiple compilation threads and scheduling

### Reading Deoptimisation Lines

```
timestamp  id  [%]  class::method @ -2 (size bytes)  made not entrant
timestamp  id  [%]  class::method @ -2 (size bytes)  made zombie
```

### jstat for Compiler Info

```bash
jstat -compiler <pid>        # summary: compiled count, failures, time
jstat -printcompilation <pid> 1000   # last compiled method, repeated every 1s
```

COMPILE SKIPPED reasons:
- `Code cache filled` — increase `ReservedCodeCacheSize`
- `Concurrent classloading` — class modified during compilation; will retry (normal)

---

## Deoptimisation

Two triggers for making code `not entrant`:

### 1. Type Assumption Invalidation

- JIT inlines/optimises based on observed types (e.g., `obj.equals()` always called on `String`)
- When a different type is observed, a __deoptimisation trap__ fires
- Previous compiled code made `not entrant`; new code compiled with updated assumptions
- Performance impact is usually negligible — code is quickly recompiled

### 2. Tiered Compilation Replacement

- When C2 replaces C1 code, old C1 code is made `not entrant`
- Identify by checking tier level in log: `made not entrant` on level 3 with level 4 following = normal
- This "deoptimisation" is actually an improvement

### Zombie Code

- After `not entrant` code is marked, when all objects using it are GC'd, code becomes `zombie`
- Zombie code is removed from the code cache, freeing space
- If the class is loaded again and gets hot, it will be recompiled

---

## Compilation Thresholds

- Compilation triggered when: `invocation_count + back_edge_count ≥ threshold`
- __Back-edge count__ = number of loop iterations completed
- Counters decay over time (reduced at safepoints) — measure recent hotness, not total executions
- Lukewarm methods that never reach threshold won't be compiled by C2 alone; tiered compilation ensures even lukewarm methods get C1-compiled

Flags (rarely need tuning with tiered compilation enabled):
- `-XX:CompileThreshold=N` (default 10,000) — only effective with tiered compilation disabled
- `-XX:Tier3InvocationThreshold=N` (default 200) — C1 compilation trigger
- `-XX:Tier4InvocationThreshold=N` (default 5000) — C2 compilation trigger

> [!NOTE]
> `-XX:CompileThreshold` has no effect when tiered compilation is enabled (JDK 8+). Recommendations to set it are holdovers from JDK 7 and earlier.

---

## Compilation Threads

- Separate queues for C1 and C2; processed by background threads
- Queue is priority-ordered by invocation counter — hotter methods compiled first (another reason compilation IDs appear out of order)
- `-XX:CICompilerCount=N` — total compiler threads; 1/3 go to C1 queue, 2/3 to C2 queue (minimum 1 each)

Default thread counts (tiered compilation):

| CPUs | C1 threads | C2 threads |
|---|---|---|
| 1 | 1 | 1 |
| 4 | 1 | 2 |
| 8 | 1 | 2 |
| 16 | 2 | 6 |
| 64 | 4 | 8 |

When to tune `CICompilerCount`:
- Docker containers with JDK 8 — JVM sees host CPU count, not container CPU limit; set manually
- Single-CPU VMs — reducing to 1 thread can improve warm-up by reducing CPU contention
- Multiple JVMs per host — reduce threads to avoid overwhelming the system
- Excess CPU available — increasing threads may speed warm-up (rarely worth it for long-running apps)

- `-XX:+BackgroundCompilation` (default: true) — compilation is async; set false to block until compiled
- `-Xbatch` — equivalent to `-XX:-BackgroundCompilation`

---

## Inlining

- Most impactful single optimisation — eliminates method call overhead for small methods
- Enabled by default; disable with `-XX:-Inline` (causes > 50% performance regression in practice)
- Getters/setters are inlined automatically — no need to avoid encapsulation for performance

Inlining decision rules:
- Hot method (called frequently) → inline if bytecode size < 325 bytes (`-XX:MaxFreqInlineSize=N`, default 325)
- Any method → inline if bytecode size < 35 bytes (`-XX:MaxInlineSize=N`, default 35)

Tuning `MaxInlineSize` beyond 35 only benefits warm-up (cold methods inlined earlier); hot methods would be inlined anyway. Rarely worth changing for long-running applications.

Debug visibility: `-XX:+PrintInlining` (requires debug JVM build)

---

## Escape Analysis

- Enabled by default: `-XX:+DoEscapeAnalysis` (default: true)
- C2 detects objects that do not "escape" the current method or thread
- Optimisations applied to non-escaping objects:
    - Skip synchronisation lock acquisition on `synchronized` methods
    - Store fields in registers instead of heap memory
    - Eliminate object allocation entirely (scalar replacement) — fields kept in registers

```java
for (int i = 0; i < 100; i++) {
    Factorial f = new Factorial(i);      // may not be allocated on heap
    list.add(f.getFactorial());          // lock on getFactorial() may be skipped
}
```

- Occasionally causes incorrect behaviour (historical bugs); if suspected, simplify the code rather than disabling
- This is why microbenchmarks can produce misleading results — dead objects get optimised away

---

## CPU-Specific Code

- JIT detects CPU and emits the best available instruction set automatically
- Intel AVX (Advanced Vector Extensions): `-XX:UseAVX=N`
    - 0 — no AVX
    - 1 — AVX level 1 (Sandy Bridge+)
    - 2 — AVX level 2 (Haswell+)
    - 3 — AVX-512 (Knights Landing+); JDK 8 does not support; JDK 11 ≥ 11.0.6 defaults to 2
- Intel SSE: `-XX:UseSSE=N` (1–4); stable and rarely needs tuning
- Use the latest patch version of JDK 8 or 11 — important CPU optimisation fixes are backported

---

## AOT (Ahead-of-Time) Compilation

- Experimental feature (JDK 9+ Linux; JDK 11 all platforms)
- Precompiles selected methods to a shared library loaded at JVM startup
- Targets long-startup applications (REST servers) — offsets library load time

```bash
# 1. Capture all touched methods
java -XX:+UnlockDiagnosticVMOptions -XX:+LogTouchedMethods \
     -XX:+PrintTouchedMethodsAtExit MyApp > touched.txt

# 2. Build compile commands file
# (prepend "compileOnly" to each method line, remove colon before args)

# 3. Precompile
jaotc --compile-commands=methods.txt \
      --output=MyLib.so \
      --compile-for-tiered \
      --module java.base

# 4. Run with precompiled library
java -XX:AOTLibrary=MyLib.so MyApp
java -XX:+PrintAOT ...   # shows when AOT methods are loaded and used
```

- `--compile-for-tiered` — allows AOT-compiled methods to still be recompiled by C2; essential for long-running servers
- Precompile only the subset of classes the application actually uses (not everything)

---

## GraalVM

### As JIT Compiler Replacement

- GraalVM includes a new C2-replacement compiler written in Java
- Enable in standard JVM (experimental): `-XX:+UnlockExperimentalVMOptions -XX:+EnableJVMCI -XX:+UseJVMCICompiler`
- EE (Enterprise Edition) version is faster than the bundled CE version
- JDK 13+ bundles a newer Graal build that approaches C2 performance

### Native Compilation

- Produces a fully native binary that runs without the JVM
- Very fast startup; much lower initial memory footprint
- Trade-off: no C2-level runtime optimisations; long-running programs run slower than traditional JVM
- Limitations: no dynamic class loading (`Class.forName()`), no finalizers, no Security Manager, no JMX/JVMTI, reflection/proxies/JNI require special configuration

---

## javac Flags That Do NOT Affect Performance

- `-g` (debug info in bytecode) — no performance impact
- `final` keyword — no faster compiled code
- Recompiling with a newer `javac` — generally no performance change
- Exception: JDK 11 introduces faster string concatenation, but only when bytecodes are recompiled
