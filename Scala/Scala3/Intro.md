# Scala

- **Scala** stands for _scalable language_ - can be applied to wide range of tasks such as writing small scripts to building large systems.
- Scala programs compile to JVM bytecode.
- Statically-typed language.
- Blend of object-oriented and functional programming paradigms.
- Scala is "pure" object-oriented - every value is an object and every operation is a method call.
- Also in Scala, you can write purely functional code, but the language itself does not enforce it. 

- Key concepts of functional rogramming -
    - functions are first-class values.
    - should not have side effects i.e. the program should only depends on its inputs and produces the same output every time for specific inputs without modifying anything outside of its scope.

- **Referential Transparent** - for any given input the method call could be replaced by its result without affecting the program’s semantics.

- **Instructions** vs **Expressions** - Instructions are executed, whereas expressions are evaluated to a value.

- Everything in scala is an expression. Eg -
```
val x: Int = if (true) 42 else 56
```

## Values & Variables

- `val` is immutable value - once initialized, can never be assigned.
- `var` is mutable variable - can be reassigned through its lifetime.
- For eg -
```
val x: Int = 42
x = 45              // compilation error - "Reassignment to val x"
```

```
var y: Int = 42
y = 45              // works!
```

## Data Types

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
> - Methods with empty parameter lists (eg, `fun_v1()`) implies that the method performs an action or computation when invoked. When _eta-expanded_, it is converted to a function of type `() => Int`
> - Methods without empty parameter lists (eg, `fun_v2()`) are often used for read-only accessors or values that don't require computation, giving them a more "field-like" usage. They cannot be eta-expanded.
> - Trying to eta-expand directly causes a compilation error -
> ```
> val f2: () => Int = fun_v2 _  // ERROR: No eta-expansion occurs for parameterless methods
> ```

> [!TIP]
> Methods ending with `:` are _right-associative_ for eg - `4 :: list` is same as `list.::(4)`

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

> [!TIP]
> When a for comprehension does not end in a yield, it is translated into a series of foreach calls. Eg -
> ```
> for {
>   x <- List(1, 2)
>   y <- List(x, x * 3)
> } println((x, y))
> ```

## Scala Scripts

- Includes a top-level function annotated as `@main`, eg - `greet.scala` -
```
@main def myFun() =
    println("Hello wordl!")
```

- Running the script - `scala greet.scala`

- Providing command-line arguments -
```
@main def myFun(args: String*) =
    println("Hello " + args(0) + "!")
```

## Loops

### **while** loop

```
@main def m(args: String*) =
    var i = 0
    while i < args.length do
        if i != 0 then
            print(" ")
        print(args(i))
        i += 1
    println()
```

or,

```
@main def m(args: String*) = {
    var i = 0;
    while (i < args.length) {
        if (i != 0) {
            print(" ")
        }
        print(args(i))
        i += 1
    }
    println()
}
```

### `for` loop

```
@main def m(args: String*) =
    for arg <- args do
        println(arg)
```

> [!NOTE]
> `arg` is a _val_. `arg` can’t be reassigned inside the body of the _for_ expression. Instead, for each element of the args array, a new `arg` val will be created and initialized to the element value, and the body of the `for` will be executed.


--------------------

- Back ticks (``) notation in pattern matching is used to match the value of variable in pattern match to some other variable with the same name -
```
def login(email: String, password: String): Boolean = 
    db.get(email) match {
        case Some(`password`) => ???
        case None => ???
    }
```

Here ``Some(`password`)`` in pattern match is same as - `Some(pwd) if password = pwd`