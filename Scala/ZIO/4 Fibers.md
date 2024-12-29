# Fibers

- A Fiber is a description of an effect being executed on some thread. The same fiber can run on multiple JVM threads. ZIO has a thread pool that manages the execution of effects.
- When we call `Runtime#unsafe.run`, the ZIO runtime creates a new fiber to execute our program and submit that Fiber for execution to some `Executor`. The Executor will then begin executing the ZIO program one instruction at a time on an underlying OS thread.
- Because a ZIO effect is just a “blueprint” for doing something, executing the ZIO program just means executing each instruction of the ZIO program one at a time. However, each fiber yields back to the runtime after every `maximumYieldOpCount` instructions so that long computations can take pauses and hand over the Executor to other fibers.

> [!TIP]
> ZIO runtime uses work-stealing thread pool.

> [!TIP]
> Semantic blocking - the fibers looks blocked but no JVM thread is actually blocked as they are actively scheduling other fibers for execution, for eg - `ZIO.sleep(1.second)` will semantically block the fiber and is interruptible, whereas, `ZIO.succeed(Thread.sleep(1000))` is a blocking thread and uninterruptible.

> [!TIP] 
> Cooperative scheduling - ZIO runtime never preempts your fiber, the fibers itself yield control of their thread.

### `fork`

- Forking creates a new fiber that executes the effect being forked concurrently with the current fiber -
```
trait ZIO[-R, +E, +A] {
    def fork: URIO[R, Fiber[E, A]]
}

// Example
val zio: Task[Int] = ZIO.attempt(10)
val fib: UIO[Fiber[Throwable, Int]] = zio.fork
```

> [!TIP]
> As soon as the fiber is created using `ZIO#fork`, it starts executing.

### `join`

- `join` method on a `Fiber` will return another effect which will block until the `Fiber` is completed -
```
for {
    fib <- zio.fork
    result <- fib.join
} yield result      // returns 10
```
- When we join a fiber, execution in the current fiber can’t continue until the joined fiber completes execution. But no actual thread will be blocked waiting for that to happen.
- Instead, internally, the ZIO runtime registers a callback to be invoked when the forked fiber completes execution, and then the current fiber suspends execution.
- That way, the Executor can go on to execute other fibers and doesn’t waste any resources waiting for the result to be available.
- Joining a fiber translates the result of that fiber back to the current fiber, so joining a fiber
that has failed will result in a failure.

### `await`
- If we want to wait for the fiber but be able to handle its result, whether it is a success
or a failure, we can use `await`.
```
trait Fiber[+E, +A] {
    def await: UIO[Exit[E, A]]
}

// Example
for {
    fib <- zio.fork
    result <- fib.await
} yield result      // returns Succeed(10)
```

- Using `Exit#foldZIO` we can handle the result of the fiber -
```
for {
    fib <- zio.fork
    exit <- fib.await
    _ <- exit.foldZIO(
        e => ZIO.debug("The fiber has failed with: " + e),
        s => ZIO.debug("The fiber has completed with: " + s)
    )
} yield ()
```

### `poll`

- `poll` is used to peek at the result of the fiber at the current instant without blocking it -
```
trait Fiber[+E, +A] {
    def poll: UIO[Option[Exit[E, A]]]
}

// Example
for {
    fib <- zio.fork
    result <- fib.poll
} yield result      // returns Some(10)
```
- This will return `Some` with an `Exit` value if the fiber has completed execution, or `None` otherwise.

### `zip`

- Zipping fibers to return a tuple -
```
for {
    fib1 <- ZIO.succeed(10).fork
    fib2 <- ZIO.succeed(20).fork
    fiber = fib1.zip(fib2)
    tuple <- fiber.join
} yield tuple
```

> [!NOTE]
> Zipping fibers doesn't create another effect. It will use thread from one of the fibers.

### `orElse`

- `orElse` on fibers -
```
for {
    fib1 <- ZIO.fail("Error").fork
    fib2 <- ZIO.succeed(20).fork
    fiber = fib1.orElse(fib2)
    result <- fiber.join
} yield result
```