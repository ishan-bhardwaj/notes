# Locking

## `synchronized`

- Built-in Java locking mechanism.
- Used to restrict access to a critical section or an entire method to one thread at a time.
- Implemented using an object's monitor (intrinsic lock).
- Two ways to use it - `synchronized` method or block.

### Synchronized Method (Monitor Lock)

```
public synchronized void method1() {
  // critical section
}
```

  - Every Java object has an associated monitor (intrinsic lock).
  - A synchronized instance method implicitly acquires the monitor of `this` object before execution.
  - If thread `A` acquires the monitor and enters` method1()`, no other thread can enter any synchronized method (`method1()` or `method2()`) of the same object until the monitor is released.
  - Synchronization is therefore enforced per monitor (per object), not per method.

  > [!TIP]
  > Static synchronized methods use the monitor of the `Class` object instead of `this`.

### Synchronized Block (Custom Lock)

```
Object lock = new Object();

public void method1() {
  synchronized(lock) {
    // critical section
  }
}
```

  - Any Java object can be used as a monitor lock.
  - A thread must acquire the monitor of lock before entering the synchronized block.
  - Different lock objects can protect different critical sections, allowing greater concurrency.
  - Two threads can execute synchronized blocks simultaneously if they synchronize on different lock objects.

  > [!TIP]
  > A synchronized instance method is logically equivalent to synchronizing on `this`.
  >
  > ```
  > public synchronized void method1() {
  >   // critical section
  > }
  > ```
  >
  > is equivalent to -
  >
  > ```
  > public void method1() {
  >   synchronized(this) {
  >     // critical section
  >   }
  > }
  > ```

## Reentrancy

- Java monitor locks are reentrant.
- A thread that already owns a lock is allowed to enter another synchronized block/method that requires the same lock.
- Without reentrancy, the thread would block waiting for a lock that it already owns, causing self-deadlock.

## Designing Thread-Safe Classes

- Thread safety is about protecting shared mutable state and preserving invariants under concurrency.
- Encapsulation is critical because it limits access to mutable state and makes thread safety easier to reason about.

- __Steps to design a thread-safe class__ -
  - Identify the object state
  - Identify invariants/postconditions
  - Define synchronization policy

- __Object State__ -
  - Includes primitive fields, referenced objects, internal object graph, derived/cache state
  - Eg - `LinkedList` state includes -
    - head/tail
    - node links
    - size etc

- __Invariants__ -
  - Invariant = condition always true for valid object state
  - Defines legal state space
  - Eg - `value >= 0` or `lower <= upper`
  - Invariants may be temporarily broken during mutation internally
  - Synchronization prevents other threads from observing inconsistent intermediate states

- __Compound Actions & Atomicity__ -
  - Operations where next state depends on current state must be atomic
  - Eg - _check-then-act_, _read-modify-write_ etc.
  - Without synchronization, another thread may modify state between operations

- __Multivariable Invariants__ -
  - If multiple variables participate in the same invariant (`lower <= upper`) then -
    - they must usually be guarded by the same lock
    - updates must happen atomically.
  - Eg - Say `lower = 10` and `upper = 20`
    - Thread `A` is updating both the variables one-by-one, so it updates `lower = 50` and releases the lock
    - But before updating `upper = 100`, Thread `B` reads - `lower = 50` and `upper = 20`
    - Leading to inconsistent state

- __Synchronization Policy__ -
  - Defines -
    - which state variables are shared
    - which lock protects them
    - how thread safety is achieved
  - Mechanisms -
    - locking
    - immutability
    - confinement

- __State Space__ -
  - State space = all possible states an object can be in.
  - Smaller state space → easier concurrency reasoning.
  - `final` fields and immutability reduce state space.

- __State-Dependent Operations__ -
  - Some operations are only valid in certain states.
  - Eg - `remove()` requires queue to be non-empty
  - Concurrent programs may wait until condition becomes true.
  - Common tools - 
    - `BlockingQueue`
    - `Semaphore`
    - `wait()` / `notify()`

- __Ownership__ -
  - Thread safety depends on ownership of mutable state.
  - Ownership determines who controls synchronization.
  - Eg - 
    - synchronized collection protects collection structure
    - NOT mutable objects stored inside it

## Confinement

- Non-thread-safe components can safely exist inside thread-safe containers.
- Types of confinement -
  - __Instance confinement__ - private field inside object
  - __Lexical confinement__ - local variables inside method/block
  - __Thread confinement__- object accessed only by one thread
- Confinement alone is insufficient for shared access
  - Usually combined with locking

## Instance confinement

- Core idea - confined objects must NOT escape intended scope -
  - object is encapsulated within another object
  - all access paths are controlled
  - synchronization becomes easier to reason about

- __Escape Analysis__ -
  - Confinement breaks if object escapes.
  - Object may escape via -
    - returning references
    - publishing through getters
    - exposing iterators
    - inner classes/lambdas
    - callbacks/listeners
    - shared mutable references
  - Publishing confined object = synchronization bug.

- Example -
  ```
  public class PersonSet {

    private final Set<Person> mySet = new HashSet<Person>();

    public synchronized void addPerson(Person p) {
      mySet.add(p);
    }

    public synchronized boolean containsPerson(Person p) {
      return mySet.contains(p);
    }
  }
  ```

  - The state of `PersonSet` is managed by a `HashSet`, which is not thread-safe
  - But because `mySet` is private and not allowed to escape - the `HashSet` is confined to the `PersonSet`
  - The only code paths that can access `mySet` are `addPerson` and `containsPerson` -
    - each of these acquires the lock on the `PersonSet`
  - All its state is guarded by its intrinsic lock, making `PersonSet` thread-safe.

### Synchronized Wrappers

- Thread-safe wrappers around non-thread-safe collections like `ArrayList` and `HashMap`
- Eg - `Collections.synchronizedList` and `Collections.synchronizedMap`
- Uses Decorator pattern - synchronized wrapper forwards calls to underlying collection
- Wrapper is thread-safe only if -
  - underlying collection remains confined to wrapper
  - all access happens through wrapper only
- If raw collection reference escapes → thread safety breaks

### Java Monitor Pattern

- Logical conclusion of instance confinement.
- An object -
  - encapsulates all its mutable state
  - guards it with the object’s own intrinsic lock
- Used in `Vector`, `HashTable` etc

### Private Lock Pattern

- Instead of intrinsic lock (`this`), use private lock object
- Eg - `private final Object lock = new Object()`
- Advantages -
  - clients cannot acquire lock
  - synchronization policy fully encapsulated

## Delegating Thread Safety

- Composite object may be thread-safe if underlying components are thread-safe.
- Delegation = relying on underlying thread-safe components for synchronization correctness.
- Delegation works if -
  - state variables are independent
  - no cross-variable invariants exist
  - no prohibited state transitions exist
  - no compound atomic operations required
- Immutability makes delegation dramatically simpler.
- Example - `AtomicLong`, `ConcurrentHashMap`, `CopyOnWriteArrayList`
- Publishing underlying state is safe only if variable -
  - is thread-safe
  - does not participate in invariants
  - has no restricted transitions

## Adding Functionality to Existing Thread-Safe Classes

- Option 1 - __Modify Original Class__ -
  - add method directly inside original class
  - use same synchronization policy
  - Problems -
    - often source code is unavailable
    - or class cannot be modified

- Option 2 - __Extend the Class__ -
  ```
  class BetterVector<E> extends Vector<E> {
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !contains(x);
        if (absent) add(x);
        return absent;
    }
  }
  ```

  - Works if subclass uses same lock as parent
  - Fragile because synchronization policy is split across parent and subclass
  - If parent changes locking strategy, subclass may break

- Option 3 - __Client-Side Locking__ -
  - Add atomic behavior outside the class by locking on the same lock used by the object
  - Incorrect - locks `ListHelper`, not the actual `list`, only giving illusion of thread safety -
    ```
    public synchronized boolean putIfAbsent(E x) {
      boolean absent = !list.contains(x);
      if (absent) list.add(x);
      return absent;  
    }
    ```

  - Correct - works only if `list` uses its own intrinsic lock -
    ```
    public boolean putIfAbsent(E x) {
      synchronized (list) {
        boolean absent = !list.contains(x);
        if (absent) list.add(x);
        return absent;
      }
    }
    ```

  - Problems -
    - Depends on knowing another class’s lock
    - Violates encapsulation of synchronization policy
    - Dangerous if class does not document its locking strategy

- Option 4 (Best Practical) - __Composition__ -
  - Wrap the existing collection inside a new class.
  - New class controls all access using its own lock.
    ```
    class ImprovedList<T> {
      private final List<T> list;

      public synchronized boolean putIfAbsent(T x) {
        boolean absent = !list.contains(x);
        if (absent) list.add(x);
        return absent;
      }
    }
    ```
