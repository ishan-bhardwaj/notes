# Java Performance — Chapter 7: Heap Memory Best Practices

## Heap Analysis

### Heap Histograms

```bash
jcmd <pid> GC.class_histogram           # forces full GC, then shows live objects
jcmd <pid> GC.class_histogram -all      # skip full GC; includes dead objects
jmap -histo <pid>                       # includes dead objects
jmap -histo:live <pid>                  # forces full GC first; live only
```

- Shows instance count and total bytes per class, sorted by total bytes
- Expect `[C` (char[]), `String`, `[B` (byte[]), `[Ljava.lang.Object;` near the top — normal
- Look for application-specific classes with unexpectedly high counts
- Fast to obtain; useful in automated test pipelines

### Heap Dumps

```bash
jcmd <pid> GC.heap_dump /path/to/dump.hprof    # forces full GC by default
jcmd <pid> GC.heap_dump -all /path/to/dump.hprof  # include dead objects
jmap -dump:live,file=/path/to/dump.hprof <pid>  # forces full GC
```

Automated flags:
- `-XX:+HeapDumpOnOutOfMemoryError` — dump on OOM (default: false)
- `-XX:HeapDumpPath=<path>` — output directory or filename
- `-XX:+HeapDumpAfterFullGC` — dump after every full GC
- `-XX:+HeapDumpBeforeFullGC` — dump before every full GC

Tools for analysis:
- `jvisualvm` — Monitor tab; browse heap; run queries
- Eclipse MAT (Memory Analyzer Tool) — open source; reports, dominator tree, histogram, SQL-like queries; best for comparing two dumps

### Key Analysis Concepts

- __Shallow size__ — size of the object itself (references counted as 4/8 bytes; not their targets)
- __Deep size__ — size of object + all objects it references (including shared ones)
- __Retained size__ — size of object + all objects that would be freed if this object were GC'd (excludes shared objects with other live references)
- __Dominators__ — objects that retain large amounts of heap; freeing one frees many others

Analysis workflow:
1. Look at dominator tree — if a few objects dominate, address them directly
2. Look at histogram — large counts of one type (e.g. `TreeMap$Entry`) indicate a leak
3. Trace back references — find where the collection holding the objects is rooted
4. Compare two dumps (taken minutes apart) to identify growing object counts

> [!TIP]
> Start with collection objects (`HashMap`, `TreeMap`) rather than their entries. Look for the biggest collections, not the smallest retained sizes.

---

## Out-of-Memory Errors

### Four Causes

| Error Message | Cause | Solution |
|---|---|---|
| `unable to create new native thread` | OS thread/process limit exceeded OR no native memory for stack | Check `ulimit -u`; reduce thread count or increase OS limit |
| `Metaspace` | Metaspace full (classloader leak or insufficient size) | Set `-XX:MaxMetaspaceSize`; diagnose classloader leak via heap dump |
| `Java heap space` | Heap full (true OOM or memory leak) | Increase heap; diagnose leak via heap dump comparison |
| `GC overhead limit exceeded` | Spending too much time in GC | Increase heap; diagnose memory leak |

### GC Overhead Limit Conditions (all four must be true)

- `>= 98%` of time spent in full GC (tunable: `-XX:GCTimeLimit=N`, default 98)
- `< 2%` of heap freed per full GC (tunable: `-XX:GCHeapFreeLimit=N`, default 2)
- Condition persists for 5 consecutive full GC cycles (not tunable)
- `-XX:+UseGCOverheadLimit` is true (default)

As a last resort before the 5th full GC: all soft references are freed (may prevent the error)

### OOM Behaviour

- An OOM in one thread does not kill the JVM — other threads continue
- Memory used by the failed thread becomes eligible for GC
- Server frameworks catch OOM and keep the thread alive; request memory is still freed
- To exit on any OOM: `-XX:+ExitOnOutOfMemoryError` (default: false)

---

## Using Less Memory

### Reducing Object Size

Object header overhead (64-bit JVM, heap < 32 GB):
- Regular object: 16 bytes header
- Array: 16 bytes header
- All objects padded to 8-byte boundaries

Type sizes:
| Type | Bytes |
|---|---|
| `byte` | 1 |
| `char`, `short` | 2 |
| `int`, `float` | 4 |
| `long`, `double` | 8 |
| reference (64-bit, compressed oops) | 4 |
| reference (64-bit, no compressed oops) | 8 |

Practical implications:
- `null` instance variables still consume space for the reference field (but not for the referenced object)
- Object size always rounded up to next multiple of 8 — adding a field may cost 0 extra bytes if it fills existing padding
- Prefer `float` over `double`, `int` over `long`, `byte` over `int` where the value range permits
- Use `jol` (OpenJDK tool) to measure actual object sizes

Time vs space trade-off:
- Caching a computed value (e.g. `String.hashCode()`) → saves CPU but uses memory → justified if reused frequently
- Recalculating on demand (e.g. `toString()`) → saves memory but uses CPU → justified if rarely used

### Lazy Initialization

```java
// Eager (always allocates)
private Calendar calendar = Calendar.getInstance();

// Lazy (allocates only when needed)
private Calendar calendar;
private void report(Writer w) {
    if (calendar == null) calendar = Calendar.getInstance();
    w.write(...);
}
```

- Best when the variable is used in fewer than ~50% of object lifetimes
- Slight performance penalty on each use (null check)
- If variable is always needed → no memory saving, only overhead → don't use lazy init

Thread-safe lazy init (simple — synchronize on method):
```java
private synchronized void report(Writer w) {
    if (calendar == null) calendar = Calendar.getInstance();
    ...
}
```

Thread-safe lazy init for thread-safe objects (double-checked locking):
```java
private volatile ConcurrentHashMap instanceChm;

public void doOperation() {
    ConcurrentHashMap chm = instanceChm;
    if (chm == null) {
        synchronized(this) {
            chm = instanceChm;
            if (chm == null) {
                chm = new ConcurrentHashMap();
                instanceChm = chm;
            }
        }
    }
    // use chm
}
```

- `volatile` is mandatory for double-checked locking to work correctly
- Assign instance variable to local variable for slight performance gain (avoids repeated volatile read)

Eager deinitialization (`= null`) — only useful when:
- A long-lived collection discards items it no longer needs
- Eg - `ArrayList.remove()` sets `elementData[--size] = null` to avoid stale reference

### Canonical (Deduplication) Objects

- __Immutable objects__ with the same value can share a single canonical instance
- Eg - `Boolean.TRUE` / `Boolean.FALSE` — never create `new Boolean(...)`, use constants
- `String.intern()` — returns canonical string (see Ch12)

For custom immutable classes:
```java
public class ImmutableObject {
    private static WeakHashMap<ImmutableObject, WeakReference<ImmutableObject>> map =
        new WeakHashMap<>();

    public ImmutableObject canonicalVersion(ImmutableObject io) {
        synchronized(map) {
            WeakReference<ImmutableObject> ref = map.get(io);
            ImmutableObject canonical = (ref != null) ? ref.get() : null;
            if (canonical == null) {
                map.put(io, new WeakReference<>(io));
                canonical = io;
            }
            return canonical;
        }
    }
}
```

- Use `WeakHashMap` to prevent the cache from becoming a memory leak
- Synchronisation may become a bottleneck at high concurrency — use a concurrent weak map from JSR-166 extras

---

## Object Life-Cycle Management

### Object Reuse — When It Makes Sense

Object reuse hurts GC efficiency because:
- Reused objects get promoted to old gen and stay there
- Large old gen with many live objects → slower full GC / concurrent marking
- Single full GC that frees very little takes much longer than one that frees most of the heap

Object reuse is justified when:
- Initialisation cost is very high (not just allocation cost)
- The pool is small and bounded
- Eg - JDBC connections, threads, `SecureRandom`, `MessageDigest`, large `byte[]`, NIO direct buffers

### Object Pools

Pros:
- Throttles access to scarce resources (JDBC connections, threads)
- Avoids repeated expensive initialisation

Cons:
- Synchronisation contention on pool access can be slower than just creating a new object
- Developer must return objects to the pool
- Hard to size correctly
- Many live objects → slow GC

### Thread-Local Variables

```java
ThreadLocal<SimpleDateFormat> sdf = ThreadLocal.withInitial(SimpleDateFormat::new);
ThreadLocalRandom tlr = ThreadLocalRandom.current();  // built-in
```

Pros:
- No synchronisation needed
- Simpler lifecycle (no explicit return)
- Thread-local `get()` is fast in modern JVMs

Cons:
- One object per thread — can't share objects between threads on demand
- Can't throttle resource access (unless thread count itself is the throttle)

Performance comparison (10,000 random numbers, 4 threads):
| Approach | Time |
|---|---|
| Create new `Random` each call | 134.9 µs |
| `ThreadLocalRandom` | 52.0 µs |
| Shared `Random` (contended) | 3,763 µs |

---

## Soft, Weak, and Other References

### Reference Types

| Type | When freed | Use case |
|---|---|---|
| Strong | Never while reachable | Normal use |
| Soft | When heap is low AND not recently accessed | LRU cache |
| Weak | At next GC after last strong reference drops | Secondary index; multi-consumer cache |
| Phantom | After finalizer; when all strong refs drop | Cleanup hook (JDK 11: use `Cleaner`) |
| Final (Finalizer) | After `finalize()` runs; two GC cycles | Avoid — use `Cleaner` instead |

Memory overhead of any indefinite reference: ~40 bytes for the reference object itself, plus at minimum two extra GC cycles before the reference object is freed.

### Soft References

```java
SoftReference<StockHistory> ref = new SoftReference<>(history);
StockHistory h = ref.get();  // null if cleared
```

Clearing policy:
```
ms = SoftRefLRUPolicyMSPerMB × freeHeapMB
if (now - lastAccessTime > ms): clear referent
```

- `-XX:SoftRefLRUPolicyMSPerMB=N` (default 1000)
- 4 GB heap, 50% free (2 GB free) → referent kept for 2,048 seconds (~34 min)
- Decrease this value if soft references are filling the heap before they can be cleared
- Effective LRU cache — the JVM manages eviction automatically
- If heap fills completely, all soft references are cleared (last resort before OOM)

### Weak References

```java
WeakReference<StockHistory> ref = new WeakReference<>(history);
StockHistory h = ref.get();  // null after next GC if no strong references remain
```

- Cleared at every GC cycle once the referent has no strong references
- Use when another part of the application holds a strong reference and you want secondary access to the object
- Do NOT use as a general-purpose cache — object may disappear at any GC

### WeakHashMap / WeakIdentityMap

- Keys are weakly referenced; entry removed when key is GC'd
- Entry cleanup happens on next map operation — not immediately
- Map operations have unpredictable latency when many keys are cleared at once
- Use with caution; application-managed collections are often more predictable

### Finalizers (Avoid)

- Deprecated in JDK 11
- Performance problems:
    - Both the finalizer reference object AND the referent are held in memory until after `finalize()` runs
    - Takes at least 2 GC cycles to free everything
    - `finalize()` can inadvertently resurrect objects
- Functional problems:
    - `finalize()` can create a new strong reference to the object, preventing collection
    - No guarantee when (or if) `finalize()` will be called

### Cleaner Objects (JDK 11 Replacement)

```java
// Register cleanup action; phantom-reference-based
Cleaner cleaner = Cleaner.create();
Runnable cleanup = () -> freeNativeMemory(addr);
Cleaner.Cleanable cleanable = cleaner.register(owner, cleanup);

// Explicitly clean (when close() is called)
cleanable.clean();
```

Rules:
- The cleanup `Runnable` must NOT hold a reference to `owner` (otherwise owner can never become phantom reachable)
- Use a static inner class or a separate class for the cleanup; never a lambda that captures `this`

---

## Compressed Oops

- 64-bit JVM uses 8-byte references by default — takes twice the space of 32-bit references
- __Compressed oops__ store references as 32-bit values in the heap; shift left by 3 bits when loading into a register
- Objects must be on 8-byte boundaries (already required by JVM for alignment)
- Max addressable memory: 2^35 = 32 GB

Flags:
- `-XX:+UseCompressedOops` — enabled by default when `-Xmx < 32 GB`
- `-XX:+UseCompressedClassPointers` — also enabled by default; compresses class pointer in object header

Implication:
- A 31 GB heap with compressed oops typically outperforms a 33 GB heap without them
- Without compressed oops, references take twice the space → more GC cycles to compensate for the extra memory consumption
- If you must exceed 32 GB, go to at least ~38 GB to offset the expanded reference overhead (references ≈ 20% of average heap)

---

## GC Log — Reference Processing

Enable with: `-XX:+PrintReferenceGC` (JDK 8) or `gc+ref*=debug` (JDK 11)

```
[WeakReference, 238425 refs, 0.0236510 secs]
```

- Shows time spent processing each reference type per GC cycle
- Large numbers of weak references (238K+) adding 23ms per young GC → investigate
