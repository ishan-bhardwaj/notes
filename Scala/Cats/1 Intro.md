# CATS
- Contravariance can use the superclass instances if nothing is available strictly for that type.
- Covariant type classes will always use the more specific type class instance for that type, but may confuse the compiler if the general type class instance is also present.
- Because we cannot have covariance and contravariance at the same time i.e. -
    - ability to use the superclass typeclass instance
    - preference for more specific typeclass instance
Hence, Cats uses invariance -
```
Option(2) === Option.empty[Int] // compiles fine
Some(2) === None    // doesn't compile
```

### `Eq`

- `Eq` type class applies the type checking on equality operation -
```
import cats.Eq
import cats.instances.int._

val aTypeSafeComparison: Boolean = Eq[Int].eqv(2, 5)
```
- Using extension methods -
```
import cats.syntax.eq._

val anotherComparison = 2 === 5
val notEqualComparison = 2 =!= 5
```
> [!NOTE]
> The `===` and `=!=` operators are type-safe, hence `2 === "Hello"` will result in compilation error.

- Extending the type class operations to composite types like `List`, `Option` etc -
```
import cats.instances.list._    // we bring Eq[List[Int]] in scope

val aListComparison = List(2) === List(3)
```

- Create a type class instance for custom types -
```
case class Car(model: String, price: Double)

implicit val carEq: Eq[Car] = Eq.instance[Car] { (car1, car2) => car1.price == car2.price }
```

- Import all -
```
import cats._
import cats.implicits._
```

