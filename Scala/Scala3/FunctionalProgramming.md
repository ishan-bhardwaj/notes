## Functions

- In functional programming, functions are considered as _first-class citizens_, but the JVM only understands classes and objects. 
- Therefore, a `Function` is defined as a `trait` in Scala and all functions are instances of `FunctionX` trait where `X` is the number of arguments the function can accept and can range from `0` to `22` -
```
val adder = new Function1[Int, Int, Int] {
    override def apply(v1: Int, v2: Int): Int = v1 + v2
}

val sum: Int = adder(2, 3)       // same as adder.apply(2, 3)
```

- Syntax sugars for function implementations - 
```
val adder_v2: (Int, Int) => Int = (v1, v2) => v1 + v2

// or
val adder_v3 = (v1: Int, v2: Int) => v1 + v2

// or
val adder_v2: (Int, Int) => Int = _ + _
```
where `(Int, Int) => Int` is same as `Function2[Int, Int, Int]`

## Currying

- Currying functions groups parameters in consecutive function calls -
```
val adder = new Function1[Int, Function1[Int, Int]] {
    override def apply(v1: Int) = new Function1[Int, Int] {
        override def apply(v2: Int) = v1 + v2
    }
}

// or
val adder_v2: Int => Int => Int = (v1: Int) => (v2: Int) => v1 + v2
```

- It can then be used as -
```
val adder2 = adder(2)
val result_v1 = adder2(10)     // returns 12
val result_v2 = adder(2)(10)   // also returns 12
```

