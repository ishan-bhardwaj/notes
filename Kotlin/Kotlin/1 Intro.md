# Kotlin
- Kotlin is a statically typed language
- Main function - 
```
fun main() {
    println("Hello World!")
}
```

### Values and Variables
```
val x: Int = 20     // immutable - cannot be reassigned
var x: Int = 10     // mutable
x = 20              // can be reassigned
```

> [!TIP]
> Kotlin provides Type Inference i.e. compiler figures out the type from the RHS of the assignment -
> `val x = 42` - here compiler will figure out the type of `x` as `Int` automatically.

- Constants - top-level values that are not present inside any block. These values are computed first when an application is launched.
```
const val x: Int = 10
```

### Common Datatypes
- Boolean - `val aBool: Boolean = true`
- Char - `val aChar: Char = 'K'`
- Byte - `val aByte: Byte = 127` (1 byte representation)
- Short - `val aShort: Short = 1234` (2 bytes)
- Int - `val anInt: Int = 20` (4 bytes)
- Long - `val aLong: Long = 5363366L` (8 bytes)
- Float - `val aFloat: Float = 2.5f` (4 bytes)
- Double - `val aDouble: Double = 3.14` (8 bytes)
- String - `val aString: String = "Hello"`

### Common Expressions
- Math Expressions -
- `+` - Addition
    - `-` - Subtraction
    - `*` - Multiplication
    - `/` - Division
    - `%` - Modulus
- Bitwise Expressions -
    - `shr` - Shift Right
    - `shl` - Shift Left
    - `ushr` - Unsigned Shift Right
    - `and` - Bitwise And
    - `or` - Bitwise Or
    - `xor` - Bitwise XOR
    - `inv` - Negates all the bits
- Comparison Expressions -
    - `==`, `!=` - Equals and Not Equals
    - `<`, `<=` - Less Than and Less Than Or Equal To
    - `>`, `>=` - Greater Than and Greater Than Or Equal To
    - `===`, `!==` - Checks whether two values refer to the same instance in memory
- Boolean Expressions -
    - `!` - Negation
    - `&&` - Logical And
    - `||` - Logical Or
- Conditional Expressions -
```
val output = if (a == b) 10 else 20
```

### `When` Expression
```
val x: Int = 10
val message = when(x) {
    10 -> "Correct"
    11 -> "Incorrect"
    else -> "Something else"
}
```
- Matching multiple values - `10, 11 ->` - matches either 10 or 11
- Matching arbitrary expressions - `10 + 1 ->` - matches with the expression result i.e. 11
- Conditions in branches -
```
val message = when {
    x < 10 -> "Correct"
    x > 100 -> "Incorrect"
    else -> "Something else"
}
```
- Testing Types -
```
val something: Any = 10
val message = when(something) {
    is Int -> "It's an Integer!"
    is String -> "It's a String!"
    else -> "Unknown Datatype"
}
```





