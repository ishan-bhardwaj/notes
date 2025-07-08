## Promises

- Promises block current thread until some other thread completes the ZIO.
- Creating a promise - 
```
val aPromise: ZIO[Any, Nothing, Promise[Throwable, Int]] = Promise.make[Throwable, Int]
```

- `Promise#await` blocks the promise until the promise has a value -
```
val reader: ZIO[Any, Throwable, Int] = aPromise.flatMap(promise => promise.await)
```

- Promises can be  - `promise.succeed(Int)`, `promise.failed(Throwable)`, `promise.completed(zio.IO[Throwable, Int])`

- Promises use-cases -
  - Purely functional block on a fiber until you get a signal from another fiber.
  - Waiting on a value may not yet be available, without thread starvation - avoids busy waiting.
  - Inter-fiber communication.

