# Locking

## `synchronized`

- Built-in Java locking mechanism - restricts access to a critical section or an entire method to one thread at a time
- Implemented using an object's monitor (intrinsic lock) - every Java object has an associated monitor
- Two ways to use it - `synchronized` method or block

### Synchronized Method (Monitor Lock)

```
public synchronized void method1() {
  // critical section
}
```

  - Implicitly acquires the monitor of `this` object before execution
  - If thread `A` acquires the monitor and enters` method1()`, no other thread can enter any synchronized method (`method1()` or `method2()`) of the same object until the monitor is released

  > [!TIP]
  > Static synchronized methods use the monitor of the `Class` object instead of `this`

### Synchronized Block (Custom Lock)

```
Object lock = new Object();

public void method1() {
  synchronized(lock) {
    // critical section
  }
}
```

  - Any Java object can be used as a monitor lock
  - A thread must acquire the monitor of lock before entering the synchronized block
  - Two threads can execute the same synchronized block simultaneously if they synchronize on different lock objects

  > [!TIP]
  > A synchronized instance method is logically equivalent to synchronizing on `this`
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

- Java monitor locks are reentrant
- A thread that already owns a lock is allowed to enter another synchronized block/method that requires the same lock
- Without reentrancy, the thread would block waiting for a lock that it already owns, causing self-deadlock

## Designing Thread-Safe Classes

- Thread safety = protecting shared mutable state and preserving invariants
- __Design Steps__ -
  - Identify object state
  - Identify invariants
  - Define synchronization policy
- __Object State__ -
  - Includes fields + referenced mutable objects
  - Eg - `LinkedList` → head, tail, size, node links
- __Invariants__ -
  - Conditions that must always be true for valid state
  - Eg - `value >= 0`, `lower <= upper`
  - Synchronization prevents threads from seeing broken intermediate state
- __Compound Actions__ - check-then-act, read-modify-write must be atomic
- __Multivariable Invariants__ -
  - Related variables should usually use the same lock
  - Updates must happen atomically
- __Synchronization Policy__ -
  - Defines -
    - what is shared
    - which lock protects it
    - how thread safety is achieved
- __State-Dependent Operations__ -
  - Valid only in certain states
  - Eg - `remove()` requires non-empty queue
- __Ownership__ -
  - Owner controls synchronization
  - Synchronized collection protects collection structure, not mutable elements inside it

## Confinement

- Non-thread-safe components can safely exist inside thread-safe containers
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
  - Confinement breaks if object escapes
  - Object may escape via -
    - returning references
    - publishing through getters
    - exposing iterators
    - inner classes/lambdas
    - callbacks/listeners
    - shared mutable references
  - Publishing confined object = synchronization bug

- Eg -
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
  - All its state is guarded by its intrinsic lock, making `PersonSet` thread-safe

### Synchronized Wrappers

- Thread-safe wrappers around non-thread-safe collections like `ArrayList` and `HashMap`
- Eg - `Collections.synchronizedList` and `Collections.synchronizedMap`
- Uses Decorator pattern - synchronized wrapper forwards calls to underlying collection
- Wrapper is thread-safe only if -
  - underlying collection remains confined to wrapper
  - all access happens through wrapper only
- If raw collection reference escapes → thread safety breaks

### Java Monitor Pattern

- Logical conclusion of instance confinement
- An object -
  - encapsulates all its mutable state
  - guards it with the object’s own intrinsic lock (`this`)
- Used in `Vector`, `HashTable` etc

### Private Lock Pattern

- Instead of intrinsic lock (`this`), use private lock object
- Eg - `private final Object lock = new Object()`
- Advantages -
  - clients cannot acquire lock
  - synchronization policy fully encapsulated

## Delegating Thread Safety

- Delegation = relying on underlying thread-safe components for synchronization correctness
- Delegation works if -
  - state variables are independent
  - no cross-variable invariants exist
  - no prohibited state transitions exist
  - no compound atomic operations required
- Immutability makes delegation dramatically simpler
- Eg - `AtomicLong`, `ConcurrentHashMap`, `CopyOnWriteArrayList`
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
  - Wrap the existing collection inside a new class
  - New class controls all access using its own lock
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
