


## Atomic Operations

- All reference assignments are atomic, eg - all the getters and setters are atomic.
- All assignments to primitive types except `long` and `double` are atomic -
  - Even with 64-bit CPU, it is possible that a write to `long` or `double` will actually be completed in two operations by the CPU - write to lower 32-bits and then to upper 32-bits.
  - Solution - declare them as `volatile` - they are guranteed to be performed by a single hardware operation.

## Data Race

```
public class SharedClass {
  int x = 0;
  int y = 0;

  public void increment() {
    x++;
    y++;
  }

  public void checkForDataRace() {
    if (y > x) {
      throw new RuntimeException("Not possible");       // this will be printed out multiple times
    }
  }
}
```

- Compiler and CPU may execute the instructions out of order to optimize performance and utilization - while maintaining the logical correctness of the code.
- These optimizations include -
  - Branch prediction (optimized loops, if statements etc)
  - Vectorization - parallel instruction execution (SIMD)
  - Prefetching instructions - better cache performance
- CPU re-arranges instructions for better hardware unit utilizations.
- Solutions -
  - Synchronization
  - Declare shared variables as `volatile` - gurantees that code that comes before/after access a volatile variable, will be executed before/after that access instruction.

## Lock Strategies

- __Coarse-Grained Locking__ - 
  - One lock for all the shared resources.
  - Easier to manage.

- __Fine-Grained Locking__ -
  - Separate lock for all the shared resources.
  - More parallelism and less contention.
  - Can cause deadlocks.

## Deadlock

- Situation where every thread is trying to make progress, but cannot because they are waiting for another thread to progress.

- Deadlock example -
  - Thread 1 - 
  ```
  lock(A)
    lock(B)
      delete(A, item)
      add(B, item)
    unlock(B)
  unlock(A)
  ```

  - Thread 2 - 
  ```
  lock(B)
    lock(A)
      delete(B, item)
      add(A, item)
    unlock(A)
  unlock(B)
  ```

  - Order of events causing deadlock -
  ```
  Thread 1 - lock(A)
  Thread 2 - lock(B)
  Thread 2 - lock(A)    ❌
  Thread 1 - lock(B)    ❌
  ```

- __Conditions for Deadlock__ -
  - Mutual Exclusion - Only one thread have exclusive access to a resource at any given moment.
  - Hold and Wait - At least one thread is holding a resource and is waiting for another.
  - Non-preemptive Allocation - A resource is released only after the thread is done using it.
  - Circular Wait - A chain of at least two threads each one is holding one resource and waiting for another.

- __Solution__ -  
  - Avoid Circular Wait - Enforce a strict order in lock acquisition, eg -
    - Thread 1 -
    ```
    lock(A)
      lock(B)
        ...
      unlock(B)
    unlock(A)
    ```

    - Thread 2 -
    ```
    lock(A)
      lock(B)           // Locking order is same as Thread 1 - no deadlocks!
        ...
      unlock(A)         // Unlocking order doesn't matter
    unlock(B)
    ```

- Other solutions -
  - Deadlock detection using Watchdog - 
    - Can be implemented in many ways.
    - Example, in microcontrollers, it's usually implemented by a low level routine that periodically checks the status of particular register.
    - That register needs to be updated by every thread, every few instructions, and if the watchdog detects that the register hasn't been updated, it knows that the threads are non-responsive and will simply restart them.
    - In similar way, a watchdog can be implemented as a different thread, which will detect deadlock threads and will try to interrupt them - but not possible with synchronized.
  - tryLock operations - checks if a lock is already acquired by another thread before actually trying to acquire a lock, and possibly getting suspended. The synchronized keyword does not allow a suspended thread to be interrupted.

## Reentrant Lock

- Requires explicit locking and unlocking.
- With `synchronized`, locking happens in the beginning of the synchronized block, and unlocking happens when the block ends.
- In reentrant locking, we use `ReentrantLock` object which implements the `Lock` interface; and then call `lock()` / `unlock()` explicitly.
```
Lock lockObject = new ReentrantLock();
Resource resource = new Resource();

public void method1() {
  lockObject.lock();
  use(resource);
  lockObject.unlock();
}
```
  
- Problems -
  - If we forget to unlock the `lockObject`, it would leave the `lockObject` locked forever.
  - If the critical section throws an exception, the `lockObject.unlock()` method will never be reached - to overcome this, the common pattern is to wrap them in try-finally -
  ```
  lockObject.lock();
  try {
    someOperations();
    return value; 
  } finally {
    lockObject.unlock();
  }
  ```

- `ReentrantLock` querying methods (for testing) -
  - `getQueuedThreads()` - returns a list of threads waiting to acquire a lock.
  - `getOwner()` - returns the thread that currently owns the lock.
  - `isHeldByCurrentThread()` - queries if the lock is held by the current thread.
  - `isLocked()` - queries if the lock is held by any thread.

> [!TIP]
> The `ReentrantLock` and `synchronized` by default do NOT gurantee any fairness i.e. if we have many threads waiting to acquire a lock on a shared object, we may have a situation that one thread gets to acquire the lock multiple times, while another thread may be starved.
>
> To gurantee fairness - `ReentrantLock(true)` - but it may reduce the throughput of the application because the lock acquisition may take longer.

- __`lockInterruptibly()`__ -
  - Interrupts the suspended thread -
    ```
    public class SomeThread extends Thread {
      @Override
      public void run() {
        while(true) {
          try {
            lockObject.lockInterruptibly();
            // ...
          } catch(InterruptedException) {
            cleanupAndExit();
          }
        }
      }
    }
    ```

  - Useful in the implementation of watchdog for deadlock detection and recovery.
  
- __`tryLock()`__ -
  - `boolean tryLock(long timeout, TimeUnit unit)`
    - Returns `true` and acquires a lock if available.
    - Returns `false` and does not get suspended, if the lock is unavailable.

  ```
  if (lockObject.tryLock()) {
    try {
      useResource();
    } finally {
      lockObject.unlock();
    }
  }
  ```

## ReentractReadWriteLock

- Synchronized and ReentrantLock do not allow multiple readers to access a shared resource concurrently.
- Generally, it's not a problem - read operations are usually fast and if we keep the critical section short, the changes of contention over a lock are minimal.
- But when reading from complex data structures, mutual exclusion of reading threads negatively impacts the performance.
- ReentrantReadWriteLock solves this issue -
```
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

// Provides two internal locks
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// Modifying a shared resource
writeLock.lock();
try {
  modifySharedResource();
} finally {
  writeLock.unlock();
}

// Reading a shared resource
readLock.lock();
try {
  readFromSharedResource();
} finally {
  readLock.unlock();
}
```

- Multiple threads can acquire the read lock simultaneously. The read lock internally keeps count of how many readers threads are holding at any given moment.
- Only a single thread is allowed to lock a write lock. 

- Mutual exclusion between readers and writers -
  - If a _write lock_ is acquired, no thread can acquire a _read lock_.
  - If at least one thread holds a _read lock_, no thread can acquire a _write lock_.

## Semaphore

- Used for inter-thread communication.
- Can be used to restrict the number of users to a particular resource or a group of resources, unlike the locks that allows only one user per resource.
- Initializing a semaphore with number of permits -
```
Semaphore semaphore = new Semaphore(NUMBER_OF_PERMITS);
```

- Every time thread calls its `acquire()` method, it takes a permit if it's available and moves to the next instruction -
```
semaphore.acquire();            // 1 permit acquired
```

- A thread can also acquire more than 1 permit at a time -
```
semaphore.acquire(5);           // 5 permits acquired
```

- Releasing permits - 
  - 1 permit - `semaphore.release()`
  - More than 1 permit - `semaphore.release(5)`

- Calling the `acquire()` method on a semaphore that doesn't have any more permits left, will result in blockin the thread until a semaphore is release by another thread.

> [!TIP]
> A lock is particular case of a semaphore, with only one permit to give.

- __Semaphores vs Locks__ -
  - Semaphores doesn't have a notion of owner thread.
  - Many threads can acquire a permit.
  - The same thread can acquire the semaphore multiple times.
  - The binary semaphore (initialized with 1) is _not reentrant_.
  - Semaphore can be release by any thread - even can be released by a thread that hasn't actually acquired it.

- __Producer-Consumer Problem__ -
  ```
  Semaphore full = new Semaphore(0);
  Semaphore empty = new Semaphore(1);
  Item item = null;

  // Producer
  while(true) {
    empty.acquire();
    item = produceNewItem();
    full.release();
  }

  // Consumer
  while(true) {
    full.acquire();
    consume(item);
    empty.release();
  }
  ```

  - The consumer thread first try to acquire the `full` semaphore & will block, and then wait for the producer to produce an item.
  - The producer is first going to acquire `empty` semaphore, and then produce an `item` and store it in shared variable.
  - After that, the producer will release the `full` semaphore - which will result in waking up the consumer thread and allowing it to consume the `item`.
  - Now, if the producer proceeds before the consumer was done consuming the item, then it would try to acquire the `empty` semaphore.
  - The producer would get blocked and wait until the consumer is done consuming the `item`.
  - Then when the consumer is done, it will release the `empty` semaphore which would wake up the producer and allow it produce a new item.
  - If the consumer is faster than the producer, it would spend most of the time in the suspended mode and will not consume any CPU.
  - And if the consumer is slower than the producer, it is guranteed that the producer will not produce more items until the consumer is done consuming them.
  - This implementation will work only with a single producer and consumer.

- __Multiple producers and consumers__ -
  ```
  Semaphore full = new Semaphore(0);
  Semaphore empty = new Semaphore(CAPACITY);
  Queue queue = new ArrayDequeue();
  Lock lock = new ReentrantLock();

  // producer
  while(true) {
    Item item = produce();
    empty.acquire();
    lock.lock();
    queue.offer(item);
    lock.unlock();
    full.release();
  }

  // consumer
  while(true) {
    full.acquire();
    lock.lock();
    Item item = queue.poll();
    lock.unlock();
    consume(item);
    empty.release();
  }
  ```

  - Replace the `item` with a queue of items.
  - Then we can decide on the `queues` capacity and initialize the `empty` semaphore with that `CAPACITY` value.
  - To protect the `queue` from concurrent access by multiple consumers and multiple producers, we can add any type of lock, for instance `ReentrantLock`.
  - In the producer, we produce the items separately and offer it to the `queue`, which is shared between the producer and the consumer.
  - Before offering the `item` to the `queue`, we `lock` the `queue` and after the `item` is added to the `queue`, we `unlock` it.
  - In the consumer, we can first get the `item` from the `queue` and the consume it.
  - And just like in the producer, we `lock` the acces to the `queue` before calling the `pull` method and `unlock` it after we got the `item` from the `queue`.

- Applications of producer-consumer model -
  - Actors
  - Sockets
  - Every inter-thread communication library of framework would use a variation of this pattern.

## Condition

- Condition variable is always associated with a lock.
- The lock ensures that checking the condition and modifying the shared variables used in the condition are performed atomically.
```
Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();
```

- Awaiting -
  - `void await()` - unlock lock, wait until signalled.
  - `long awaitNanos(long nanosTimeout)` - wait no longer than `nanosTimeout`.
  - `boolean await(long time, TimeUnit unit)` - wait no longer than `time`, in given time units.
  - `boolean awaitUntil(Date deadline)` - wake up before the deadline date.

- Signal -
  - `void signal()` - wakes up only a single thread, waiting on the condition variable.
  - A thread that wakes up has to reacquire the lock associated with the condition variable.
  - If no thread is currently waiting on the condition variable, the signal method does not do anything.
  - `void signalAll()` - broadcast a signal to all threads currently waiting on the condition variable.
  

- Example - a UI thread is waiting for user credentials and another thread fetches the username/password from the database (to keep the UI thread unlocked) -
```
String username = null, password = null;

// Thread 1
lock.lock();
try {
  while(username == null || password = null) {
    condition.await();
  }
} finally {
  lock.unlock();
}

doStuff();

// UI Thread
lock.lock();
try {
  username = userTextbox.getText();
  password = passwordTextbox.getText();
  condition.signal();
} finally {
  lock.unlock();
}
```

## `wait()`, `notify()`, `notifyAll()`

- The `Object` class contains the following methods -
  - `public final void wait() throw InterruptedException`
  - `public final void notify()`
  - `public final void notifyAll()`
- Every Java class inherits from the `Object` class.
- We can use any object as a condition variable and a lock (using the `synchronized` keyword).
- `wait()` -
  - Causes the current thread to wait until another thread wakes it up.
  - In the wait state, the thread is not consuming any CPU.
- Two ways to wake up the waiting thread -
  - `notify()` - wakes up a _single_ thread waiting on that object. If multiple threads are waiting on the same object, any one thread is chosen randomly.
  - `notifyAll()` - wakes up all the threads waiting on that object.

> [!NOTE]
> To call `wait()`, `notify()` or `notifyAll()`, we need to acquire the monitor of that object (use `synchronized` on that object).

