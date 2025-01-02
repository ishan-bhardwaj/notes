# Parallelism
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