# Locking

## `synchronized`

- Built-in Java locking mechanism.
- Used to restrict access to a critical section or an entire method to one thread at a time.
- Implemented using an object's monitor (intrinsic lock).
- Two ways to use it -

  - **Synchronized Method (Monitor Lock)** -

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

  - **Synchronized Block (Custom Lock)** -

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
