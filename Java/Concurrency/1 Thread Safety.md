## Thread Safety

- **Concurrency** aims to encapsulate shared mutable state from uncontrolled concurrent access.
- **Synchronization** is used when a shared mutable variable is accessed by more than one thread and at least one of them modifies it.
- Ways to fix synchronization -
    - Don't share the variable between threads
    - Make the variable immutable
    - Use appropriate synchronization upon access
- A class is **thread-safe** when it continues to behave correctly when accessed from multiple threads. Thread-safe classes encapsulate any needed synchronization so that clients need not provide their own.
- **Stateless objects** are always thread-safe because -
    - No shared state - Since there’s no internal state to be shared, there’s nothing for threads to conflict over.
    - Local variables - Any variables defined inside a method are are stored on the thread's stack and are accessible only to the executing thread.

- Example where each thread calling the add method gets its own copy of result hence no shared state -
```
public class Calculator {
    public int add(int a, int b) {
        int result = a + b; // 'result' is a local variable
        return result;
    }
}
```

- Each thread has its own call stack. When a thread calls the add method, a stack frame is created for that method call containing -
    - Method parameters (a and b) - Stored as local variables.
    - Local variable (result) - Also stored in the same stack frame.

## Atomicity

- **Atomicity** means that an operation is uninterruptible and appears as a single unit of execution. From the perspective of other threads, the operation either it is successfully executed or rolled back.
- Atomicity is required for operations like Read-Modify-Write (RMW) i.e. involve reading a value, modifying it, and writing it back -
    - Example - `count++` - where the new state (incremented value) is derived from the previous state. If the previous state changes unexpectedly (e.g., another thread modifies it), the new state might be incorrect.
    - The increment operation `++count` is shorthand for three separate steps -
        1. `LOAD count` - The current value of count is fetched from memory (or possibly from a CPU register or cache).
        2. `ADD 1` - Increment the value in the register by 1. This happens in the CPU's arithmetic logic unit (ALU).
        3. `STORE count` - Write the updated value back to memory (or the register holding the value of count).
    - Thread interleaving - assume `count = 5` initially and two threads, Thread A and Thread B, execute `++count` simultaneously -
        - Thread A -
            - Reads count (value = 5).
            - Adds 1 (new value = 6).
        - Thread B (before Thread A writes back) -
            - Reads count (value = 5, because Thread A hasn’t written yet).
            - Adds 1 (new value = 6).
        - Thread A writes -
            - Updates count to 6.
        - Thread B writes -
            - Overwrites count to 6.
        - Result - The final value of count is 6, but it should have been 7 since two increments were performed.

- To make `count++` thread safe in this case, we can use **Atomic Variables** such as `AtomicInteger` which can make the increment operation atomic -
```
import java.util.concurrent.atomic.AtomicInteger;

private AtomicInteger count = new AtomicInteger(0);

public void increment() {
    count.incrementAndGet(); // Atomically increments the value
}
```

> [!WARNING]
> A single atomic operation may not suffice when multiple operations depend on a shared state.
> Example -
> ```
> if (balance >= 100) {
>     balance -= 100;
> }
> ```
> Even if the subtraction (`balance -= 100`) is atomic, the combined check (`if`) and subtraction are not, leading to potential _race conditions_.

## Race Condition

- A race condition occurs when the correctness of a computation depends on the timing or interleaving of threads, meaning -
    - Multiple threads access shared resources simultaneously.
    - The outcome depends on the specific order in which the threads execute.
    - The system behaves unpredictably, sometimes producing correct results and other times failing.
- One hallmark of race conditions is the stale observations i.e. the decisions or actions are based on observations that may no longer be valid.
- Types of race conditions -
    1. **Check-Then-Act** -
        - This occurs when a thread checks a condition (e.g., a file doesn't exist, or an object is not initialized).
        - Acts based on that condition (e.g., creates the file or initializes the object).
        - However, between the check and the act, another thread may have modified the state, causing unexpected results.
        - Example -
        ```
        class SharedResource {
            public static SharedResource getInstance() {
                if (instance == null) {                 // Thread A and Thread B could both see this as null
                    instance = new SharedResource();    // Both threads could initialize the instance
                }
                return instance;
            }
        }
        ```
        - To fix this race condition, synchronization can be used to ensure only one thread can initialize the instance at a time -
        ```
        public static synchronized SharedResource getInstance() {
            if (instance == null) {
                instance = new SharedResource();
            }
            return instance;
        }
        ```
        - or with **Double-Checked Locking** -
        ```
        public static SharedResource getInstance() {
            if (instance == null) {                  // First check (no locking)
                synchronized (SharedResource.class) {
                    if (instance == null) {          // Second check (with locking)
                        instance = new SharedResource();
                    }
                }
            }
            return instance;
        }
        ```
        - In the above example -
            - Both _Thread A_ and _Thread B_ enter the `getInstance()` method simultaneously.
            - Since instance is initially `null`, both threads pass the `if (instance == null)` check.
            - Each thread proceeds to create a new `SharedResource` object, leading to two instances being created.

    2. **Read-Modify-Write (RMW) Race Condition** - same as `count++` example described in Atomicity.

### Data Races vs Race Conditions

- **Data Race** - occurs when two threads access a shared variable -
    - At least one of the accesses is a write.
    - The accesses are not properly synchronized.
    - Example: A thread writes to a field while another thread reads it, without synchronization.
- **Race Condition** - A broader term that includes data races but also encompasses logic errors caused by incorrect assumptions about timing or interleaving of threads.

> [!NOTE]
> All data races are race conditions, but not all race conditions are data races.

- Example of a Race Condition without a Data Race - Two threads access a thread-safe queue. Thread A observes the queue is not empty and decides to dequeue an item, but before it does, Thread B dequeues it first. The queue is now empty, and Thread A’s action fails.

> [!WARNING] 
> Synchronizing the entire method ensures thread safety but can hurt performance -
>   - Only one thread can use the method at a time, even if multiple threads are performing non-conflicting operations.
>   - This can degrade responsiveness and scalability in high-concurrency environments.

## Locking

- Locking is a synchronization mechanism which ensures that only one thread can access a shared resource or critical section of code at a time. 
- It helps prevent race conditions and ensures thread safety when multiple threads or processes are accessing shared data concurrently.
- Key features -
    - **Mutual exclusions** - ensures that only one thread can execute a critical section of code that accesses shared resources at a time.
    - **Critical section** - section of code that accesses shared data and must be executed by only one thread at a time to avoid conflicts.
    - **Lock Acquisition** - a thread requests or acquires a lock before entering the critical section to guarantee exclusive access.
    - **Lock Release** - once the thread finishes executing the critical section, it releases the lock, allowing other threads to acquire it.

## Types of Locking -
### 1. **Intrinsic Locks** or **Monitor Locks**
- Intrinsic locks are Java's built-in synchronization mechanism. Every object in Java has an associated intrinsic lock (or monitor) that can be used to synchronize access to shared resources.
- Key features -
    - **One Lock per Object** - Each object in Java has a unique intrinsic lock. Threads acquire this lock when they enter a synchronized block or method associated with the object.
    - **Mutual Exclusion** - At most, one thread can hold an intrinsic lock at any given time. Other threads trying to acquire the same lock must wait until the lock is released.
    - **Thread Blocking** - If a thread tries to acquire a lock that is already held by another thread, it will block until the lock becomes available.
    - **Reentrancy** - Intrinsic locks are reentrant, meaning that a thread that already holds the lock can acquire it again without getting blocked. This is useful for methods that call other synchronized methods within the same object.
    - **Automatic Release** - The lock is managed by the JVM implicitly and is automatically acquired and released when the thread exits the synchronized enters or block or method, either normally or by throwing an exception.

- Limitations -
    - **Coarse-Grained Locking** - Using a single intrinsic lock for large critical sections can reduce concurrency, as only one thread can access the locked resource at a time.
    - **No Timeout or Try-Lock** - Intrinsic locks do not support timeout-based acquisition or checking if the lock is available without blocking.
    - **Potential Deadlocks** - Improper usage of multiple intrinsic locks (nested `synchronized` blocks) can lead to deadlocks if two or more threads hold locks and wait for each other's locks.

- Examples -
1. **Synchronized Instance method** - The lock is the current instance of `Counter` class. Only one thread can execute `increment` or `getCount` at a time for the same object. When you mark a method as `synchronized`, the lock is implicitly associated with the object instance (i.e., `this`) on which the method is being called -
```
public class Counter {
    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public synchronized int getCount() {
        return count;
    }
}
```

2. **Synchronized Static Method** - The lock is the Class object (`GlobalCounter.class`). Hence, if two threads are calling different static methods on the same class, they can execute concurrently without blocking each other, even if both methods are synchronized because the lock that is acquired for synchronization is not the instance of the object, but the Class object that represents the class in the JVM. -
```
public class GlobalCounter {
    private static int count = 0;

    public static synchronized void increment() {
        count++;
    }

    public static synchronized int getCount() {
        return count;
    }
}
```

3. **Synchronized Block** - The `this` object (current instance of `BankAccount`). Only one thread can execute code within the synchronized block at a time with same lock -
```
public class BankAccount {
    private int balance = 0;

    public void deposit(int amount) {
        synchronized (this) {
            balance += amount;
        }
    }

    public int getBalance() {
        synchronized (this) {
            return balance;
        }
    }
}
```

4. **Explicit Locking with Another Object** - The explicitly defined `lock` object. This approach separates the locking mechanism from the object itself, providing more flexibility -
```
public class AccountManager {
    private final Object lock = new Object();
    private int balance = 0;

    public void deposit(int amount) {
        synchronized (lock) {
            balance += amount;
        }
    }

    public int getBalance() {
        synchronized (lock) {
            return balance;
        }
    }
}
```

> [!WARNING]
> Primitive types such as `int`, `double` cannot be used as locks in Java because they are not objects and do not have a monitor object associated with them.

### 2. **Extrinsic Locks**
- Extrinsic locks are locks that are not inherently tied to a specific object. They are separate from the objects being locked and can be applied to any shared resource.
- These are usually implemented using explicit lock objects provided by classes like `java.util.concurrent.locks.ReentrantLock`.
- These locks allow for more flexible and sophisticated control over thread synchronization.
- Key features -
    - **Locking on External Objects** - not tied to the object being worked on (like this or ClassName.class), but instead, it is an explicit lock object that you manage externally from the object itself.
    - **Fine-grained Lock Control** - let you manually control locking, and even try to acquire the lock without blocking indefinitely, with methods like `tryLock()`.
    - **Locking Specific Resources** - can use extrinsic locks to synchronize on specific resources independently of the objects being worked on. For example, you could have multiple locks for different resources (e.g., separate locks for account number, transaction data, etc.) rather than synchronizing on the entire object.
    - **No Deadlocks or Performance Issues** - when used properly, can allow fine-grained control of synchronization, leading to better performance and avoiding issues like deadlocks.
    
- Usage -
    1. **External Lock Object** - create a lock object, typically a ReentrantLock, that is separate from the object you're synchronizing -
    ```
    ReentrantLock lock = new ReentrantLock();
    ```

    2. **Locking a Resource** - acquire the lock explicitly in the code where you need it -
    ```
    lock.lock();
    try {
        balance += amount;  // Critical section of code where you need synchronization
    } finally {
        lock.unlock();      // Ensure the lock is released even if an exception occurs
    }
    ```

    3. **Control Over Locking** - With extrinsic locks, you have full control over the locking behavior. For example, you can use `tryLock()` to attempt to acquire the lock without blocking indefinitely, or `lockInterruptibly()` to acquire a lock and be able to interrupt if the thread is waiting for too long -
    ```
    if (lock.tryLock()) {
        try {
            // Critical section code
        } finally {
            lock.unlock();
        }
    } else {
        // Handle the case where the lock is unavailable
    }
    ```

- Limitations -
    - Requires manual lock management (you must ensure the lock is released, typically using a finally block).
    - Can lead to deadlocks if locks are not managed carefully.

### 3. **Optimistic Locks**
- Optimistic locking is based on the idea that conflicts between threads will be rare. In this approach, a thread proceeds with its operation assuming no other thread will interfere. 
- Before committing changes, the thread checks if any other thread has modified the resource, and if a conflict is detected, it retries the operation.
- Typically involves versioning or timestamps on data, and the thread checks if the data was modified before committing.
- Example -
    - `StampedLock` (with optimistic reading)
    - Versioned data in databases or software frameworks.

    