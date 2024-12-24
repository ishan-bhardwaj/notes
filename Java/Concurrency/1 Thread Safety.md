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
        - In the above example -
            - Both _Thread A_ and _Thread B_ enter the `getInstance()` method simultaneously.
            - Since instance is initially `null`, both threads pass the `if (instance == null)` check.
            - Each thread proceeds to create a new `SharedResource` object, leading to two instances being created.

    2. Read-Modify-Write (RMW) Race Condition - same as `count++` example described in Atomicity.

### Data Races vs Race Conditions
- **Data Race** - occurs when two threads access a shared variable -
        - At least one of the accesses is a write.
        - The accesses are not properly synchronized.
        - Example: A thread writes to a field while another thread reads it, without synchronization.
- **Race Condition** - A broader term that includes data races but also encompasses logic errors caused by incorrect assumptions about timing or interleaving of threads.

> [!NOTE]
> All data races are race conditions, but not all race conditions are data races.

- Example of a Race Condition without a Data Race - Two threads access a thread-safe queue. Thread A observes the queue is not empty and decides to dequeue an item, but before it does, Thread B dequeues it first. The queue is now empty, and Thread A’s action fails.