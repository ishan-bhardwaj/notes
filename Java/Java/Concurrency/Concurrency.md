# Core Java Vol. I — Chapter 10: Concurrency

## Running Threads

```java
Runnable r = () -> { task code };
var t = new Thread(r);
t.start();
```

- Call `t.start()` — NOT `t.run()`; calling `run()` directly executes in the calling thread
- `Thread.start()` creates a new thread and calls `run()` on it
- Prefer `Runnable`/`Callable` over subclassing `Thread` — decouple task from execution mechanism

---

## Thread States

Six states: `NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TIMED_WAITING`, `TERMINATED`

- `NEW` — created with `new Thread(r)` but not yet started
- `RUNNABLE` — after `start()`; may or may not be actively running (scheduler decides)
    - Scheduler can preempt running threads; threads can voluntarily yield with `Thread.yield()`
- `BLOCKED` — waiting to acquire an intrinsic object lock held by another thread
- `WAITING` — waiting indefinitely for notification (`Object.wait()`, `Thread.join()`, `Lock`/`Condition` wait)
- `TIMED_WAITING` — waiting with timeout (`Thread.sleep()`, timed `wait()`, `join()`, `tryLock()`, `await()`)
- `TERMINATED` — `run()` returned normally or uncaught exception terminated it
- Query state: `t.getState()`

> [!NOTE]
> `stop()` throws `UnsupportedOperationException` since Java 21. `suspend()`/`resume()` removed in Java 25.

---

## Thread Properties

### Virtual Threads

- __Platform thread__ — mapped 1:1 to OS thread; heavyweight (KB-MB of memory, thousands of CPU instructions to start)
- __Virtual thread__ — many-to-few mapping onto __carrier threads__ (one per processor by default); lightweight
- Virtual threads: start with `Thread.startVirtualThread(runnable)` or `Thread.ofVirtual().start(r)`
- `t.isVirtual()` — true if virtual
- Use virtual threads for blocking I/O workloads; not for CPU-intensive tasks
- All virtual threads are daemon threads; `setDaemon(false)` has no effect on them
- Tune carrier thread count: VM option `jdk.virtualThreadScheduler.parallelism`

### Thread Interruption

- `t.interrupt()` — sets the interrupted status flag to `true`
- If thread is blocked on `sleep`/`wait`: throws `InterruptedException` and clears the flag
- `Thread.currentThread().isInterrupted()` — check flag without clearing
- `Thread.interrupted()` — static; checks and CLEARS the flag
- `isInterrupted()` — instance; checks without clearing
- Do NOT swallow `InterruptedException` silently — either:
    - Propagate: declare `throws InterruptedException`
    - Re-set the flag: `Thread.currentThread().interrupt()` in catch block
- Calling `sleep()` when interrupted status is set: doesn't sleep, clears status, throws `InterruptedException`

### Daemon Threads

- `t.setDaemon(true)` — must be called before `start()`
- JVM exits when only daemon threads remain
- Use for background service threads (timers, cache cleaners)

### Thread Names and IDs

- `t.setName("name")` — useful in thread dumps
- `t.threadId()` — unique positive long ID (use over deprecated `getId()`)

### Uncaught Exception Handlers

- `run()` cannot throw checked exceptions; unchecked exceptions go to handler
- `t.setUncaughtExceptionHandler(handler)` — per-thread handler
- `Thread.setDefaultUncaughtExceptionHandler(handler)` — static default
- Handler interface: `Thread.UncaughtExceptionHandler.uncaughtException(Thread t, Throwable e)`
- Without a handler: thread group's `uncaughtException` is called, then default handler, then stack trace to `System.err`

### Thread Priorities

- `setPriority(int)` — range 1 (`MIN_PRIORITY`) to 10 (`MAX_PRIORITY`); 5 is `NORM_PRIORITY`
- Highly system-dependent; Linux OpenJDK ignores priorities entirely
- Do NOT use priorities in modern code
- Virtual threads always have `NORM_PRIORITY`; changing has no effect

### Thread Factories and Builders

```java
Thread.Builder builder = Thread.ofVirtual().name("request-", 1);
Thread t = builder.start(myRunnable);
ThreadFactory factory = builder.factory();
```

- `Thread.ofPlatform()` / `Thread.ofVirtual()` — builder APIs
- Builder methods: `name(prefix, start)`, `daemon()`, `priority()`, `uncaughtExceptionHandler()`

---

## Coordinating Tasks

### `Callable<V>` and `Future<V>`

- `Callable<V>` — like `Runnable` but returns `V` and can throw checked exceptions
- `Future<V>` methods:
    - `get()` — blocks until done; throws `ExecutionException` if task failed
    - `get(timeout, unit)` — blocks with timeout; throws `TimeoutException`
    - `resultNow()` — non-blocking; throws `IllegalStateException` if not done successfully
    - `exceptionNow()` — non-blocking; throws `IllegalStateException` if not failed
    - `cancel(boolean mayInterrupt)` — cancels task (requires task cooperation via interrupt)
    - `isDone()`, `isCancelled()`, `state()` — query state (`RUNNING`/`SUCCESS`/`FAILED`/`CANCELLED`)
- `FutureTask<V>` — implements both `Future` and `Runnable`; wraps a `Callable`

### Executor Services

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<V> f = executor.submit(myCallable);
    ...
} // executor.close() blocks until all tasks finish
```

Common factory methods:

| Method | Description |
|---|---|
| `newCachedThreadPool()` | Creates threads as needed; idles kept 60s |
| `newFixedThreadPool(n)` | Fixed pool size; excess tasks queued |
| `newSingleThreadExecutor()` | Single thread; sequential execution |
| `newVirtualThreadPerTaskExecutor()` | New virtual thread per task |

- `submit(Callable<T>)` → `Future<T>`
- `submit(Runnable)` → `Future<?>`
- `close()` — shuts down, blocks until complete (Java 19+)
- `shutdown()` — no new tasks accepted; does not block
- `awaitTermination(timeout, unit)` — blocks until terminated or timeout

### Invoking Groups of Tasks

- `invokeAll(tasks)` — blocks until all complete; returns `List<Future<T>>` in submission order
- `invokeAny(tasks)` — blocks until one completes successfully; cancels others; returns result
    - Failing tasks should throw exceptions (not return null) so `invokeAny` doesn't stop on them
- `ExecutorCompletionService<T>` — wraps an executor; `take()` returns futures in completion order

### Thread-Local Variables

```java
public static final ThreadLocal<SimpleDateFormat> DATEFORMAT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

- `get()` — returns this thread's instance; initialises on first call
- `set(v)` / `remove()` — set or clear the value
- MUST call `remove()` on task completion when using thread pools — prevents memory leaks and cross-task pollution
- `InheritableThreadLocal` — child threads inherit a copy of parent's value
- `ThreadLocalRandom.current()` — per-thread random generator, more efficient than shared `Random`
- Trace virtual thread usage of thread locals: `-jdk.traceVirtualThreadLocals` VM flag

### Scoped Values (Java 25)

```java
public static final ScopedValue<Connection> CONNECTION = ScopedValue.newInstance();
ScopedValue.where(CONNECTION, connect(...)).run(() -> doWork());
```

- Immutable and bounded lifetime — no `remove()` needed; automatic cleanup after `run()`/`call()`
- `ScopedValue.where(key, value).run(runnable)` — binds value for the duration of `run()`
- `CONNECTION.get()` — retrieves the current thread's binding
- Rebinding in nested scope reverts when nested scope exits
- More performant than inheritable thread locals for virtual thread hierarchies
- Inherited by virtual threads created in a `StructuredTaskScope`

### Fork-Join Framework

```java
class Counter extends RecursiveTask<Integer> {
    protected Integer compute() {
        if (to - from < THRESHOLD) { /* direct */ }
        else {
            var first = new Counter(values, from, mid, filter);
            var second = new Counter(values, mid, to, filter);
            invokeAll(first, second);
            return first.join() + second.join();
        }
    }
}
new ForkJoinPool().invoke(counter);
```

- `RecursiveTask<T>` — produces a result; `RecursiveAction` — no result
- `invokeAll(tasks...)` — submits subtasks and blocks until complete
- `join()` — gets result; `get()` also works but throws checked exceptions
- Uses __work stealing__ — idle threads steal tasks from tail of other threads' deques
- Not suitable for blocking workloads; can starve the pool

---

## Synchronization

### Race Conditions

- Non-atomic compound operations (read-modify-write) cause data corruption when threads interleave
- Eg - `accounts[to] += amount` compiles to load/add/store bytecodes; preemption between them corrupts state
- Must synchronize any shared mutable state

### `ReentrantLock` and `Condition`

```java
private final Lock bankLock = new ReentrantLock();
private final Condition sufficientFunds = bankLock.newCondition();

public void transfer(int from, int to, double amount) throws InterruptedException {
    bankLock.lock();
    try {
        while (accounts[from] < amount)
            sufficientFunds.await();
        accounts[from] -= amount;
        accounts[to] += amount;
        sufficientFunds.signalAll();
    } finally {
        bankLock.unlock();
    }
}
```

- `lock()` / `unlock()` — always put `unlock()` in `finally`; cannot use try-with-resources
- __Reentrant__ — same thread can acquire the lock it already holds; hold count tracks nesting
- `ReentrantLock(fair)` — fair lock favours longest-waiting thread; significantly slower
- `condition.await()` — releases lock and waits; thread enters condition's wait set
- `condition.signalAll()` — moves all waiting threads from wait set to runnable; they re-compete for lock
- `condition.signal()` — wakes one random thread; risk of deadlock if wrong thread is chosen
- `await()` always in a loop: `while (!condition) condition.await()`
- `signalAll()` whenever state changes in a way that might help waiting threads

### `synchronized` Keyword

```java
public synchronized void transfer(...) throws InterruptedException {
    while (accounts[from] < amount) wait();
    accounts[from] -= amount;
    accounts[to] += amount;
    notifyAll();
}
```

- Each object has an intrinsic lock and single intrinsic condition
- `synchronized` method = acquire intrinsic lock on entry, release on exit
- `wait()` = `intrinsicCondition.await()`; `notifyAll()` = `intrinsicCondition.signalAll()`
- `static synchronized` = acquires lock on the `Class` object
- Limitations vs `ReentrantLock`:
    - Before Java 25: virtual threads pinned while in `synchronized` block (cannot unmount from carrier)
    - Cannot interrupt a thread waiting to acquire intrinsic lock
    - Cannot specify timeout on lock acquisition
    - Single condition per lock

> [!NOTE]
> Since Java 25, `synchronized` no longer pins virtual threads. Native methods and foreign functions still do. Monitor pinning with JFR events `VirtualThreadPinned`/`VirtualThreadSubmitFailed`.

### Synchronized Blocks

```java
synchronized (obj) { critical section }
```

- Acquires intrinsic lock of `obj`
- Avoid locking on string literals (shared), primitive wrappers (`Integer.valueOf` may return cached instances), or `getClass()` (breaks with subclasses)
- Lock on `MyClass.class` for static fields
- __Client-side locking__ — hijacking another object's lock — fragile and not recommended
- __Monitor concept__ — Java loosely implements monitors; differs by: fields need not be private, methods need not be synchronized, intrinsic lock is accessible to clients

### Volatile Fields

```java
private volatile boolean done;
```

- Ensures visibility across threads — compiler/JVM insert memory barrier instructions
- Does NOT provide atomicity — `done = !done` is still not thread-safe
- Use when: one thread writes, others only read, and no compound operations are needed

### Final Fields

- `final` fields safely visible to all threads after constructor completes — no synchronisation needed
- Requires: object was __properly constructed__ — `this` must not escape during construction
- Non-final fields of a properly constructed object with all-final fields are also safe
- Mutable operations on objects stored in `final` fields still require synchronisation

### Atomic Classes (`java.util.concurrent.atomic`)

- `AtomicLong`, `AtomicInteger`, `AtomicReference`, etc.
- `incrementAndGet()`, `decrementAndGet()` — atomic; equivalent to `++`/`--`
- `compareAndSet(expected, update)` — atomic CAS; basis of all other atomic updates
- `updateAndGet(x -> f(x))` — atomically applies a function
- `accumulateAndGet(value, binaryOp)` — atomically combines with existing value
- `LongAdder` / `DoubleAdder` — high-contention counter; splits into multiple summands; use `increment()`, `add()`, `sum()`
- `LongAccumulator(op, identity)` — generalises `LongAdder` to arbitrary associative/commutative operations

### On-Demand Initialisation

```java
public static OnDemandData getInstance() { return Holder.INSTANCE; }
private static class Holder {
    static final OnDemandData INSTANCE = new OnDemandData();
}
```

- JVM initialises static initializers exactly once, under a lock — safe lazy init without explicit synchronization

### Safe Publication

An object is safely published when its reference is stored in:
- A static initializer
- A `volatile` field or `AtomicReference`
- A `final` field of a properly constructed object
- A field protected by a lock (at assignment time)
- A thread-safe collection (Eg - `BlockingQueue`, `ConcurrentHashMap`)

---

## Thread-Safe Collections

### Blocking Queues

- Producer threads `put`; consumer threads `take`; queue handles synchronisation
- `put(e)` — blocks if full; `take()` — blocks if empty
- `offer(e)` / `poll()` — return `false`/`null` immediately on failure
- `offer(e, time, unit)` / `poll(time, unit)` — timed versions
- Implementations:
    - `ArrayBlockingQueue(capacity)` — bounded circular array; optional fairness
    - `LinkedBlockingQueue()` / `LinkedBlockingDeque()` — unbounded (or optionally bounded)
    - `PriorityBlockingQueue` — unbounded priority queue
    - `LinkedTransferQueue` — `transfer(e)` blocks until consumer removes item
- `null` cannot be inserted — used as failure indicator by `poll`/`peek`
- __Poison pill__ — special sentinel object inserted by producer to signal termination to consumers

### Concurrent Maps, Sets, Queues

- `ConcurrentHashMap<K,V>` — fine-grained locking; different buckets lockable concurrently
    - `size()` is approximate; use `mappingCount()` for long counts
    - Buckets use trees (not lists) when key type is `Comparable` — O(log n) worst case
    - `compute(key, (k,v) -> ...)` — atomic update; returns new value
    - `merge(key, value, BiFunction)` — put if absent, otherwise combine
    - `computeIfAbsent(key, f)` — compute and put only if absent; returns new value (chainable)
    - Bulk operations: `forEach`, `search`, `reduce` (with `Keys`, `Values`, `Entries` variants); require parallelism threshold
    - `newKeySet()` — creates a `Set<K>` backed by the map
- `ConcurrentSkipListMap<K,V>` / `ConcurrentSkipListSet<E>` — sorted, concurrent
- `ConcurrentLinkedQueue<E>` — unbounded non-blocking queue
- All return __weakly consistent__ iterators — no `ConcurrentModificationException`; may not reflect all recent updates

### Copy-on-Write Collections

- `CopyOnWriteArrayList<E>` / `CopyOnWriteArraySet<E>` — mutators copy the underlying array
- Safe for concurrent iteration without locks
- Iterator sees a snapshot — modifications after iterator construction not reflected
- Best when reads vastly outnumber writes

### Parallel Array Algorithms

- `Arrays.parallelSort(array)` — parallel merge sort; stable
- `Arrays.parallelSetAll(array, i -> ...)` — parallel fill by index function
- `Arrays.parallelPrefix(array, (x,y) -> ...)` — parallel prefix accumulation (Eg - running totals)

### Synchronization Wrappers

```java
List<E> synchList = Collections.synchronizedList(new ArrayList<>());
```

- All methods synchronised on the collection object
- Client-side locking still needed for iteration:

```java
synchronized (synchList) {
    for (var e : synchList) { ... }
}
```

- Prefer `java.util.concurrent` classes over wrappers for new code

---

## Asynchronous Computation

### `CompletableFuture<T>`

- Implements both `Future<T>` and `CompletionStage<T>`
- Two completion modes: result (`complete(v)`) or exception (`completeExceptionally(e)`)
- `CompletableFuture.supplyAsync(supplier, executor)` — creates a future from a `Supplier<T>`
- `cancel()` completes exceptionally with `CancellationException` — does not interrupt tasks

Key composition methods:

| Method | Parameter | Description |
|---|---|---|
| `thenApply(f)` | `T → U` | Apply function when complete |
| `thenAccept(f)` | `T → void` | Consume result |
| `thenCompose(f)` | `T → CompletableFuture<U>` | Chain another async computation |
| `thenRun(r)` | `Runnable` | Run after completion |
| `handle(f)` | `(T, Throwable) → U` | Process result or exception |
| `whenComplete(f)` | `(T, Throwable) → void` | Like handle but void |
| `exceptionally(f)` | `Throwable → U` | Provide fallback on exception |
| `completeOnTimeout(v, t, u)` | value, timeout | Complete with value on timeout |
| `orTimeout(t, u)` | timeout | Throw `TimeoutException` on timeout |
| `thenCombine(f2, fn)` | Future + BiFunction | Combine two results |
| `applyToEither(f2, fn)` | Future + Function | Use whichever finishes first |
| `allOf(futures...)` | varargs | Complete when all complete |
| `anyOf(futures...)` | varargs | Complete when any completes |

- Pipeline pattern:

```java
CompletableFuture.completedFuture(uri)
    .thenComposeAsync(this::readPage, executor)  // async step → supply executor here
    .thenApply(this::parse)                       // sync step
    .thenCompose(this::fetchImages)               // async step
    .thenAccept(this::save);                      // final consumer
```

### `SwingWorker<T, V>` (UI Background Tasks)

- Never do long work on the EDT (Event Dispatch Thread) — freezes UI
- `doInBackground()` — runs on worker thread; call `publish(V... data)` for progress updates
- `process(List<V> data)` — runs on EDT; receives batched progress data
- `done()` — runs on EDT after completion; call `get()` for result
- `execute()` — starts the worker
- `cancel(true)` — interrupts the worker thread

---

## Processes

### Building and Starting

```java
Process p = new ProcessBuilder("gcc", "myapp.c")
    .directory(path.toFile())
    .redirectErrorStream(true)
    .start();
```

- First string must be an executable, not a shell builtin (on Windows: `"cmd.exe", "/C", "dir"`)
- `inheritIO()` — inherit JVM's stdin/stdout/stderr
- `redirectInput(file)` / `redirectOutput(file)` / `redirectError(file)` — redirect to files
- `redirectOutput(ProcessBuilder.Redirect.appendTo(file))` — append mode
- `builder.environment()` — mutable map of env vars
- `ProcessBuilder.startPipeline(List<ProcessBuilder>)` — pipe output of one to input of next

### Running and Waiting

- `p.getOutputStream()` / `p.getInputStream()` / `p.getErrorStream()` — stream access
- `p.outputWriter()` / `p.inputReader()` / `p.errorReader()` — text access
- `p.waitFor()` — blocks; returns exit code (0 = success)
- `p.waitFor(timeout, unit)` — returns `true` if exited within timeout
- `p.exitValue()` — exit code without blocking (must be done)
- `p.isAlive()`, `p.destroy()`, `p.destroyForcibly()`
- `p.onExit()` — returns `CompletableFuture<Process>` for async notification

### Process Handles

```java
ProcessHandle.allProcesses()
    .filter(h -> h.info().command().filter(s -> s.contains("java")).isPresent())
    .forEach(h -> h.info().commandLine().ifPresent(System.out::println));
```

- `ProcessHandle.of(pid)`, `ProcessHandle.current()`, `p.toHandle()`
- `handle.pid()`, `handle.parent()`, `handle.children()`, `handle.descendants()`
- `handle.info()` — yields `ProcessHandle.Info` with `command()`, `arguments()`, `user()`, `startInstant()`, `totalCpuDuration()` (all `Optional`)
- Streams from `allProcesses`/`children`/`descendants` are snapshots — processes may terminate before you use them