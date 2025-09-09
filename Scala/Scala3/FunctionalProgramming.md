## Functions

- In functional programming, functions are considered as _first-class citizens_, but the JVM only understands classes and objects. 
- Therefore, a `Function` is defined as a `trait` in Scala and all functions are instances of `FunctionX` trait where `X` is the number of arguments the function can accept and can range from `0` to `22`. We can define them as **Anonymous functions** -
```
val adder_v1 = new Function1[Int, Int, Int] {
    override def apply(v1: Int, v2: Int): Int = v1 + v2
}

val sum: Int = adder_v1(2, 3)       // same as adder.apply(2, 3)
```

- **Lambdas** are anonymous functions instances - 
```
val adder_v2: (Int, Int) => Int = (v1, v2) => v1 + v2

// or - if return type is mentioned, the types of "v1" & "v2" are inferred by the compiler
val adder_v3 = (v1: Int, v2: Int) => v1 + v2

// or
val adder_v2: (Int, Int) => Int = _ + _
```
where `(Int, Int) => Int` is same as `Function2[Int, Int, Int]`

- Multi-line function implementation -
```
val func: Int => Int = (value: Int) => {
    // implementation
}
```

or,

```
val func: Int => Int = { (value: Int) =>
    // implementation
}
```

> [!TIP]
> The arrow symbol (`=>`) is right-associate i.e. `Int => Int => Int` is inferred as `Int => (Int => Int)` and means a function that takes an `Int` and returns a function from `Int` to `Int`.

## Currying

- Currying functions groups parameters in consecutive function calls -
```
val adder = new Function1[Int, Function1[Int, Int]] {
    override def apply(v1: Int) = new Function1[Int, Int] {
        override def apply(v2: Int) = v1 + v2
    }
}
```

- It can then be used as -
```
val adder2 = adder(2)
val result_v1 = adder2(10)     // returns 12
val result_v2 = adder(2)(10)   // also returns 12
```

- Curried functions as lambda -
```
val adder_v1 = (x: Int) => (y: Int) =>  x + y

val adder_v2: Int => Int => Int = x => y => x + y
```

## High Order Functions

- HOFs are functions that takes other functions are input arguments and/or return functions as output.
- Example -
    - Functions as input -
    ```
    val hof1: (Int, (Int => Int)) => Int = (x, func) => x + func(x)
    ```

    - Functions as return value -
    ```
    val hof2: Int => (Int => Int) = x => (y => x + y)
    ```

## for-comprehensions

- **for-comprehensions** are syntactic sugars for `flatMap` and `map` chains.

> [!TIP]
> The `if` guard in for-comprehension is complied into `withFilter`.

- If the for-comprehension does not have a `yield` clause then it returns `Unit`, but it will only work if the collection used in the for-comprehension has `foreach` method.

## Partial Functions

- Partial functions are extension of total functions that are "partially" applicable to subset of values of input type.
```
trait PartialFunction[-A, +B] extends (A => B)
```

- PFs are based on pattern matching.
```
val pf: PartialFunction[Int, Int] = {
    case 1 => 42
    case 2 => 56
    case 5 => 999
}
```

- Lifting PFs to total functions -
```
val liftedPF: Int => Option[Int] = pf.lift
```

- Chaining -
```
val anotherPF: PartialFunction[Int, Int] = {
    case 45 => 54
}

pf.orElse[Int, Int](anotherPF)
```

- HOFs accepts partial functions -
```
case class Person(name: String, age: Int)
val persons: List[Person] = ???

persons.map {
    case Person(n, a) => Person(n, a + 1)
}
```

## Functional Collections

- **`Set`** -
    - `Set` is a function `(A => Boolean)`, therefore, has an `apply` method that accepts input type `A` and returns `Boolean` - if element is present in the set or not.
    - `Set` is a total function i.e. always returns either `true` or `false`.
    - Using set as a function - `aSet` acts like a function accepting `Int` and returning `Boolean` -
    ```
    val aSet = Set(1, 2, 3, 4)
    val aList = (1 to 10).toList

    aList.filter(aSet)
    ```

- **`Seq`** -
    - `Seq` is a function `PartialFunction[Int, A]` - `apply` returns an element present at a given index.
    - `Seq` is a partial function - throws an error in case of index out-of-bound. 

- **`Map`** -
    - `Map` is a partial function `PartialFunction[K, V]` - `apply` returns the value present at given key.

## Property Based Set

```
class PBSet[A](property: A => Boolean) extends (A => Boolean) {
    def contains(elem: A): Boolean = property(elem)

    infix def +(elem: A): PBSet[A] = 
        new PBSet(x => x == elem || property(x))

    infix def ++(anotherSet: PBSet[A]): PBSet[A] =
        new PBSet[A](x => property(x) || anotherSet(x))

    def filter(predicate: A => Boolean): PBSet[A] =
        new PBSet(x => property(x) && predicate(x))

    infix def -(elem: A): PBSet[A] =
        filter(x => x != elem)

    infix def &(anotherSet: PBSet[A]): PBSet[A] =
        filter(anotherSet)

    infix def --(anotherSet: PBSet[A]): PBSet[A] =
        filter(!anotherSet)     

    def unary_! : PBSet[A] =
        new PBSet(x => !contains(x))

    // map, flatMap & foreach can run infinitely, so cannot be implemented    
}
```

- A Set containing everything of given type - 
```
class AllInclusiveSet[A]() extends PBSet[A](_ => true)
```

## Eta-expansion

- Eta-expansion is a process by which the scala compiler converts a method into a function value.
```
def curriedAdder(a: Int)(b: Int): Int = a + b

val add4: Int => Int = curriedAdder(4)
```

```
def increment(x: Int): Int = x + 1
List(1, 2, 3).map(increment)
```

- With underscores -
```
def concatenate(a: String, b: String, c: String): String = a + b + b

val insertName: String => String = concatenate("Hello ", _: String, " How are you?")
```

> [!TIP]
> Converting normal function to curried -
> ```
> val adder = (a: Int, b: Int): Int = a + b
> val curriedAdder: Int => Int => Int = adder.curried
> ```

> [!WARN]
> Methods vs functions with 0-lambdas
> ```
> def fun(f: () => Int) = f() + 1
>
> def method: Int = 42
> def parenMethod(): Int = 42
>
> fun(23)                   // will not compile
> fun(method)               // eta-expansion not possible - will not compile
> fun(parenMethod)          // eta-expansion possible - compiles fine
> fun(() => parenMethod())  // compiles
> ```