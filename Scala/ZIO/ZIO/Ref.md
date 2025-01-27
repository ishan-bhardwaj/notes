## Ref

- **Ref** allow us to describe mutable states in a purely functional and concurrent way. It represents some piece of mutable state that is shared between fibers, so it is extremely useful for communicating between fibers.
- Ref is internally backed by an `AtomicReference` - a mutable data structure safe for concurrent access which is not exposed directly, so the only way to access it is through the combinators (`modify`, `get`, `set`, `update`) which all return effects.
- `Ref.make` is the only way to create a Ref instance.

- In the implementation of `Ref#modify`, we use a _compare-and-swap_ approach where we get the value of the `AtomicReference`, compute the new value using the specified function, and then only set the `AtomicReference` to the new value if the current value is still equal to the original value i.e. if no other fiber has modified the `AtomicReference` between us reading from and writing to it. If another fiber has modified the `AtomicReference`, we just retry.
```
def modify[B](f: A => (B, A)): UIO[B] =
    ZIO.succeed {
        var loop = true
        var b: B = null.asInstanceOf[B]
        while (loop) {
            val current = atomic.get
            val tuple = f(current)
            b = tuple._1
            loop = !atomic.compareAndSet(current, tuple._2)
        }
    }
```

- `Ref#modify` is used in implementing other combinators on `Ref`.

> [!WARNING]
> `Ref` operations are atomic, but not their composition. Eg - `ref.update(_ + 1)` is atomic, but if we use `ref.get.flatMap(n => ref.set(n + 1))` is not atomic.
> Also, having a mutable state in two different references will not be atomic.

> [!TIP]
> To compose Ref operations, we use Software Transactional Memory (STM).

> [!TIP]
> Best practises when using Refs -
> - Put all pieces of state that need to be “consistent” with each other in a single Ref.
> - Always modify the Ref through a single operation.

## `Ref.Synchronized`

- We can't perform effects inside the `Ref#modify` i.e. the function parameter `f` must be a pure function because _compare-and-swap_ operations running in a loop is safe when computing the new value has no side effects. We can increment an integer once or a hundred times, and it will be exactly the same.

- However, evaluating an effect many times is not the same as evaluating it a single time. Evaluating `Console.printLine("Hello, World!")` a hundred times will print `“Hello, World!”` to the console a hundred times versus evaluating it once will only print it a single time. So if we perform side effects in `Ref#modify`, they could be repeated an arbitrary number of times in a way that would be observable to the caller and would violate the guarantees of Ref that updates should be performed atomically and only a single time.

- Workaround - 
```
def updateAndLog[A](ref: Ref[A])(f: A => A): URIO[Any, Unit] =
    ref.modify { oldValue =>
        val newValue = f(oldValue)
        ((oldValue, newValue), newValue)
    }.flatMap { case (oldValue, newValue) =>
        Console.printLine(s"updated $oldValue to $newValue").orDie
    }
```

- `Ref.Synchronized` does the similar thing - 
```
trait Synchronized[A] {
    def modifyZIO[R, E, B](f: A => ZIO[R, E, (B, A)]): ZIO[R, E, B]
}
```

- Internally, `Ref.Synchronized` is implemented using a Semaphore to ensure that only one fiber can interact with the reference simultaneously.

- `Ref.Synchronized` is not as fast as `Ref`, hence we should do the minimum work possible within `modify` operation of `Ref.Synchronized` (eg - creating a `Ref`, forking a fiber) and perform additional required effects based on the return value of the modify operation because Effects occurring within the modify operation can only occur one at a time, whereas effects outside of it can potentially be done concurrently.

## `FiberRef`

- `FiberRef `allows fibers to maintain fiber-local state.
- Each fiber can have its own independent value for the `FiberRef`. When a fiber is forked, it inherits the `FiberRef` values from its parent.
- Modifications to the `FiberRef` in the child fiber do not affect the parent. However, custom merge logic can be applied when a fiber joins back to its parent.

- Creating `FiberRef` -
```
def make[A](
    initial: A,
    fork: A => A,
    join: (A, A) => A
): UIO[FiberRef[A]] = ???
```

    - `fork` operation defines how the value of the parent reference from the parent fiber will be modified to create a copy for the new fiber when the child is forked.
    - `join` specifies how the value of the parent fiber reference will be combined with the value of the child fiber reference when the fiber is joined back.

- `FiberRef#locally` sets the `FiberRef` to the specified value, runs the zio effect, and then sets the value of the `FiberRef` back to its original value.
```
trait FiberRef[A] {
    def locally[R, E, A](value: A)(zio: ZIO[R, E, A]): ZIO[R, E, A]
}
```

- The value is guaranteed to be restored to the original value immediately after the zio effect completes execution, regardless of whether it completes successfully, fails, or is interrupted.

- Example use case - can be used to change the logging level for a specific effect without changing it for the rest of the application.

