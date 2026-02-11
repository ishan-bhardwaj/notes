# Multi-threading

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
- Context switching with too many threads causes _thrashing_ which means spending more time in management than real productive work.

- Threads consume less resources than process. Therefore, context switching between threads from the same process is cheaper than context switching between processes.

- Threads are much faster to create and destroy than processes.

## Thread Scheduling

- __First Come First Serve (FCFS)__ -
  - Executes threads based on their order of arrival.
  - Problem - long thread can cause starvation.

- __Shortest Job First (SJF)__ -
  - Executes threads based on their duration - shortest job is executed first.
  - Problem - If shorter jobs keep coming all the time then the longer job will never be executed.

- __Epochs based__ -
  - Used in most OS.
  - The OS divides the time into moderately size pieces called _epochs_.
  - In each epoch, the OS allocates a different time slice for each thread.
  - Note that not all threads get to run or complete in each epoch.
  - The decision on how to allocate the time for each thread is based on dynamic priority that OS maintains for each thread - $Dynamic Priority = Static Priority + Bonus$, where $Bonus$ can be negative.
  - Static priority is set by the developer programmatically.
  - Bonus is adjusted by the OS in every epoch for each thread.
  - Using dynamic threads, the OS will give preference for interactive threads (such as UI threads).
  - OS will also give preference to threads that did not complete in the last epochs, or did not get enough time to run - preventing starvation.

## Thread Creation

- Creating a thread (`java.lang.Thread`) -
```
Thread thread = new Thread(new Runnable() {
  @Override
  public void run() {
    System.out.println("Inside thread: " + Thread.currentThread().getName());
  }
});

// or - using lambda
Thread thread = new Thread(() -> {
  // code that will run in a separate thread
});
```

- To start the thread - `thread.start()` - this will instruct the JVM to create a new thread and pass it to the OS.

- `Thread.currentThread()` - returns instance of the current thread.
- `thread.getId()` / `thread.getName()` - returns the id / name of the current thread.
- `Thread.sleep(1000)` - puts the current thread to sleep for 1000 ms. During this time, this thread will not be consuming any CPU.
- `thread.setName("worker-thread")` - sets the name of the thread.
- `thread.setPriority(1)` - sets the priority of the thread -
  - Priority ranges from 1 (min) to 10 (max). 
  - `Thread.MIN_PRIORITY` / `Thread.MAX_PRIORITY` - min/max priority.
  - `Thread.NORM_PRIORITY` - default

- To get the priority value - `thread.getPriority()`

- Setting an exception handler for the entire thread - the handler will be called if an exception was thrown inside the thread and did not get caught anywhere -
```
thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
  @Override
  public void uncaughtException(Thread t, Throwable e) {
    ...
  }
})
```

- Another way to create a thread -
```
public class NewThread extends Thread {
  @Override
  public void run() {
    System.out.println("Inside thread: " + this.getName());
  }
}

// Instatiating
Thread thread = new Thread();
thread.start();
```

> [!TIP]
> `Thread` class implements `Runnable` interface.

> [!TIP]
> If any thread is running and `main` thread finishes, the application will still be in the running state.

## Thread Coordination

- We can interrupt a thread -
  - If the thread is executing a method that throws an `InterruptedException`, eg - `Thread.sleep()`.
  ```
  public class BlockingTask implements Runnable {
    @Override
    public void run() {
      try {
        Thread.sleep(500000);
      } catch {
        System.out.println("Exiting blocking thread");
        return ;                                            // necessary
      }
    }
  }

  Thread thread = new Thread(new BlockingTask());
  thread.start();
  thread.interrupt();
  ```

  - If the thread's code is handling the interrupt signal explicitly -
    - Note that if the `BlockingTask` does not have logic to handle the interrupt signal, then `thread` will not be interrupted and continues running.
    - To handle the interrupt signal explicitly, find the hotspots in the code and add a check if the thread is interrupted -
    ```
    if (Thread.currentThread().isInterrupted()) {
      System.out.println("Prematurely interrupted exception");
    }
    ```

- __Daemon Threads__ -
  - Background threads that do not prevent the application from exiting if the main thread terminates.
  - To set a thread as daemon - `thread.setDaemon(true)`

- __`thread.join()`__ -
  - Waits for the `thread` to complete.
  - Provides more control over independent thread.
  - Safely collect and aggregate results.
  - Gracefully handle runaway threads using `thread.join(timeout)` -
    - Example - `thread.join(2000)` - waits for 2 seconds for `thread` to complete, otherwise return forcefully.  

## Performance in multi-threading

- __Latency__ -
  - the time to completion of a task. 
  - measured in time units.

- __Throughput__ -
  - the amount of tasks completed in a given period.
  - measured in tasks/time unit.

- We divide the tasks into `N` parallel units, so theoretically reducing the latency by `N`.
  - `N` = #cores is optimal, only if all threads are runnable and can run without interruption (no IO blocking calls/sleep etc).
  - The assumption is nothing else is running that consumes a lot of CPU.
  - These assumptions are rare in real world so we will never achieve the optimal utilisation, but we can get close to it.

- __Hyperthreading__ -
  - Means a single physical core can run two threads at a time which is achieved by having some hardware units in a physical core duplicated.
  - So, the two threads run in parallel and some hardware units are shared - this means we can run two threads closer to 100% in parallel, but never exactly 100%.

- __Inherent costs of parallelization and aggregation__ -
  - Breaking tasks into multiple subtasks
  - Thread creation and passing tasks to threads
  - Time between `thread.start()` to thread getting scheduled by the OS
  - Time until the last thread finishes and signals
  - Time until the aggreating thread runs
  - Aggregation of the results into a single artifact

- __Task classification__ -
  - Parallelizable tasks - tasks that are inherently parallelizable and can be easily broken into sub tasks.
  - Sequential tasks - the unbreakable tasks that we are just forced to run on a single thread from start to finish.
  - Partially parallelizable, partially sequential (most common) - tasks which can be partially broken into sub tasks and partially we have to run them sequentially.

## Thread Pooling

- Creating the threads once and reusing them for future tasks.
- Once the threads are created, they sit in the pool and tasks are distributed among the threads through a queue.
- If all the threads are busy, the tasks will stay in the queue and wait for a thread to become available.
- JDK comes with a few implementations of thread pools -
  - Fixed Thread Pool Executor -
    - Creates a thread pool with a fixed number of threads in the pool.
    - Also comes with a built-in queue.
    - Example -
    ```
    int numThreads = 4;
    Executor executor = Executors.newFixedThreadPool(numThreads);
    ```

    - To add a task to the queue and have it executed by one of the threads, we simply pass a runnable task into the `execute` method -
    ```
    Runnable task = executor.execute(task);
    ```

## Data sharing between threads

- __Stack__ -
  - Stack is a memory region where -
    - methods are called
    - arguments are passed
    - local variables are stored
  - The method calls and their arguments & variables are stored together in a stack frame.
  - State of each thread's execution = Stack + Instruction Pointer
  - Stack Properties -
    - All variables belong to the thread executing on that stack.
    - Statically allocated when the thread is created.
    - The stack's size is fixed and is relatively small (platform specific).
    - If our calling hierarchy is too deep, we may get a `StackOverflowException` - risky with recursive calls.

- __Heap__ -
  - Heap is a shared memory region that belongs to the process.
  - Stores -
    - Objects (anything created with the `new` operator like String, Object, Collection etc).
    - Member of classes - even its primitive members
    - Static variables
  - Heap memory management -
    - Governed and managed by Garbage Collector.
    - Objects stay as long as we have at least one reference to them.
    - Member of classes exist as long as their parent objects exist (i.e. same lifecycle as their parents).
    - Static variables stay forever.

> [!TIP]
> Object References -
>   - If references are declared as local variables inside a method, they are allocated on the stack.
>   - If they are member of the class, then they are allocated on the heap together with their parent objects.
>
> Objects are always allocated on the heap.

## Critical Section

- Section of the code which cannot be executed by more than one thread at a time.

- __Synchronized__ -
  - Locking mechanism.
  - Used to restrict access to a critical section or entire method to a single thread at a time.
  - Two ways to use it -
    - Synchronized - Monitor -
      - Define methods as synchronized -
      ```
      public synchronized void method1() {
        // critical section
      }
      ```

      - Note that if multiple methods (say, `method1` and `method2`) of a class are synchronized, and thread `A` is executing `method1`, then thread `B` will not be allowed to execute both methods because `synchronized` is applied per object - called a monitor.

    - Synchronized - Lock -
      - Restrict access section within `synchronized` -
      ```
      Object lock = new Object();      // this will serve as a lock - can be any object

      public void method1() {
        synchronized(lock) {
          // critical section
        }
      }
      ```

      - We can have different locking objects for different critical sections, so two threads can different locks to enter into those different critical sections at the same time.
      - Synchronized Lock is _Reentrant_ - a thread cannot prevent itself from entering a critical section i.e. if thread `A` is accessing a synchronized method while already being in a different synchronized method or block, it will be able to access that synchronized method with no problem.

> [!TIP]
> Synchronized monitor is logically equivalent to using `this` object with `synchronized` i.e. -
>
> ```
> public void method1() {
>   synchronized(this) {
>     // critical section (the entire content of the method)  
> }
> }
> ```

## Atomic Operations

- All reference assignments are atomic, eg - all the getters and setters are atomic.
- All assignments to primitive types except `long` and `double` are atomic -
  - Even with 64-bit CPU, it is possible that a write to `long` or `double` will actually be completed in two operations by the CPU - write to lower 32-bits and then to upper 32-bits.
  - Solution - declare them as `volatile` - they are guranteed to be performed by a single hardware operation.

## Race Condition

- Condition when multiple threads are accessing a shared resource.
- At least one thread is modifying the resource.
- The timing of threads scheduling may cause incorrect results.
- The core of the problem is non-atomic operations performed on the shared resource.
- Solution - 
  - Identify critical section where the race condition is happening.
  - Protect that critical section by a `synchronized` block.

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

- `void await()` - unlock lock, wait until signalled.
- `long awaitNanos(long nanosTimeout)` - wait no longer than `nanosTimeout`.
- `boolean await(long time, TimeUnit unit)` - wait no longer than `time`, in given time units.
- `boolean awaitUntil(Date deadline)` - wake up before the deadline date.

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

