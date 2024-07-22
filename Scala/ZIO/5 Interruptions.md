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

## Parallelism
- General parallelism rule -
    - If all the effects succeed => success
    - If any effect fails => everything else is interrupted, error of the failed effect is surfaced out
    - If any effect is interrupted => everything else is interrupted, error of the entire interruption is surfaced out
    - If the entire thing is interrupted => all effects are interrupted
- Zipping two fibers using `zip` is a sequential effect. To run them in parallel, we can use `zipPar` method -
`fib1.zipPar(fib2)`
- `zipWithPar` allows you to provide a combinator function.
- `collectAllPar` - sequence of effects will start to run in different threads and all of them will be collected in a Vector in the same sequence as they were created -
```
val effects: Seq[ZIO[Any, Nothing, Int]] = (1 to 10).map(i => ZIO.succeed(i))
val collectedValues: ZIO[Any, Nothing, Seq[Int]] = ZIO.collectAllPar(effects)
```
- `foreachPar` -
```
val printParallel = ZIO.foreachPar((1 to 10).toList)(i => ZIO.succeed(println(i)))
```
- `reduceAllPar` -
```
val sumPar = ZIO.reduceAllPar(ZIO.succeed(0), effects)(_ + _)
```
- `mergeAllPair` -
```
val sumPar_v2 = ZIO.mergeAllPair(effects)(0)(_ + _)
```