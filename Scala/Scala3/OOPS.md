## Object-oriented Programming

- Classes & objects -
```
class Person(name: String, val age: Int) {
    /* fields and methods */
}

val john = new Person("John", 25)
```

- `val age` makes the `age` parameter as class field and therefore, can be accessed from object - `john.age`
- But `name` is only a constructor parameter, not a field, therefore `john.name` will cause compilation error.
- `this` keyword points to current instance of the class.

- Auxillary constructors -
```
class Person(name: String, val age: Int) {
    def this(name: String) = this(name, 0)
}
```

- Auxillary constructors can only call another constructors which eventually call the primary constructor.
- Default arguments -
```
class Person(name: String, age: Int = 0)
```

## Method notations
- **Infix** - 
    - Only available for methods with _one_ argument.
    - Example -
    ```
    class Num(x: Int) {
        infix def isValueGreaterThan(y: Int): Boolean = x > y
    }

    val a = new Num(10)
    a.isValueGreaterThan(5)     // normal method call
    a isValueGreaterThan 5      // "infix" notation - both are identical
    ```
    - Operators in scala are infix methods -
    ```
    2 + 3
    2.+(3)
    ```
    - Infix notation is optional but is recommended. Might be removed in future Scala versions.

- **Prefix** -
    - Only available for methods with _zero_ argument.
    - Example -
    ```
    class Num(x: Int) {
        infix def unary_+ : Int = x + 1     // whitespace between unary_+ and :
    }

    val a = new Num(10)
    ++a             // 11
    a.unary_+      // 11 - "prefix" notation - both are identical
    ```
    - Supported unary operators - `-, +, ~, !`

- **Postfix** -
    - Only available for methods with _zero_ argument.
    - Example -
    ```
    import scala.language.postfixOps        // mandatory when using postfix notation

    class Num(x: Int) {
        infix def isFive: Boolean = x == 5
    }

    val a = new Num(5)
    a.isFive        // true
    a isFive        // true - "postfix" notation - both are identical
    ```
    - Generally discouraged to use postfix notation.

- **apply()** -
    ```
    class Sum(x: Int) {
        def apply(y: Int): Int = x + y
    }

    val a = new Num(5)
    a.apply(10)       // 15
    a(10)             // 15 - both are identical
    ```

## Inheritance

```
class Person(name: String, age: Int) {
    def eat(): Unit = println(s"$name eating")
    def sleep(): Unit = println(s"$name sleeping")
}

class Adult(name: String, age: Int, id: String) extends Person(name, age) {  // must specify super-constructor
    def wakeup(): Unit = println(s"$name wake up")
    def morningRoutine(): Unit = {
        eat()               // call to parent method implementation
        wakeup()
    }
    override def sleep(): Unit = println(s"$name sleeping in 10 mins")      // method overriding
}
```

- Inheritance implies _IS-A_ relationship implying **subtype polymorphism** -
```
class Animal {
    def speak(): String = "animal"
}
class Dog extends Animal {
    override def speak(): String = "dog"
}
val dog: Animal = new Dog       // Dog IS-A animal
dog.speak()                     // dog - most specific method is called at runtime
```

## Access Modifiers

- `public` (by default) class members can be accessed from everywhere.
- `protected` members can be accessed within the class or sub-classes, not the class instances.
- `private` members can only be accessed within the class.

## Preventing Inheritance

- `final` -
    - classes cannot be inherited
    - class members cannot be overriden
- `sealed` -
    - classes cannot be inherited outside the _file_.

- Recommended practise is to not inheritance a class which as no modifier specified. To explicitly mark the class open for inheritance, we use `open` keyword (optional) -
```
open class Animal
```

## Objects

- `object` represents the type and the only instance of that type.
```
object Pi {
    val value = 3.14
    def sum(i: Double): Double = value + sum
}

val instanceOne = Pi
val instanceTwo = Pi
instanceOne == instanceTwo      // true

// fields and method calls
Pi.value                        // 3.14
val sumWithPi = Pi.sum(2.0)     // 5.14
```

- **Companion object** -
    - implies the presence of a `class` and an `object` with same name in same file.
    - In `class`, we define instance-specific members. In `object`, we define class-specific members.
    - Example -
    ```
    class Person(name: String, age: Int)
    object Person {
        val N_EYES: Int = 2
    }
    ```

    - `class` and `object` can access each other's private members.

## Equality

- Equality of reference (`eq`)
    ```
    class X(value: Int)
    object X

    val x1 = new X(5)
    val x2 = new X(5)

    x1 eq x2        // false
    X == x          // true
    ```

- Equality of similarity (`==` or `equals`)
    - By default, points to reference equality (`eq`) but can be overriden. 

## Abstract classes

- Abstract class describes an entity (_IS-A_ relationship).
- Abstract classes can have both abstract and non-abstract members.
```
abstract class Animal {
    val kind: String
    def speak(): Unit
    def meal: String = "anything"
}

class Dog extends Animal {
    override val kind: String = "domestic"
    override def speak(): Unit = println("woof")
    override val meal: String = "cookie"
}
```

- Abstract classes can have constructor arguments.
- Can extend only one class.

> [!NOTE]
> `meal` method is overriden as a `val` and it is only possible if the method (`meal`) does not have any paranthesis or arguments (i.e. accessor methods). 

## Traits

- Traits describes a behavior (_HAS-A_ relationship).
- Can have both abstract and non-abstract members.
```
trait Carnivore {
    def eat(): Unit
}

class Lion extends Carnivore {
    override def eat(): Unit = println("Lion eating")
}
```

- From Scala 3, traits can also have constructor arguments.
- Can extend multiple traits.
```
trait A
trait B
trait C

class X extends A with B with C
```

## Scala classes

- `Any` has two subtypes - `AnyRef` and `AnyVal`. `AnyRef` represents object references and `AnyVal` represents value classes like - `Int`, `Boolean`, `Char` etc.
- Compiler automatically extends `AnyRef` to our classes.
```
class Animal
class Animal extends AnyRef     // compiled code
```

- `scala.Null` contains only the `null` reference and is the lowest sub-type of `AnyRef`.
- `scala.Nothing` is the lowest sub-type of _any_ type and has _no_ instances. Expressions that return `Nothing` are throwing an exception, `???`

## Anonymous classes

```
abstract class Animal {
    def speak(): Unit
}

val dog = new Animal {
    def speak(): Unit = println("woof")
}
```

- Compiled code -
```
class AnonClass$1 extends Animal {
    def speak(): Unit = println("woof")
}

val dog = new AnonClass$1
```

- Class, abstract class, trait - all of them can be instantiated like this.

## Case class

```
case class Person(name: String, age: Int)
val john = Person("John", 20)
```

- Properties -
    - Constructor arguments are class members - `john.name`
    - Overriden `toString` prints arguments - `Person(John,20)`
    - `equals` is based on argument value matching
    - `hashCode` is implemented based on the arguments
    - extends `Serializable`
    - has companion object containing -
        - `apply` method with sam arguments as constructor's - `Person.apply("John", 25)`
        - `unapply` method - for pattern matching on fields
    - `copy` method to create new instance with updated values - `john.copy(age = 25)`

- Case class must have zero or more arguments i.e. `case class Animal` will not compile. For this purpose, we either define it with zero-parameter - `case class Animal()` or use _case objects_ - `case object Animal`

## Enums

```
enum Priority {
    case LOW, MID, HIGH
    def isHighest: Boolean = this == HIGH
}

val status: Status = Status.MID
```

- Ordinal `ordinal()` represents the index at which the enum value is defined - `status.ordinal()` will return 1.

- Enum can also take constructor arguments -
```
enum Priority(value: Int) {
    case LOW extends Priority(3)
    case MID extends Priority(2)
    case HIGH extends Priority(1)
}
```

- Can have companion object -
```
object Priority {
    def fromValue(value: Int): Priority = ???
}
```

- Accessing all values of enum - `Priority.values`
- Creating an enum instance from string - `Priority.valueOf("MID")` - returns `Priority.MID`

