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


