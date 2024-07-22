# Fibers
- A Fiber is a description of an effect being executed on some other thread. The same fiber can run on multiple JVM threads.
- ZIO has a thread pool that manages the execution of effects.
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
> Semantic blocking - the fibers looks blocked but no JVM thread is actually blocked as they are activey scheduling other fibers for execution.

> [!TIP] 
> Cooperative scheduling - ZIO runtime never preempts your fiber, the fibers itself yield control of their thread. 
