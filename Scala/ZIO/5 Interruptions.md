# Interruptions
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

## Manual Interruption
```
val intZio = ZIO.succeed("computing...") *> ZIO.interrupt *> ZIO.succeed(42)
```
- The ZIO will be interrupted after printing `computing...`.

## Finalizer
```
intZio.onInterrupt(ZIO.succeed("Interrupted!"))
```

- Will print `Interrupted` in case the `intZio` is interrupted.

## Uninterruptible

