## Async

- `ZIO.async` allows you to wrap asynchronous computations that use callbacks (e.g., legacy APIs or Java SDKs) into a ZIO effect.
- Eg - Typical Java-style async function -
```
def asyncAdd(a: Int, b: Int)(callback: Int => Unit): Unit = {
  new Thread { () => 
    callback(a + b)    // return result through callback
  }.start()
}
```

- To convert this into ZIO effect -
```
def zioAdd(a: Int, b: Int): ZIO[Any, Nothing, Int] =
  ZIO.async { callback =>
    asyncAdd(a, b) { result =>
      callback(ZIO.succeed(result))
    }
  }
```

- Use-cases -

  - Lifting a computation running on some external thread to a ZIO -
  ```
  def external2ZIO[A](computation: () => A)(executor: ExecutorService): Task[A] =
    ZIO.async[Any, Throwable, A] { cb =>
      executor.execute { () =>
        try {
          val result = computation()
          cb(ZIO.succeed(result))
        } catch {
          case e: Throwable => ZIO.fail(e)
        }
      }
    }
  ```

  - Lifting Future to ZIO -
  ```
  def future2ZIO[A](future: => Future[A])(using ec: ExecutionContext): Task[A] =
    ZIO.async[Any, Throwable, A] { cb =>
      future.onComplete {
        case Success(value) => cb(ZIO.succeed(value))
        case Failure(ex) => cb(ZIO.fail(ex))
      }
    }
  ```

> [!NOTE]
> Runnable & Future will run on some other thread pool, whereas ZIO effect will run on its native threadpool -

  - Never-ending ZIO -
  ```
  def neverEndingZIO[A]: UIO[A] =
    ZIO.async(_ => ())

  // Same implementation for -
  ZIO.never  
  ```

> [!TIP]
> When invoking `ZIO.async`, the fiber that evaluates it is semantically blocked until we invoke the `callback`. Therefore, we don't invoke the `callback`, the ZIO will be suspended indefinitely. For eg - "completed" will never be executed in below code -

  ```
  object Main extends ZIOAppDefault {
    override def run = ZIO.logInfo("Computing") *> neverEndingZIO[Int] *> ZIO.logInfo("completed")
  }
  ```