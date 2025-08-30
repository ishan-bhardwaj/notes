## Promises

- A `Promise[E, A]` is a write-once, read-many synchronization construct that can be completed exactly once—either with a value of type `A` (success) or with an error of type `E` (failure). Once completed, it can be observed (awaited) multiple times by different fibers.
- Promise begins its lifetime with empty state and can be completed exactly once. Once it’s completed (success or failure), any further attempts to complete it are ignored.
- Creating a promise - 
```
val aPromise: ZIO[Any, Nothing, Promise[Throwable, Int]] = Promise.make[Throwable, Int]
```

- A Promise can be completed in three ways -
  - `promise.succeed(value: A)`
  - `promise.fail(error: E)`
  - `promise.complete(effect: IO[E, A])`

- `Promise#await` semantically blocks the fiber until some other fiber completes the promise, for eg - this will produce output `Customer received: Margherita Pizza` after 2 seconds -
```
def customer(pizzaPromise: Promise[Nothing, String]): UIO[Unit] =
  pizzaPromise.await.flatMap(pizza => Console.printLine(s"Customer received: $pizza"))

def deliveryGuy(pizzaPromise: Promise[Nothing, String]): UIO[Unit] =
  ZIO.sleep(2.seconds) *> pizzaPromise.succeed("Margherita Pizza")

def run = for {
  promise <- Promise.make[Nothing, String]
  _       <- customer(promise).fork     // Customer waits
  _       <- deliveryGuy(promise)       // Delivery guy delivers
} yield ()
```

- Advanced completion methods -
  - `die(t: Throwable): UIO[Boolean]` -
    - Used to signal fatal errors (defects).
    - The promise is completed as died with the specified Throwable.
  - `done(e: Exit[E, A]): UIO[Boolean]` -
    - Completes the promise with a full `Exit` value, which can represent -
      - Success with a value `A`
      - Failure with an error `E`
      - Defect (unchecked failure)
      - Interrupt
  - `failCause(c: Cause[E]): UIO[Boolean]` -
    - Completes the promise with a failure cause, which can be:
      - One or more errors (E)
      - Defects
      - Interruptions

> [!NOTE]
> All these methods return a `UIO[Boolean]` indicating whether the completion was successful (`true`) or if the promise was already completed (`false`).

