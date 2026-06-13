# Native Memory

## Footprint Overview

- __Footprint__ — total memory consumed by the JVM: Java heap + all native (non-heap) memory
- Native memory includes: JVM internals, thread stacks, code cache, metaspace, NIO buffers, native library allocations
- OS performance suffers if total footprint exceeds physical RAM (triggers swapping)

### Committed vs Reserved Memory

- __Reserved__ — virtual memory the OS promises to make available; JVM declares max sizes at startup (e.g. `-Xmx4g` reserves 4 GB)
- __Committed__ — memory actually allocated and in use; grows as the JVM expands heap/metaspace/code cache
- Performance depends on committed memory only — over-reserving is harmless on 64-bit JVMs
- On 32-bit JVMs: over-reserving is dangerous — 4 GB process limit means large heap reservation leaves little room for stacks/code cache

### Measuring Footprint

- Linux: `top`, `ps` → look at __RSS__ (resident set size) — approximate committed memory
    - PSS (newer kernels) — adjusts for pages shared with other processes; more accurate
- Windows: task manager → __working set__
- Neither includes memory committed but currently paged out

---

## Native Memory Tracking (NMT)

Enable at JVM startup only:
```bash
java -XX:NativeMemoryTracking=off|summary|detail MyApp
```

Query at runtime:
```bash
jcmd <pid> VM.native_memory summary
jcmd <pid> VM.native_memory detail         # full memory map with allocation sites
jcmd <pid> VM.native_memory summary.diff   # compare to baseline
jcmd <pid> VM.native_memory baseline       # save current state for later diff
```

Print on exit: `-XX:+PrintNMTStatistics` (default: false)

NMT is automatically disabled when the JVM is severely resource-constrained. To prevent that: `-XX:-AutoShutdownNMT`

### NMT Summary Sections

| Section | What it covers |
|---|---|
| Java Heap | Heap reserved/committed; grows with GC pressure |
| Class | Class metadata (metaspace); grows as more classes are loaded |
| Thread | Thread stacks; fully committed on thread creation (~1 MB each on 64-bit JVM) |
| Code | JIT code cache; grows as methods are compiled |
| GC | G1/CMS/other collector overhead; larger for concurrent collectors |
| Compiler | JIT compiler working memory |
| Internal | JVM internals + **direct NIO buffers** |
| Symbol | String constants from class files |
| Native Memory Tracking | NMT overhead itself |

Typical summary on a 4 GB heap JVM (512 MB initial):
- Reserved: ~5.9 GB (mostly heap + metaspace reservation)
- Committed: ~620 MB (heap actually used + everything else)

> [!NOTE]
> NMT tracks only memory allocated by the JVM engine (HotSpot). It does NOT track memory from native shared libraries (including JDK's own native libs like libzip, libnet, etc.).

### Using NMT for Tuning

- If total committed ≈ RSS → JVM fits in physical memory
- If RSS < committed → OS is paging out JVM memory → performance issue
- If one NMT section is unexpectedly large → tune its corresponding `-XX:Max*` flag

---

## Native Memory from Shared Libraries

NMT cannot see native library allocations. Signs of a native library leak:
- RSS / working set grows continuously over time
- NMT committed total is stable but OS-reported process size grows

Diagnosis approach:
- Use a native profiler (e.g. Oracle Developer Studio) with memory allocation tracing
- Native profiler shows `mmap` / `malloc` call sites in native code

### Inflater / Deflater Objects

- `Inflater` / `Deflater` (and stream classes that wrap them: `GZIPInputStream`, `ZipInputStream`, etc.) allocate native memory for their compression algorithm state
- Must call `.end()` when done (or `.close()` on the stream)
- If not called: native memory held until the object is finalized (JDK 8) or cleaned (JDK 11)
- In apps with large heaps and infrequent full GC, these objects can live in old gen for hours → native memory appears to leak

Diagnosis: take a heap dump and count `Inflater` / `Deflater` instances in the histogram

### NIO Direct Byte Buffers

```java
ByteBuffer buf = ByteBuffer.allocateDirect(1024 * 1024);  // allocates native memory
FileChannel.open(...).map(...);                            // also native memory
```

- Visible in NMT under `Internal` section
- Monitor via JMX: `java.nio:type=BufferPool,name=direct` and `java.nio:type=BufferPool,name=mapped`
- `allocateDirect()` is expensive — reuse buffers (thread-local or pool)
- Maximum total direct memory: `-XX:MaxDirectMemorySize=N` (default in JDK 11: same as `-Xmx`)

Reuse strategies:
- Thread-local: one buffer per thread (simple, but each thread ends up with max-size buffer)
- Object pool: shared pool of direct buffers (complex, allows variable sizes)
- Slicing: allocate one large buffer, slice off portions (works well only if slices are uniform size)

### Linux Native Memory Fragmentation

- Glibc's allocator partitions native memory into arenas (one per core × 8 by default)
- Native memory is never compacted → fragmentation can cause OOM despite available virtual memory
- Symptom: OOM error for native memory; `smaps` file shows many small (64 KB) allocations
- Fix: set env var `MALLOC_ARENA_MAX=2` or `4`

---

## Large Pages

### Why Large Pages Help

- OS manages memory in pages (typically 4 KB); TLB caches recent page→address mappings
- TLB has limited entries; more entries needed → more TLB misses → slower memory access
- Large pages (2 MB each) cover more memory per TLB entry → fewer misses → faster access
- JVM heap benefits significantly since it accesses memory across a large contiguous range

### Traditional (Locked) Huge Pages — Linux

Enabled in Java: `-XX:+UseLargePages` (default: false)

Setup:
```bash
# 1. Check supported page size
grep Hugepagesize /proc/meminfo
# → typically 2048 KB

# 2. Allocate enough huge pages (heap_GB × 512 pages/GB, ×1.1 for headroom)
echo 2200 > /proc/sys/vm/nr_hugepages

# 3. Persist across reboots
echo "vm.nr_hugepages=2200" >> /etc/sysctl.conf

# 4. Allow user to lock memory (/etc/security/limits.conf)
appuser soft memlock 4613734400
appuser hard memlock 4613734400

# 5. Verify
java -Xms4G -Xmx4G -XX:+UseLargePages -version
```

- If huge pages are unavailable: JVM prints warning and falls back to regular pages (no crash)
- Traditional huge pages are **locked in RAM** — can never be swapped → good for GC

### Transparent Huge Pages (THP) — Linux

- Kernel 2.6.32+; allocated on demand (not pre-reserved)
- Can be swapped → bad for GC
- Allocation can stall while kernel compacts memory → GC pause spikes

Configure at OS level:
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# → always | [madvise] | never

echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

- `always` — all processes get THP; no Java flag needed
- `madvise` — only processes that request THP; enable in Java with `-XX:+UseTransparentHugePages`
- `never` — no THP regardless of Java flags

> [!NOTE]
> THP is enabled by default to `always` on CentOS/RHEL 7 and to `madvise` on Ubuntu 18.04 LTS. Cloud images may differ. For smoothest GC pauses, use traditional (locked) huge pages rather than THP.

### Large Pages — Windows

- Available on server editions only
- Setup: MMC → Local Computer Policy → Computer Configuration → Windows Settings → Security Settings → Local Policies → User Rights Assignment → "Lock pages in memory" → add user → reboot
- Enable in Java: `-XX:+UseLargePages`
- If unsupported: JVM silently sets flag to false (no warning on non-server editions)

---

## Thread Stack Memory

- Each thread: fully committed on creation (not reserved)
- 64-bit JVM default: 1 MB per thread
- Control with: `-Xss<size>` (e.g. `-Xss512k`)
- 77 threads × 1 MB each = 77 MB of committed native memory (visible in NMT `Thread` section)
- Reduce if many threads are needed and memory is constrained; balance against stack overflow risk

---

## Key Tuning Flags Summary

| Flag | Default | Purpose |
|---|---|---|
| `-XX:NativeMemoryTracking=summary` | off | Enable NMT |
| `-XX:+UseLargePages` | false | Enable traditional huge pages |
| `-XX:+UseTransparentHugePages` | false | Enable THP (Linux, when `madvise` mode) |
| `-XX:MaxDirectMemorySize=N` | = -Xmx | Limit direct NIO buffer allocation |
| `-Xss<size>` | 1 MB (64-bit) | Thread stack size |
| `-XX:+ExitOnOutOfMemoryError` | false | Kill JVM on any OOM |
| `-XX:-AutoShutdownNMT` | true | Prevent NMT from disabling itself under stress |
