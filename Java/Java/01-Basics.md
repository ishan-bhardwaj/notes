# Java

-  _Strongly typed language_ - every variable must have a declared type.

```
javac --version         
// javac 25.0.1

java --version
// java 25.0.1 2025-10-21 LTS
// Java(TM) SE Runtime Environment (build 25.0.1+8-LTS-27)
// Java HotSpot(TM) 64-Bit Server VM (build 25.0.1+8-LTS-27, mixed mode, sharing)
```

- Compile and Launch -

  ```
  javac HelloWorld.java          // compiles and generates bytecode - HelloWorld.class in same directory
  java HelloWorld                // executes main method in HelloWorld.class (bytecode)

  java HelloWorld                // compiles and executes in single command
  ```

## JShell
  
- Provides a _read-evaluate-print loop_ or _REPL_.
- Supports tab completion.
  ```
  > jshell              // starts JShell
  > /exit               // exits JShell
  ```

- Example -
  ```
  jshell> "Hello World".length()
  $1 ==> 11

  jshell> $1 + 5
  $2 ==> 16

  jshell> var result = 10 + 5
  result ==> 15
  ```

## Hello World

```
public class MyApp {
  void main() {
    IO.println("Hello, World!");
  }
}
```

## Comments
  
- Single-line comment - `//`
- Multi-lne comment - `/* ... */`
- Multi-line comment - `/** ... */`
  - Used for generating documentation automatically.

## Variables and Constants

- Variables -
  ```
  double salary;
  int vacationDays;
  long earthPopulation;
  boolean done;

  int i, j;                     // declaring multiple variables in same line
  ```

- Constants -
  ```
  final double PI = 3.14;
  ```

## Type inference
  
- Types can be inferred from the value -
```
var x = 5;                // x is an int
var greet = "Hello";      // greet is a String
```

## Input and Output

- `IO.readln()` reads one line of input and returns it as a `String`.
- Display prompt -
  ```
  String name = IO.readln("What is your name? ");
  ```

- Read an integer or double -
  ```
  int age = Integer.parseInt(IO.readln("How old are you? "));
  double rate = Double.parseDouble(IO.readln("Interest rate: "));
  ```

- Read passwords -
  ```
  char[] passwd = System.console().readPassword("Password: ");
  Arrays.fill(passwd, '*');                     // overwrite with *'s immediately after use
  ```

- Formatting output -
  ```
  IO.print("%8.2f".formatted(x));                                             // 3333.33
  IO.print("Hello, %s. Next year, you'll be %d.".formatted(name, age + 1));   // multiple arguments
  ```

- Specify flags - 
  - The comma flag adds group separators, eg - 
    ```
    IO.println("%,.2f".formatted(10000.0 / 3.0));           // prints 3,333.33
    ```

  - Use multiple flags - `"%,(.2f"` uses group separators and enclose negative numbers in parentheses.

> [!TIP]
> Use `%s` conversion to format any object. 
> 
> If it implements `Formattable` interface, its `formatTo` method is used, otherwise `toString()` is used.

> [!TIP]
> Formatting depends on the system locale (e.g., Germany uses `,` instead of `.`). 
> 
> Specifying a fixed locale -
> ```
> IO.print(String.format(Locale.US, "%8.2f", x));
> ```

# Data Types

## Integer Type

- Integers are represented using _signed two's completement scheme_.

| Type    | Storage Requirement | Range (Inclusive)              | Default | Min Value           | Max Value           |
|---------|---------------------|--------------------------------|---------|---------------------|---------------------|
| `byte`  | 1 byte              | $–2^7 \text{ to } 2^7−1$       | 0       | `Byte.MIN_VALUE`    | `Byte.MAX_VALUE`    |
| `short` | 2 bytes             | $–2^{15} \text{ to } 2^{15}−1$ | 0       | `Short.MIN_VALUE`   | `Short.MAX_VALUE`   |
| `int`   | 4 bytes             | $–2^{31} \text{ to } 2^{31}−1$ | 0       | `Integer.MIN_VALUE` | `Integer.MAX_VALUE` |
| `long`  | 8 bytes             | $–2^{63} \text{ to } 2^{63}−1$ | 0       | `Long.MIN_VALUE`    | `Long.MAX_VALUE`    |


> [!TIP]
> Use underscores in long numbers for readability, eg - `1_000_000` - Java compiler simply ignores the underscores.

## Floating-Point Type

| Type     | Storage Requirement | Range                          | Default | Precision            |
| -------- | ------------------- | ------------------------------ | ------- | -------------------- |
| `float`  | 4 bytes             | $±3.4 × 10^{38}$ (7 digits)    | `0.0f`  | 6-7 decimal digits   |
| `double` | 8 bytes             | $±1.8 × 10^{308}$ (15 digits)  | `0.0d`  | 15-16 decimal digits |

- Floating-point numbers are not suitable for financial calculations because they can produce roundoff errors -    
  - Eg - `2.0 - 1.1` returns `0.8999999999999999`
  - Use `BigDecimal` for exact precision.

- Exponentials - `1.729E3 = 1729`
- Hex floating literals - `0x1.0p-3 = 0.125`, where `p` = exponent (base `2`), mantissa = hex, exponent = decimal.

- IEEE 754 special values -
  - Positive infinity, eg - `Positive number / 0` → `Positive infinity`
  - Negative infinity, eg - `Negative number / 0` → `Negative infinity`
  - `NaN` (Not a Number), eg - `0.0 / 0` → `NaN`, `sqrt(negative number)` → `NaN`

- `x == Double.NaN` will always return `false` - all `NaN` values are distinct.
  - Use `Doulbe.isNaN(x)` instead.

- `0.0 == -0.0` will always return `true`.
  - Check whether it is negative - `Double.compare(x, -0.0) == 0`

## `char` type

- 16-bits unsigned integer
- Use single quotes - `'A' = 65`
- `char` literals use _single quotes_, e.g., `'A'` = 65.

> [!TIP]
> Each primitive wrapper has two constants -
>   - `<Type>.SIZE` - size in bits
>   - `<Type>.BYTES` - size in bytes
>
> Example -
>   - `Integer.SIZE` - 32
>   - `Integer.BYTES` - 4

## `boolean` type

- `true` / `false`
- Used for evaluating logical conditions.

## Enum Type

- Used when a variable should hold only a _fixed set of values_.
- Example -

  ```
  enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }

  Size s = Size.MEDIUM;           // declare variables of the enum type
  Size.valueOf("MEDIUM");         // returns Size.MEDIUM
  Size.valueOf("Medium");         // Error!
  Size.values();                  // returns all Size values in an array
  ```

- Enum type variable (e.g., `Size`) -
  - can only hold one of its defined values, or
  - `null` if not set.

## Arithmetic Operators

- `+`, `-`, `*`, `/`
- Division -
  - `int / int` returns `int`.
  - If anything is `float`/`double`, the result is `float`/`double`.
- Division by `0` -
  - `int / 0` throws an exception.
  - `float or double / 0` returns `Infinity`.
- Modulus `n % 2` -
  - `0` for even `n`.
  - `1` for odd positive `n`.
  - `-1` for odd negative `n`.
- `Math.floodMod` -
  - `Math.floorMod(-5, 2)` returns `1`.
  - `Math.floorMod(5, -2)` returns `-1`.

- __Legal conversions between numeric types__ -

  ![Legal conversions between numeric types](assets/numeric_types_conversions.png)

  - Solid arrows - conversions without information loss. 
  - Dotted arrows - conversions that may lose precision, eg -
    ```
    int n = 123456789;
    float f = n;            // 1.23456792E8 - magnitude correct, precision lost
    ```
    
  - Binary operators convert operands to a common type before computing -
    - If either operand is `double`, convert the other to `double`.
    - Else if either operand is `float`, convert the other to `float`.
    - Else if either operand is `long`, convert the other to `long`.
    - Else convert both operands to `int`.

- __Casts__ -
  - Conversions in which loss of information is possible are done by means of _casts_ -
    ```
    double x = 9.997;
    int nx = (int) x;           // 9
    ```

  - Java 25 preview adds safe casts using `instanceof` pattern matching.
    - Example - `if (n instanceof byte b)`
    - If `n` fits in a `byte` without loss, `b` is automatically set to `(byte) n`.

> [!NOTE]
> Casting to a smaller numeric type can truncate the value if it’s out of range, eg - `(byte) 300` becomes `44`.

## Assignment Operators

- Compound assignment operators - `+=`, `-=`. `*=`, `/=`, `%=`
- Compound assignment operators perform an implicit cast to the type of the left-hand side - even if the conversion is narrowing.
- Example -
  ```
  int x = 0;
  x += 3.5;                 // returns 3 - fractional part is discarded

  // equivalent to -
  x = (int)(x + 3.5); 

  // but
  x = x + 3.5;              // compiler error!
  ```

- Java 20+ can warn about such lossy conversions when linting is enabled. To enable such warnings -
  ```
  javac -Xlint:lossy-conversions MyApp.java
  ```

- In Java, an assignment is an _expression_ and returns the assigned value, eg -
  ```
  int x = 1;
  int y = x += 4;           // y = 5
  ```

## Increment & Decrement Operators

- `++`, `--`
- Works on variables, not on literals (e.g., `4++` is illegal).
- Two forms - 
  - Prefix (`++x` / `--x`) - value is changed before being used in an expression.
  - Postfix - (`x++` / `x--`) - value is used first, then changed.

- Example -
  ```
  int m = 7;
  int n = 7;

  int a = 2 * ++m;              // a = 16, m = 8
  int b = 2 * n++;              // b = 14, n = 8
  ```

## Relational Operators

- Equality (`==`) and inequality (`!=`).
- Comparisons - `<`, `>`, `<=`, `>=`
- Logical operators -
  - Logical AND - `&&`
  - Logical OR - `||`
  - Logical NOT - `!`
- __Short-circuit Evaluation__ -
  - `&&` stops evaluating if the first operand is `false`.
  - `||` stops evaluating if the first operand is `true`.

- __Conditional Operator (`?:`)__ -
  - Syntax - `condition ? expression1 : expression2`
  - Returns `expression1` if condition is `true`, otherwise returns `expression2`.

## Bitwise Operators

- Work on bit patterns - `&` ("and"), `|` ("or"), `^` ("xor"), `~` ("not").
- `&` and `|` work on boolean values also and return `boolean` - similar to `&&` and `||` - but they do not provide short-circuiting.

- __Bit Shift operators__ -
  - `<<` - left shift
  - `>>` - right shift (sign-extends the leftmost bit)
  - `>>>` - unsigned right shift (fills leftmost bits with 0)
  - No `<<<` operator exists

- __Integer Bit-level Methods__ -
  - `Integer.bitCount(n)` - number of 1 bits in binary form of `n`.
  - `Integer.reverse(n)` - reverses bits of `n`.

## Big Numbers

- `java.math.BigInteger` -
  - Used for very large integer arithmetic.
  - From a normal number - `BigInteger.valueOf(100)`
  - From a long number (string) - `new BigInteger("123456")`
  - Constants - `BigInteger.ZERO`, `BigInteger.ONE`, `BigInteger.TWO`, `BigInteger.TEN`

- `java.math.BigDecimal` -
  - Used for very precise decimal numbers (money, financial calculations, etc.)
  - Always construct from integers or string.
    ```
    new BigDecimal(0.1);      // avoid - returns 1000000000000000055511151231257827021181583404541015625
    BigDecimal.valueOf(0.1);  // returns 0.1
    new BigDecimal("0.1");    // returns 0.1
    ```

- Arithmatic ethods -
  ```
  BigInteger c = a.add(b);                                      // c = a + b
  BigInteger d = c.multiply(b.add(BigInteger.valueOf(2)));      // d = c * (b + 2)
  ```

> [!NOTE]
> Only `+` operator is overloaded for string concatenation.

> [!TIP]
> Java 19 feature - `parallelMultiply()` - works like `multiply()` but may be faster using multiple CPU cores.

> [!NOTE]
> Division -
>   - `divide(other)` - throws exception if result is not finite (repeating decimal).
>   - `divide(other, mode)` - allows rounding when the result is repeating.
>
> Example - `RoundingMode.HALF_UP`
>   - Round down - 0–4
>   - Round up - 5–9

## Strings

- Java strings are sequences of `char` values.
- The JVM may store strings as byte sequences for single-byte characters and as char sequences for others, rather than always using `char[]`.

- __Strings are immutable__ - 
  - You cannot change a character inside an existing string.
  - Immutable strings allow sharing, so the compiler can store strings in a common pool and multiple variables can reference the same characters without copying.

- __Concatenation__ -
  - Use `+` operator to concatenate two strings.
  - When you concatenate a string with a non-string value, Java converts the non-string value to a string.
  - Example -
    ```
    int age = 15;
    String rating = "PG" + age;           // "PG15"
    ```

> [!WARNING]
> String concatenation uses + and is evaluated left to right.
> ```
> int age = 42;
> String output = "Next year, you'll be " + age + 1 + ".";
> ```
> Result - `"Next year, you'll be 421."`
>
> Fix - use parentheses - 
> ```
> String output = "Next year, you'll be " + (age + 1) + ".";
> ```

> [!WARNING]
> String concatenation only works with strings, not char literals — `':' + 8000` produces the integer `8058`, not a string (The colon character has Unicode value `58`).

> [!WARNING]
> Do not use the `==` operator to test whether two strings are equal - it only determines whether or not the strings are stored in the same location. 
>
> Only string literals are shared, not strings that are computed at runtime. Therefore, never use `==` to compare strings. 
>
> Always use `equals` instead.

> [!TIP]
> `CharSequence` is the interface type to which all strings belong. 

## `StringBuilder`

- String concatenation is inefficient because each concatenation creates a new String object.
- `StringBuilder` avoids this by building a string in a _mutable buffer_.

- Also, `String` class doesn’t have a method to reverse the Unicode characters of a string, but `StringBuilder` does - 
  ```
  String reversed = new StringBuilder(original).reverse().toString();
  ```