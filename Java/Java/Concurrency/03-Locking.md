
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
