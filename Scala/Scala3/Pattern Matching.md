## Pattern Matching 

```
val value: Int = ???

value match {
    case 1 => "first"
    case 2 => "second"
    case 3 => "third"
    case _ => "unknown: $value"
}
```

> [!WARNING]
> If there is _default_ case in pattern matching or the cases are not exhausitive then it will throw `scala.MatchError` if none of the cases match.

- Decompose values in case classes -

```
case class Person(name: String, age: Int)
val bob = Person("Bob", 42)

val greeting = bob match {
    case Person(n, a) => s"Hello, I am $b and I am $a years old!"
    case x => s"Unknown: $x"
}
```

- Guards in pattern matching -
```
case Person(_, a) if a < 18 => "Teen"
```

> [!TIP]
> Patterns are matching in the order they're defined.

- Sealed hieararchies -

```
sealed trait Animal

case class Dog(breed: String) extends Animal
case class Cat(eyeColor: String) extends Animal

val anAnimal: Animal = Dog("Terra Nova")

val pm_v1 = anAnimal match {
    case Dog(breed) => "A dog!"
}

val pm_v2 = anAnimal match {
    case Dog(breed) => "A dog!"
    case Cat(color) => "A cat!"
}
```

- `pm_v1` won't compile and returns error - `match may not be exhaustive`.

- Nested structures -

```
val nestedTuple = (1, (2, 3))

nestedTuple match {
    case (_, (2, v)) => s"Found :$v"
}
```

- Option -

```
val anOption = Option(2)

anOption match {
    case Some(v) => s"Found :$v"
    case None => "No value found!"
}
```

- List patterns -

```
val list = List(1, 2, 3, 4)

list match {
    case List(1, _, _, _, _) => "List starting with 1"
    case List(1, _*) => "List starting with 1"              // same as above
    case List(1, 2, _) :+ 4 => "List starting with 1, 2 and ending with 4" 
    case 1 :: tail => "Starting with 1 and rest is $tail"
}
```

- Type specifiers -

```
val unknown: Any = 42

unknown match {
    case anInt: Int => s"Found Int: $anInt"
    case aString: String => s"Found String: $aString"
}
```

- Name binding -

```
list match {
    case List(1, rest @ List(_, tail)) => "List starting from 2 bounded as rest: $rest"
}
```

- Chained patterns -

```
anInt match {
    case 1 | 2 => "Either 1 or 2"
}
```

> [!WARNING]
> Pattern match deconstructions and type checks are evaluated at runtime using reflection. And because all the generic types are erased at runtime, the following will print `list of strings` -
> ```
> val nums: List[Int] = List(1, 2, 3, 4)
> nums match {
>   case strings: List[String] => "list of strings"
>   case ints: List[Int] => "list of ints"
> }
> ```

> [!TIP]
> Generic types are erased in JVM at runtime because generics were added in Java 5 and it has to be fully-backward compatible with earlier versions.


### Based on pattern matching

- `catch` in try-catch -

```
try { ... }
catch {
  case e: RuntimeException => ???   
}
```

- Generators in for-comprehensions for deconstructions -

```
val tuples = List((1, 2), (3, 4))
for {
  (first, second) <- tuples if first < 3
} yield second * 10
```

- Deconstructing structures -

    - On tuples -
    ```
    val aTuple = (1, 2, 3)
    val (a, b, c) = aTuple
    ```

    - On lists -
    ```
    val list = List(1, 2, 3, 4)
    val head :: tail = list
    ```



