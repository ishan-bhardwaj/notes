# Fibers
- A Fiber is a description of an effect being executed on some other thread.
- ZIO has a thread pool that manages the execution of effects.
- Creating a fiber is an effectful operation i.e. the fiber will be wrapped in a ZIO -
```
val zio = ZIO.succeed(10)
val fib: ZIO[Any, Nothing, Fiber[Throwable, Int]] = zio.fork
```

> [!TIP]
> ZIO runtime uses work-stealing thread pool.

> [!TIP]
> The same fiber can run on multiple JVM threads.

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

> [!NOTE]
> Blocking effects in a fiber leads to descheduling.

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

> [!TIP]
> Semantic blocking - the fibers looks blocked but no JVM thread is actually blocked as they are activey scheduling other fibers for execution.

> [!TIP] 
> Cooperative scheduling - ZIO runtime never preempts your fiber, the fibers itself yield control of their thread. 

## Interruptions
- Interrupting a fiber (`.interrupt`) is an effectful operation that semantically blocks the calling fiber until the interrupted fiber is done or interrupted.
```
for {
    fib <- zioa.fork
    _ <- ZIO.sleep(1.second) *> fib.interrupt
    _ <- ZIO.succeed("Interruption successful")
    result <- fib.join
} yield result
```
- We can define some computation on interruption and completion of the effects - 
```
ZIO.succeed(10).onInterruption("Interrupted!")
```
- We can also `fork` the `interrupt` call so that it won't block the calling fiber -
`ZIO.sleep(1.second) *> fib.interruptFork`
or
`(ZIO.sleep(1.second) *> fib.interrupt).fork`
Note that we don't need to `join` this forked fiber causing a leaked fiber, but its lifecycle will be very short therefore leaking a fiber is fine as it will be cleaned by garbage collector.

> [!WARNING]
> Child fibers will be automatically interrupted if the parent fiber is completed.
> ```
> val zioWithTime = 
>   ZIO.succeed("starting computation") *>
>       ZIO.sleep(2.seconds) *>
>       ZIO.succeed(10)
>
> val parentEffect =
>   ZIO.succeed("spawning fiber") *>
>       zioWithTime.fork *>       // child fiber
>       ZIO.sleep(1.second) *>
>       ZIO.succeed("parent successful")
> ```
> Here, the child fiber is taking 2 seconds but the parent only takes 1 second to finish and hence, the child fiber will be interruted after 1 second automatically.

> [!TIP] 
> We can change the parent of child fiber to the `main` fiber using `zioWithTime.forDaemon`

## Racing
- `race` executes two effects independently on their own fibers and the fiber that completes first will complete the whole effect, whereas the other one will get interrupted automatically.
```
val slowEffect = (ZIO.sleep(2.seconds) *> ZIO.succeed("slow"))
                    .onInterrupt(ZIO.succeed("[Slow] interrupted"))
val faseEffect = (ZIO.sleep(1.second) *> ZIO.succeed("false"))
                    .onInterrupt(ZIO.succeed("[Fast] interrupted"))

val aRace = slowEffect.race(fastEffect)
aRace.fork *> ZIO.sleep(3.seconds)
```

## Implementing `timeout` function
- Timeout function v1 -
    - If ZIO is successful before timeout => a successful effect
    - If ZIO fails before timeout => a failed effect
    - If ZIO takes longer than timeout => interrupt the effect
```
def timeout_v1[R, E, A](zio: ZIO[R, E, A], time: Duration): ZIO[R, E, A] =
    for {
        fib <- zio.fork
        _ <- (ZIO.sleep(timeout) *> fib.interrupt).fork
        result <- fib.join
    } yield result
```

- Timeout function v2 -
    - If ZIO is successful before timeout => a successful effect with `Some(a)`
    - If ZIO fails before timeout => a failed effect
    - If ZIO takes longer than timeout => interrupt the effect, return a successful effect with `None`

```
def timeout_v2[R, E, A](zio: ZIO[R, E, A], time: Duration): ZIO[R, E, Option[A]] =
    timeout_v1(zio, time).foldCauseZIO(
        cause => if (cause.isInterrupted) ZIO.succeed(None) else ZIO.failCause(cause)
        value => ZIO.succeed(Some(value))
    )
```
