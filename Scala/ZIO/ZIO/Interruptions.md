# Interruptions

- Interrupting a fiber (`.interrupt`) is an effectful operation that semantically blocks the calling fiber until the interrupted fiber is done or interrupted.
```
trait Fiber[+E, +A] {
    def interrupt: UIO[Exit[E, A]]
}

for {
    fib <- zioa.fork
    _ <- ZIO.sleep(1.second) *> fib.interrupt
    _ <- ZIO.succeed("Interruption successful")
    result <- fib.join
} yield result
```

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