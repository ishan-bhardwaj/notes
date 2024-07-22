# Resources
- Finalizers -
```
val anAttempt = ZIO.attempt(unsafeMethod())
val attemptWithFinalizer = anAttempt.ensuring(ZIO.succeed("finalizer!"))
```
- `finalizer!` will be printed out even if the ZIO fails.
- We can chain multiple finalizers to the same effect.
- Multiple finalizers to handle different kind of events - `onInterrupt`, `onError`, `onDone`, `onExit`

## `AcquireRelease`
- Defines the creation and closing part of the resource, so that we don't have to take care of closing the resource in our business logic -
```
val cleanConnection = ZIO.acquireRelease(Connection.create("https://google.com"))(_.close())
```
- All the finalizers are guranteed to run.

> [!TIP]
> Acquiring cannot be interrupted.

- `acquireRelease`- connection when used will have the return type - `ZIO[Any with Scope, Nothing, Connection]` which has dependency called `Scope` that defines the lifetime of a resource. When we invoke the effect, the `Scope` is automatically provided by the ZIO runtime.
- If you want to eliminate the `Scope` dependency - 
```
val fetchWithScopedResource: ZIO[Any, Nothing, Unit] = ZIO.scoped(fetchWithResources)
```
- Now, the resources we're acquiring will have finite lifespan.
- If we combine multiple effects with `Scope` dependency, the same `Scope` will be used to acquire all the resources in the effects.

## `AcquireReleaseWith`
- Defines the creation, closing and usage of the resource in single method call -
```
val cleanConnection_v2 = 
    ZIO.acquireReleaseWith(
            Connection.create("https://google.com")     // acquire
        )(
            _.close()       // release
        )(
            conn => conn.open *> ZIO.sleep(2.seconds)       // use
        )
```
- `acquireReleaseWith` doesn't require `Scope` dependency.

> [!TIP]
> `acquireRelease` is a better choice for nested resources.
