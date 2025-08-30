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
    - `map`, `flatMap`, `filter` & for-comprehension
    - `Option#orElse` - chain multiple options.


## Try

