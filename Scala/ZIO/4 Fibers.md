# Fibers
- A Fiber is a description of an effect being executed on some thread. The same fiber can run on multiple JVM threads. ZIO has a thread pool that manages the execution of effects.

> [!TIP]
> When we call `Runtime#unsafe.run`, the ZIO runtime creates a new fiber to execute our program and submit that Fiber for execution to some `Executor`. The Executor will then begin executing the ZIO program one instruction at a time on an underlying OS thread.

> [!TIP]
> Because a ZIO effect is just a “blueprint” for doing something, executing the ZIO program just means executing each instruction of the ZIO program one at a time.
> However, each fiber yields back to the runtime after every `maximumYieldOpCount` instructions so that long computations can take pauses and hand over the Executor to other fibers.

- Creating a fiber is an effectful operation i.e. the fiber will be wrapped in a ZIO -
```
val zio = ZIO.succeed(10)
val fib: ZIO[Any, Nothing, Fiber[Throwable, Int]] = zio.fork
```

> [!TIP]
> ZIO runtime uses work-stealing thread pool.

- `join` method on a `Fiber` will return another effect which will block until the `Fiber` is completed -
```
for {
    fib <- zio.fork
    result <- fib.join
} yield result      // returns 10
```

- Awaiting a fiber -
```
for {
    fib <- zio.fork
    result <- fib.await
} yield result      // returns Succeed(10)
```
- `await` return type is `Exit[E, A]`

> [!NOTE]
> Blocking effects in a fiber leads to descheduling.

- `poll` method is used to peek at the result of the fiber at the current moment without blocking it -
```
for {
    fib <- zio.fork
    result <- fib.poll
} yield result      // returns Some(10)
```
- `poll` return type is `UIO[Option[Exit[E, A]]]` - if the fiber hasn't completed at the point of peeking, it will return `None`.

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

- `orElse` on fibers -
```
for {
    fib1 <- ZIO.fail("Error").fork
    fib2 <- ZIO.succeed(20).fork
    fiber = fib1.orElse(fib2)
    result <- fiber.join
} yield result
```

> [!TIP]
> Semantic blocking - the fibers looks blocked but no JVM thread is actually blocked as they are actively scheduling other fibers for execution.
> Example - `ZIO.sleep(1.second)` will semantically block the fiber and is interruptible, whereas, `ZIO.succeed(Thread.sleep(1000))` is a blocking thread and uninterruptible.

> [!TIP] 
> Cooperative scheduling - ZIO runtime never preempts your fiber, the fibers itself yield control of their thread. 
