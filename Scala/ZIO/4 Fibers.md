# Fibers
- A Fiber is a description of a computation that runs on one of the threads managed by ZIO runtime.
- Not possible to create fibers manually.
- `fork` method returns another effect whose return type `Fiber` -
```
val zio = ZIO.succeed(10)
val fib: ZIO[Any, Nothing, Fiber[Throwable, Int]] = zio.fork
```

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

> [!NOTE]
> `await` return type is `Exit[E, A]`

- `poll` method is used to peek at the result of the fiber at the current moment without blocking it -
```
for {
    fib <- zio.fork
    result <- fib.poll
} yield result      // returns Some(10)
```

> [!NOTE] 
> `poll` return type is `UIO[Option[Exit[E, A]]]` - if the fiber hasn't completed at the point of peeking, it will return `None`.

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


