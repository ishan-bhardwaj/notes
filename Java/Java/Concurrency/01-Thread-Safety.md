# Threads

- A process contains -
  - Metadata - like process id.
  - Files - that the application opens for reading and writing.
  - Code - the program instructions to be executed.
  - Heap - contains all the data that the application needs.
  - At least one thread called the main thread - 
    - Contains 2 main things -
      - Stack - region in memory where local variables are stored and passed into functions.
      - Instruction Pointer - address of the next instruction to execute.

- In multithread environment, each thread has its own stack and instruction pointer, while the other components are shared by all threads.

> [!NOTE]
> Context switching with too many threads causes _thrashing_ which means spending more time in management than real productive work.

- Threads consume less resources than process. Therefore, context switching between threads from the same process is cheaper than context switching between processes.

- Threads are much faster to create and destroy than processes.

## Thread Safety

- A class is _thread-safe_ if it behaves correctly when accessed from multiple threads, regardless of the scheduling or interleaving of the execution of those threads by the runtime environment, and with no additional synchronization or other coordination on the part of the calling code.

> [!TIP]
> Stateless objects are always thread-safe.

## Race Conditions

- A race condition occurs when -
  - Multiple threads access shared mutable state.
  - Final result depends on timing/interleaving of execution.

- Types -
  - __Read-Modify-Write__ -
    - Multiple threads read the same value, modify it, and write it back → updates get lost.
      ```
      count++;
      ```
    
    - `count++` is a sequence of 3 operations -
      - fetch the current value
      - add one to it
      - write the new value back

    - Problem  -
      - Thread `A` and `B` read the same value
      - Both write same result
      - One increment is lost - _lost updates_ problem

  - __Check-Then-Act__ -
    - A thread checks a condition, but before acting, another thread changes it.
      ```
      if (map.containsKey(key)) {
        return map.get(key);
      }
      ```

    - Problem -
      - Thread `A` checks - `key` exists
      - Thread `B` removes `key`
      - Thread `A` tries `get()` - unexpected result (`null` / error)

  - __Initialization Race__ -
    - Multiple threads initialize something that should be created once.
      ```
      if (instance == null) {
        instance = new Object();
      }
      ```

    - Problem -
      - Two threads see `null`
      - Both create objects - duplicate instances.

  - __Visibility Race__ -
    - One thread updates a variable, but another thread doesn’t see the updated value.
      ```
      boolean flag = false;

      // Thread A
      while (!flag) { }

      // Thread B
      flag = true;
      ```

    - Problem -
      - Thread `A` has its own copy of `flag = false`
      - Thread `B` updates the main memory
      - Thread `A` keeps reading its old cached value - infinite loop

  - __Order-Dependent Race__ -
    - Execution order matters, but threads interleave unpredictably.
      ```
      int x = 0, y = 0;

      // Thread A
      x = 1;
      y = 1;

      // Thread B
      if (y == 1) {
        System.out.println(x);                      // may print 0
      }
      ```

    - Problem -
      - CPU / JVM may reorder -
        ```
        y = 1;
        x = 1;
        ```

      - Thread `A` runs and sets `y = 1` first
      - Thread `B` runs, sees `y = 1`
      - Thread `B` prints `x` which is still `0`

  - __Compound Action Race__ -
    - Multiple operations that should be atomic are split.
      ```
      if (!list.isEmpty()) {
        list.remove(0);
      }
      ```

    - Problem -
      - Thread `A` checks - not empty
      - Thread `B` removes element
      - Thread `A` removes - exception  

## Atomic Classes

- `java.util.concurrent.atomic` classes provide lock-free, thread-safe operations on a single variable (single memory location).
- Core Mechanism - __CAS (Compare-and-Set)__ -
  - CAS = Update value only if it hasn’t changed since last read.
  - Enables optimistic concurrency -
    - No locking
    - Assume no contention
    - Retry on conflict
    
  ```
  do {
    prev = value;
    next = prev + 1;
  } while (!compareAndSet(prev, next));
  ```

- __Common Atomic Classes__ -
  - `AtomicInteger`, AtomicLong`
  - `AtomicBoolean`
  - `AtomicReference<T>`
  - `AtomicIntegerArray`, `AtomicReferenceArray`
  - `AtomicStampedReference`, `AtomicMarkableReference`

- __API Usage__ -
  - Counter -
    ```
    AtomicInteger count = new AtomicInteger(0);
    count.incrementAndGet();                            // count++
    ```

  - Run Once (flag) -
    ```
    AtomicBoolean flag = new AtomicBoolean(false);

    if (flag.compareAndSet(false, true)) {
      // runs only once
    }
    ```

  - Safe Update (CAS loop) - update based on current value -
    ```
    AtomicInteger balance = new AtomicInteger(100);

    while (true) {
      int curr = balance.get();
      int next = curr - 10;

      if (balance.compareAndSet(curr, next)) break;
    }
    ```

  - Prevent Check-Then-Act Bug -
    ```
    AtomicInteger stock = new AtomicInteger(1);

    while (true) {
      int curr = stock.get();
      if (curr == 0) break;

      if (stock.compareAndSet(curr, curr - 1)) break;
    }
    ```

  - Multi-field State (AtomicReference) -
    ```
    class State {
      final int count;
      State(int c) { count = c; }
    }

    AtomicReference<State> ref = new AtomicReference<>(new State(0));

    while (true) {
      State old = ref.get();
      State next = new State(old.count + 1);

      if (ref.compareAndSet(old, next)) break;
  }
    ```

- Gurantees -
  - __Atomicity__ -
    - Each operation is indivisible (no partial updates).
  - __Visibility__ -
    - Same as `volatile`.
    - Changes are immediately visible to other threads.
  - __Ordering__ -
    - Write → release
    - Read → acquire
    - Ensures correct ordering of operations across threads.

- __Implementation__ -
  - Pre Java 9 → `Unsafe.compareAndSwapXXX`
  - Java 9+ → `VarHandle`
  - Both are -
    - JVM intrinsics
    - Mapped to CPU atomic instructions
    - Include memory barriers (fences)

