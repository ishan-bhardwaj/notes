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
> As soon as the fiber is created using `ZIO#fork`, it starts executing. `fork` does not even wait for the forked fiber to begin execution before returning to the main thread.

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
- Joining a fiber translates the result of that fiber back to the current fiber, so joining a fiber that has failed will result in a failure.

### `await`
- If we want to wait for the fiber but be able to handle its result, whether it is a success or a failure, we can use `await`.
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

## Fiber Supervision

- Rules of fiber supervision model -
    - Every fiber has a scope.
    - Every fiber is forked in a scope.
    - Fibers are forked in the scope of the current fiber unless otherwise specified.
    - The scope of a fiber is closed when the fiber terminates, either through success, failure, or interruption.
    - When a scope is closed, all fibers forked in that scope are interrupted.

- Child fibers will be automatically interrupted if the parent fiber is completed.
```
val zioWithTime = 
    ZIO.succeed("starting computation") *>
        ZIO.sleep(2.seconds) *>
        ZIO.succeed(10)

val parentEffect =
    ZIO.succeed("spawning fiber") *>
        zioWithTime.fork *>       // child fiber
        ZIO.sleep(1.second) *>
        ZIO.succeed("parent successful")
```
Here, the child fiber is taking 2 seconds but the parent only takes 1 second to finish and hence, the child fiber will be interruted after 1 second automatically.

- If you do need to create a fiber that outlives its parent, you can fork a fiber on the global scope using `forkDaemon`.
```
trait ZIO[-R, +E, +A] {
    def forkDaemon: URIO[R, Fiber[E, A]]
}
```

> [!TIP]
> Good rule of thumb is to always `Fiber#join` or `Fiber#interrupt` any fiber you fork.

## Locking Effects

- To run an effect on specific executor use `ZIO#onExecutor` -
```
import scala.concurrent.ExecutionContext

trait ZIO[-R, +E, +A] {
    def onExecutor(executor: => Executor): ZIO[R, E, A]
}
```

- If the effect forks other fibers, those fibers will also be locked to the specified Executor unless otherwise specified.
- Rules when using `onExecutor` -
    - When an effect is locked to an Executor, all parts of that effect will be locked to that Executor.
    - Inner scopes take precedence over outer scopes. Eg -
    ```
    lazy val effect2 = for {
        _ <- doSomething.onExecutor(executor2).fork
        _ <- doSomethingElse
    } yield ()

    lazy val result2 = effect2.onExecutor(executor1)
    ```
    Here, `doSomething` is guaranteed to be executed on `executor2` and `doSomethingElse` is guaranteed to be executed on `executor1`.

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

## Blocking Effects
- ZIO blocking tasks should be delegated to the dedicated blocking thread pool because if we run them on normal threads, they can block the thread entirely causing thread starvation.
```
val aBlockingZIO = ZIO.attemptBlocking {
    println(s"[${Thread.currentThread().getName}] runnning a long computation")
    Thread.sleep(10000)
    25
}
```

> [!TIP]
> Blocking code using `attemptBlocking` cannot be interrupted. To apply interruption, we need to use `attemptBlockingInterrupt` method. However, this is based on `Thread.interrupt` java native method call which may result in `InterruptException` and is a heavy operation.
> So, the best way to implement interruption is by setting a flag/switch that we can turn off from inside of the computation -
> ```
> def interruptibleBlockingEffect(canceledFlag: AtomicBoolean): Task[Unit] 
>   ZIO.attemptBlockingCancelable {
>       (1 to 100000).foreach {
>           if (!canceledFlag.get()) { // do computation }
>       }
>   } (ZIO.succeed(canceledFlag.set(true)))
> ```
> Now, if the `canceledFlag` is set to `true` by some other thread then the entire loop will be interrupted and it will not execute anymore.
> When any thread receives an interruption signal, the `ZIO.succeed(canceledFlag.set(true))` line will execute, canceling the entire effect.

## Yielding
- `ZIO.yieldNow` allows the ZIO runtime to take decision of yielding the current thread and start executing the fibers on different threads. But ZIO runtime is smart enough to execute effects on same thread before yielding control.
```
(1 to 10000)
    .map(i => ZIO.succeed(i))
    .reduce((x, y) => println(x) *> ZIO.yieldNow *> println(y))
```
- `ZIO.sleep` is implemented using `ZIO.yieldNow`.