## Refs

- Refs are pure functional atomic references.
- Creating Refs -
```
val atomicMOL: ZIO[Any, Nothing, Ref[Int]] = Ref.make(42)
```

- Obtaining value (thread-safe) - `atomicMOL.flatMap(ref => ref.get)` - returns `UIO[Int]`.
- Setting value (thread-safe) - `atomicMOL.flatMap(ref => ref.set(100))` - returns `UIO[Unit]`.
- Get and set in ONE atomic operation - `atomicMOL.flatMap(ref => ref.getAndSet(500))` - returns `UIO[Int]`.
- Updating value using a function in ONE atomic operation - `atomicMOL.flatMap(ref => ref.update(_ + 1))` - returns `UIO[Unit]`.
- Updating and obtaining the value in ONE atomic operation - 
  - `Ref#updateAndGet` - updates ref and returns the updated value.
  - `Ref#getAndUpdate` - updates ref and returns the previous value.

- Modify - returns a `Tuple2` containing the value of some type T which will be surfaced out in the final effect and the next value of ref -
```
val modifiedMOL: UIO[String] = atomicMOL.modify(value => (s"Current value is: $value", value * 100))
```

> [!WARNING]
> The example below will always print `0` because - every time you call `incrementByOneEverySecond()`, it creates a new `Ref(0)` and therefore, the `counter` is reset to `0` each time.
>
> ```
> val counterRef: UIO[Ref[Int]] = Ref.make(0)
> 
> def incrementByOneEverySecond(): UIO[Unit] =
>   for {
>     counter <- counterRef
>     _       <- ZIO.sleep(1.second)
>     c       <- counter.getAndUpdate(_ + 1)
>     _       <- ZIO.logInfo(s"Incremented: $c")
>     _       <- incrementByOneEverySecond()
>   } yield ()
> ```
> 
> To solve this issue, we should pass the `counterRef` as input parameter to the method.

> [!NOTE]
> If `Ref#update` and `Ref#modify` functions are blocked then may modify the ref again once it's free.
