## Option

- Wrapper for a value that might be present or not.
```
sealed abstract class Option[+A]
case class Some[+A](x: A) extends Option[A]
case object None extends Option[Nothing]
```

- Creating an option -
```
val anOption_v1: Option[Int] = Option(42)
val emptyOption_v1: Option[Int] = Option.empty
```

- `Option` has two subtypes - `Some` & `None` -
```
val anOption_v2: Option[Int] = Some(42)
val emptyOption_v2: Option[Int] = None
```

- Operations -
    - `Option#isEmpty` - checks whether option is empty or not.
    - `Option#getOrElse` - get value of option, if present; otherwise return some default value.
    - `map`, `flatMap`, `filter` & for-comprehension.
    - `Option#orElse` - chain multiple options - if first returns `None` then return second one.


## Try

- `Try` is a wrapper for a computation that might fail.

```
import scala.util.Try

val aTry: Try[Int] = Try(42)
val aFailedTry: Try[Int] = Try(throw new RuntimeException)
```

- `Try#apply` method accepts input argument as call-by-name.
- `Try` has two subtypes - `Success` & `Failure` -
```
val aTry: Try[Int] = Success(42)
val aFailedTry: Try[Int] = Failure(throw new RuntimeException)
```

- Operations -
    - `Try#isSuccess` - check if success.
    - `Try#isFailure` - check if failure.
    - `map`, `flatMap`, `filter` & for-comprehension.
    - `Try#orElse` - chain multiple Try's - if first is failure then return second one.
    - `Try#fold` - specify success operation on both success and failure.

> [!TIP]
> In `aTry.flatMap(anotherTry)`, if `aTry` returns a `Failure`, then that exception will be returned regardless of `anotherTry`.

> [!TIP]
> In `aTry.filter(_ % 2 == 0)`, if `aTry` has value `Success(1)` then it will throw `NoSuchElementException`.

