# Atomic Classes

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
