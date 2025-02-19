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

