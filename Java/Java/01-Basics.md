# Java

- __Strongly typed__ - every variable must have a declared type
- __Compile and run__ -
  ```
  javac HelloWorld.java    // generates HelloWorld.class (bytecode)
  java HelloWorld          // executes bytecode
  ```

## JShell

- REPL environment - `jshell` to start, `/exit` to quit
- Supports tab completion and variable reuse via `$n` references

## Hello World

```java
public class MyApp {
  void main() {
    IO.println("Hello, World!");
  }
}
```

## Comments

- `//` - single-line
- `/* ... */` - multi-line
- `/** ... */` - Javadoc

## Variables and Constants

```java
int i;
double d;
long l;
boolean b;

int i, j;                   // multiple declarations
var x = 5;                  // type inferred as int
final double PI = 3.14;     // constants
```

## Input and Output

```java
String name = IO.readln("Prompt: ");                              // always returns String

int age = Integer.parseInt(IO.readln("Age: "));                   // casting to int
double rate = Double.parseDouble(IO.readln("Interest rate: "));   // casting to double

char[] passwd = System.console().readPassword("Password: ");      // reading passwords
Arrays.fill(passwd, '*');                                         // overwrite immediately after use

IO.print("%8.2f".formatted(10000.0 / 3.0));                       // 3333.33
IO.println("%,.2f".formatted(10000.0 / 3.0));                     // 3,333.33
IO.print("Hello, %s. Age: %d.".formatted(name, age + 1));       // multiple arguments
```

> [!TIP]
> Use `%s` for any object - invokes `Formattable.formatTo()` if implemented, else `toString()`

> [!TIP]
> Formatting is locale-sensitive - use `String.format(Locale.US, "%8.2f", x)` for fixed locale

## Data Types

### Integer Types

- Signed two's complement representation

| Type    | Size   | Range                          | Constants                          |
|---------|--------|--------------------------------|------------------------------------|
| `byte`  | 1 byte | $-2^7$ to $2^7-1$              | `Byte.MIN_VALUE`, `Byte.MAX_VALUE` |
| `short` | 2 bytes| $-2^{15}$ to $2^{15}-1$        | `Short.MIN_VALUE/MAX_VALUE`        |
| `int`   | 4 bytes| $-2^{31}$ to $2^{31}-1$        | `Integer.MIN_VALUE/MAX_VALUE`      |
| `long`  | 8 bytes| $-2^{63}$ to $2^{63}-1$        | `Long.MIN_VALUE/MAX_VALUE`         |

> [!TIP]
> `1_000_000` - underscores allowed in numeric literals, ignored by compiler

### Floating-Point Types

| Type     | Size   | Precision      | Default  |
|----------|--------|----------------|----------|
| `float`  | 4 bytes| 6–7 digits     | `0.0f`   |
| `double` | 8 bytes| 15–16 digits   | `0.0d`   |

- `2.0 - 1.1` → `0.8999...` - use `BigDecimal` for financial calculations
- IEEE 754 special values - `+Inf`, `-Inf`, `NaN` (`0.0/0`)
- `x == Double.NaN` always `false` - use `Double.isNaN(x)`
- `0.0 == -0.0` is `true` - use `Double.compare(x, -0.0) == 0` to distinguish

### `char` Type

- 16-bit unsigned integer, single quotes - `'A'` = 65

### `boolean` Type

- `true` / `false`

> [!TIP]
> Each primitive wrapper exposes `<Type>.SIZE` (bits) and `<Type>.BYTES`
>
> For eg - `Integer.SIZE` = 32, and `Integer.BYTES` = 4

## Enum Type

```java
enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }
Size s = Size.MEDIUM;
Size.valueOf("MEDIUM");                           // Size.MEDIUM
Size.valueOf("Medium");                           // Error - case-sensitive
Size.values();                                    // all values as array
```

- Variable holds one defined value or `null`

## Arithmetic Operators

- `int / int` → `int`; `int / 0` → exception; `double / 0` → `Infinity`
- `n % 2` → `0` even, `1` odd positive, `-1` odd negative
- `Math.floorMod(-5, 2)` → `1`; `Math.floorMod(5, -2)` → `-1`
- __Numeric promotion rules (binary ops)__ -
  - Either `double` → both `double`
  - Else either `float` → both `float`
  - Else either `long` → both `long`
  - Else both → `int`

## Casts

```java
double x = 9.997;
int nx = (int) x;    // 9 - truncates
(byte) 300           // 44 - wraps on overflow
```

- Java 25 preview - safe casts via `instanceof` pattern: `if (n instanceof byte b)`
- __Legal conversions between numeric types__ -

  ![Legal conversions between numeric types](assets/numeric_types_conversions.png)

  - Solid arrows - conversions without information loss. 
  - Dotted arrows - conversions that may lose precision, eg -
    ```
    int n = 123456789;
    float f = n;            // 1.23456792E8 - magnitude correct, precision lost
    ```

## Assignment Operators

- Compound assignment (`+=`, `-=`, etc.) performs implicit narrowing cast
- `x += 3.5` on `int x` → `3` (equivalent to `x = (int)(x + 3.5)`)
- `x = x + 3.5` on `int x` → compiler error
- Assignment is an expression - `int y = x += 4` is valid

> [!TIP]
> Enable lossy conversion warnings - `javac -Xlint:lossy-conversions MyApp.java`

## Increment / Decrement

- Prefix `++x` - increment before use; postfix `x++` - use then increment

## Relational and Logical Operators

- `==`, `!=`, `<`, `>`, `<=`, `>=`
- `&&`, `||` - short-circuit; `&`, `|` on booleans - no short-circuit
- `condition ? expr1 : expr2`

## Bitwise Operators

- `&`, `|`, `^`, `~`
- `>>` - sign-extends; `>>>` - zero-fills; no `<<<`
- `Integer.bitCount(n)`, `Integer.reverse(n)`

## Big Numbers

```java
BigInteger.valueOf(100)
new BigInteger("123456789012345678901234567890")
BigDecimal.valueOf(0.1)      // safe
new BigDecimal("0.1")        // safe
new BigDecimal(0.1)          // avoid - imprecise
```

- Arithmetic via methods - `a.add(b)`, `a.multiply(b)`
- `divide(other)` - throws if result is repeating; `divide(other, RoundingMode.HALF_UP)` - rounds

> [!TIP]
> `BigInteger.parallelMultiply()` (Java 19+) - uses multiple CPU cores

## Strings

- Sequence of `char` values; JVM may store as bytes (single-byte) or chars internally
- __Immutable__ - enables string pool sharing
- `+` concatenates; non-string operands are converted via `toString()`

> [!WARNING]
> `"value: " + 1 + 2` → `"value: 12"` - left-to-right; use parentheses to force arithmetic

> [!WARNING]
> `':' + 8000` → `8058` - char arithmetic, not string concatenation

> [!WARNING]
> Never use `==` to compare strings - compares references; use `.equals()`

> [!TIP]
> `CharSequence` - the common interface for all string types

## `StringBuilder`

- Mutable buffer - avoids creating intermediate `String` objects per concatenation
- Only way to reverse Unicode characters of a string -
  ```java
    new StringBuilder(original).reverse().toString()
  ``` 

## Control Flow

### Loops

```java
while (condition) { }

do { } while (condition);

for (int i = 0, j = 10; i < 10; i++, j--) { }       // multiple vars must be of same type
```

### Labeled `break` / `continue`

```java
outer:
  while (...) {
    for (...) {
        if (n < 0) break outer;     // exits outer loop
    }
  }
```

- `continue` - jumps to loop header (`while`) or update (`for`)
- Labeled `continue` - jumps to matching label's header

### Switch Expression

```java
String s = switch (code) {
    case 0 -> "Spring";
    case 3, 4, 5 -> "Winter";
    case null -> "null";        // explicit - default does NOT match null
    default -> "???";
};
```

- Case label must be a compile-time constant matching selector type
- Enum labels omit the type name - `case SMALL ->` not `case Size.SMALL ->`
- All enum values covered → `default` optional; String/numeric → `default` required
- Multi-line case - use `yield` to return value; `return`/`break`/`continue` not allowed

> [!TIP]
> Java 23 preview - `switch` selector supports `float`, `double`, `long`, `boolean`

### Switch Statement

```java
switch (choice) {
    case 1: ...; break;
    default: ...;
}
```

- Without `break` - falls through to next case
- Detect unintentional fallthrough - `javac -Xlint:fallthrough`
- Suppress intentional fallthrough - `@SuppressWarnings("fallthrough")`

## Arrays

```java
int[] a = new int[100];                        // all zeros
int[] primes = { 2, 3, 5, 7, 11, 13 };        // inferred size
var a = new int[n];                            // runtime-sized
```

- Fixed size after creation - use `ArrayList` for resizable collections
- `a.length`, `a[i]`, `Arrays.sort(a)`, `Arrays.toString(a)`

### Copying

```java
int[] copy = Arrays.copyOf(src, src.length);
int[] grown = Arrays.copyOf(src, src.length * 2);    // extra elements → 0/false/null
```

- Assignment copies reference, not values - `b = a` means both point to same array
- Arrays are heap-allocated

### Multi-dimensional Arrays

```java
double[][] grid = new double[ROWS][COLS];
int[][] magic = {{16,3},{5,10}};
Arrays.deepToString(grid);
```

- Java models as arrays-of-arrays - each row is an independent heap object
- Ragged arrays are valid - rows can have different lengths

## For-Each Loop

```java
for (int element : a) IO.println(element);
```

- Works on arrays and any `Iterable` (e.g., `ArrayList`)