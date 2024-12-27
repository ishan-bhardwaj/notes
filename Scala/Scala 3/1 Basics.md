## Values and Types

- `val` is immutable value and `var` is mutable variable.
- For eg -
```
val x: Int = 42
x = 45              // compilation error - "Reassignment to val x"
```

```
var y: Int = 42
y = 45              // works!
```

- Common Types -
    - Short - `val aShort: Short = 10` - 2 bytes representation
    - Int - `val anInt: Int = 42` - 4 bytes representation
    - Long - `val aLong: Long = 23783573680L` - 8 bytes representation. Note that trailng `L` with the number is only used to distinguish the `Int` and `Long` for readability, hence it is optional. 
    - Float - `val aFloat: Float = 2.5f` - 4 bytes representation
    - Double - `val aDouble: Double = 3.14` - 8 bytes representation
    - Boolean - `val aBool: Boolean = false`
    - Char - `val aChar: Char = 'a'`
    - String - `val aString: String = "hello"`

> [!Note]
> Scala `String` type is just an alias for Java `String` i.e.
> ```
> type String = java.lang.String
> ```

> [!NOTE]
> **Instructions** vs **Expressions** - Instructions are executed, whereas expressions are evaluated to a value.
> Everything in scala is an expression. Eg -
> ```
> val x: Int = if (true) 42 else 56
> ```

## Functions

- Method syntax -
```
def funName(arg1: String, arg2: Int): String = { /* function body */ }
```

- Methods with no arguments -
```
def fun_v1(): Int = 45
def fun_v2: Int = 45
```

> [!TIP]
> Methods with empty parameter lists (eg, `fun_v1()`) implies that the method performs an action or computation when invoked. When _eta-expanded_, it is converted to a function of type `() => Int`

> Methods without empty parameter lists (eg, `fun_v2()`) are often used for read-only accessors or values that don't require computation, giving them a more "field-like" usage. They cannot be eta-expanded.

> Trying to eta-expand directly causes a compilation error -
> ```
> val f2: () => Int = fun_v2 _  // ERROR: No eta-expansion occurs for parameterless methods
> ```

> [!NOTE]
> Scala compiler provides **Type Inference** - can infer types based on the return value, eg -
> ```
> val x = 10        // type of `x` is inferred as `Int`
> def fun() = 10    // return type of `fun()` is inferred as `Int`
> ```
> However, we need to explicitly specify the return type for recursive functions.

### Tail-recursion

- Factorial -
```
def factorial_v1(n: Int): Int =
    if (n <= 0) 1
    else n * factorial_v1(n - 1)

@tailrec def factorial_v2(n: Int, acc: Int): Int =
    if (n <= 0) acc
    else factorial_v2(n - 1, n * acc)    
```

- In, `factorial_v1`, the JVM has to keep track of the call stack and can throw `StackOverflowException` for a large number. In `factorial_v2`, the JVM computes everything in the same stack frame as the `acc` keeps track of the partial results, hence no risk of `StackOverflowException` and therefore, also called as _Stack recursion_.
- `@tailrec` annotation (from `scala.annotation.tailrec`) asks the compiler to validates whether the recursive call is in tail position or not.

## Call by name vs Call by value
- **Call by value** - arguments are evaluated before function evaluation, eg - 
```
def printCBV(x: => Long): Unit = {
    println(x)
    println(x)
}

fun(System.currentTimeMillis)   // prints the same number (the current time value) twice.
```

- **Call by name** - delays evaluation of the argument and will be evaluated _everytime_ they are used in the function body, eg -
```
def printCBV(x: => Long): Unit = {
    println(x)
    println(x)
}

fun(System.currentTimeMillis)   // prints different numbers as the `System.currentTimeMillis` is evaluated twice
```

- If a call-by-name argument is not used in the function body, it will not be evaluated at all.

## String Interpolations

- **s-interpolation** -
```
val name = "John"
val age = 20
val amount = 4.2
s"Hello, I am $name and I'm ${age + 1} years old"
s"You total price is: $amount"     // 4.2
```

- **f-interpolation** - used to specify formats
```
val amount = 4.2
f"You total price is: $amount%2.2f"     // 4.20
```

- **raw-interpolation** - can ignore escape characters
```
raw"This is a \n newline"
```