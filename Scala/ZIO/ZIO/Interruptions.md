# Interruptions

- Interrupting a fiber (`.interrupt`) is an effectful operation that semantically blocks the calling fiber until the interrupted fiber is done or interrupted.
```
trait Fiber[+E, +A] {
    def interrupt: UIO[Exit[E, A]]
}
```

- Example - The ZIO will be interrupted after printing `computing...` -
```
ZIO.succeed("computing...") *> ZIO.interrupt *> ZIO.succeed(42)
```

- Some other ways to interrupt a fiber includes - `ZIO.race`, `ZIO.zipPar`, `ZIO.collectAllPar` or outliving parent fiber.

- If the fiber has already completed execution by the time it is interrupted, the returned value will be the result of the fiber. Otherwise, it will be a failure with `Cause.Interrupt`.

- We can define some computation on interruption using finalizer (`onInterruption`) - 
```
ZIO.succeed(10).onInterruption("Interrupted!")
```

- We can also `fork` the `interrupt` call so that it won't block the calling fiber -
`ZIO.sleep(1.second) *> fib.interruptFork`
or
`(ZIO.sleep(1.second) *> fib.interrupt).fork`

> [!NOTE]
> We don't need to `join` this forked fiber causing a leaked fiber, but its lifecycle will be very short therefore leaking a fiber is fine as it will be cleaned by garbage collector.

## Finalizer

```
intZio.onInterrupt(ZIO.succeed("Interrupted!"))
```

- Will print `Interrupted` in case the `intZio` is interrupted.

## Uninterruptible

- `ZIO.uninterruptible` ignores any interruption signals on the specified ZIO -
```
ZIO.uninterruptible(zioA)
```
or
```
zioA.uninterruptible
```

> [!NOTE]
> With `ZIO.uninterruptible`, the interruption signals will be ignored and the effect will complete successfully, but there will still be interruption exception after the execution of fiber because the fiber was interrupted.

- Interruptibility is regional -
    - All the ZIOs are uninterruptible -
    ```
    (zio1 *> zio2 *> zio3).uninterruptible
    ```

    - Inner scopes override outer sccope i.e. `zio2` is interruptible in both cases below -
    ```
    (zio1 *> zio2.interruptible *> zio3).uninterruptible
    ```
    or
    ```
    (zio1 *> zio2.interruptible *> zio3).uninterruptible
    ```

- `uninterruptibleMask` can specify which part of the effects can be interruptible. If nothing is specified, all the effects are considered as uninterruptible by default.

- `zioB` & `zioD` are interruptible, whereas `zioA` & `zioC` are uninterruptible
```
ZIO.uninterruptibleMask { restore =>
    zioA *> restore(zioB) *> zioC *> restore(zioD)
}
```

> [!TIP]
> Uniterruptible calls are masks which suppress cancellation. Restored open "gaps" in the uninterruptible region. If you wrap an entire structure with another `ZIO.uninterruptible` / `ZIO.uninterruptibleMask`, you'll cover those gaps too. Example -
```
object Test extends ZIOAppDefault {
    
    val zioA = ZIO.interrupt *> ZIO.succeed(42)

    val zioB = ZIO.uninterruptible(zioA)

    val zioC = ZIO.uninterruptible(zioB)

    def run = zioB          // Will be interrupted

    def run = zioC          // Will not be interrupted

}
```

