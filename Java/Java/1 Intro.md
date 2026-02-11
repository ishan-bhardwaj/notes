# Java

- Java is a _strongly typed language_ - meaning that every variable must have a declared type.
- To check java version installed - `javac --version` - will return - `javac 25.0.4`.
- Launching a java program from command line -
  - Compile and generate the bytecode - `javac Welcome.java` - it will create a file (`Welcome.class`) containing the bytecodes for this class and stores it in the same directory as the source file.
  - Luanch the JVM and execute the bytecode - `java Welcome` - execution starts with the code in the `main` method of the class.
  - Combining both steps in a single cmd - `java Welcome.java`

- __JShell__ -
  - Provides a _read-evaluate-print loop_ or _REPL_.
  - JShell evaluates your input java expression, prints the result, and waits for your next input.
  - Start JShell - type `jshell` in a terminal window.
  - Exit JShell - `/exit`
  - Example -
  ```
  jshell> "Hello World".length()
  $1 ==> 11

  jshell> $1 + 5
  $2 ==> 16

  jshell> var result = 10 + 5
  result ==> 15
  ```

> [!TIP]
> JShell also supports tab completion.

- __Hello World__ - 

```
public class MyApp {
  void main() {
    IO.println("Hello, World!");
  }
}
  ```

> [!NOTE]
> `IO.println()` is a variant of the println method with no arguments which just prints a blank line.

> [!TIP]
> The `IO` class also has a `print` method that doesn’t add a newline character to the output.

- __Comments__ -
  - Single-line comment - `//`
  - Multi-lne comment - `/* ... */`
  - Multi-line comment used for generating documentation automatically - `/** ... */`

## Data Types

- Eight primitive types -
  - Four integer types
  - Two floating-point number types
  - `char` type used for UTF-16 code units in the Unicode encoding scheme
  - `boolean` type for truth values

### Integer Types

| Type    | Storage Requirement | Range (Inclusive)              | Default | Min Value           | Max Value           |
|---------|---------------------|--------------------------------|---------|---------------------|---------------------|
| `byte`  | 1 byte              | $–2^7 \text{ to } 2^7−1$       | 0       | `Byte.MIN_VALUE`    | `Byte.MAX_VALUE`    |
| `short` | 2 bytes             | $–2^{15} \text{ to } 2^{15}−1$ | 0       | `Short.MIN_VALUE`   | `Short.MAX_VALUE`   |
| `int`   | 4 bytes             | $–2^{31} \text{ to } 2^{31}−1$ | 0       | `Integer.MIN_VALUE` | `Integer.MAX_VALUE` |
| `long`  | 8 bytes             | $–2^{63} \text{ to } 2^{63}−1$ | 0       | `Long.MIN_VALUE`    | `Long.MAX_VALUE`    |

> [!TIP] 
> Internally, Java uses _signed two's completement scheme_ to represent integers.

> [!NOTE]
> The ranges of the integer types do _not_ depend on the machine on which you will be running the Java code. 

- Long integers end with `L` or `l`, eg - `4000000000L`.
- Hexadecimal numbers start with `0x` or `0X`, eg - `0xCAFE`
- Octal numbers start with `0`, eg - `010` = 8
- Binary numbers start with `0b` or `0B`, eg - `0b1001` = 9
- Underscores can be added for readability, eg - `1_000_000` - Java compiler simply ignores the underscores.

- __Unsigned__ -
  - Java does _not_ have any unsigned versions of the `int`, `long`, `short`, or `byte` types.
  - If values are never negative and you need an extra bit, you can treat signed integers as unsigned, eg - 
    - A byte normally represents `−128` to `127`
    - You can instead interpret it as `0` to `255`.
    - You can store it in a byte and binary arithmetic still works (as long as it doesn’t overflow).
  - For other operations -
    - Use `Byte.toUnsignedInt(b)` to convert to an `int` (0 to 255).
    - Process the value.
    - Cast back to `byte` if needed.
  - `Integer` and `Long` also provide methods for unsigned division and remainder.

### Floating-Point Types

| Type     | Storage Requirement | Range                          | Default | Precision            |
| -------- | ------------------- | ------------------------------ | ------- | -------------------- |
| `float`  | 4 bytes             | $±3.4 × 10^{38}$ (7 digits)    | `0.0f`  | 6-7 decimal digits   |
| `double` | 8 bytes             | $±1.8 × 10^{308}$ (15 digits)  | `0.0d`  | 15-16 decimal digits |

- Java 20 adds methods for half-precision (16-bit) floats -
  - `Float.floatToFloat16` & `Float.float16ToFloat` for storing “half-precision” 16-bit floating-point numbers in `short` values.
  - Used for implementing neural networks.

- Float literals end with `F` or `f`, eg - `3.14F`.
- Floating-point literals without `F` are `double` by default, though you can optionally use `D` or `d`, eg - `3.14D`.

- `E` or `e` denotes a decimal exponent, eg - `1.729E3` = `1729`.

- __Hexadecimal notation__ -
  - Floating-point literals can be written in hexadecimal.
  - Example - `0.125` (which is $2^{-3}$) can be written as - `0x1.0p-3`.
  - In hexadecimal floating literals -
    - Use `p` (not `e`) for the exponent.
    - `e` itself is a hexadecimal digit.
    - Mantissa is hexadecimal and exponent is decimal.
    - The base of the exponent is `2`, not `10`.

- Floating-point computations follow IEEE 754 -
  - There are 3 special values for overflow/errors -
    - Positive infinity
    - Negative infinity
    - `NaN` (Not a Number)
  - Examples -
    - Positive number / 0 → Positive infinity
    - 0.0 / 0 → `NaN`
    - sqrt(negative number) → `NaN`

> [!NOTE]
> All “not a number” values (for both `Double` and `Float`) are considered distinct, therefore you _cannot_ use `x == Double.NaN` because it will always return `false`. 
>
> Better way - `Double.isNaN(x)`.

> [!NOTE]
> There are both positive and negative floating-point zeroes, `0.0` and `-0.0`, but `0.0 == -0.0` will always return `true`. To check whether a value is negative zero, use this test - `Double.compare(x, -0.0) == 0`.

> [!WARNING]
> Floating-point numbers are not suitable for financial calculations because they can produce roundoff errors (e.g., `2.0 - 1.1` prints `0.8999999999999999`), so use `BigDecimal` for exact precision.

### `char` type

- `char` literals use _single quotes_, e.g., `'A'` = 65.
- Data representation -
  - Bit depth - 16 bits unsigned integer
  - Value range - $0 \text{ to } 2^{16}−1$
  - Default - `\u0000`
- `'A'` is _not_ the same as `"A"` (string).
- `char` values can be written in hexadecimal from `\u0000` to `\uFFFF`.
- Escape sequences work in both `char` and `String` literals, e.g., `'\u005B'`, `"Hello\n"`.
- The `\u` escape sequence _can also be used outside quotes_, e.g.,
  - `void main()\u007BIO.println("Hello, World!");\u007D`
  - `\u007B` and `\u007D` are the encodings for `{` and `}`

-  Escape Sequences for Special Characters -

| Escape Sequence | Meaning | Unicode Value |
|----------------|---------|---------------|
| `\b`           | Backspace | `\u0008` |
| `\t`           | Tab | `\u0009` |
| `\n`           | Line feed | `\u000a` |
| `\r`           | Carriage return | `\u000d` |
| `\f`           | Form feed | `\u000c` |
| `\"`           | Double quote | `\u0022` |
| `\'`           | Single quote | `\u0027` |
| `\\`           | Backslash | `\u005c` |
| `\s`           | Space (text blocks only) | `\u0020` |
| `\newline`     | Join this line with the next (text blocks only) | — |

> [!WARNING]
> Unicode escape sequences are processed before the code is parsed, eg - `"\u0022+\u0022"` is not a string consisting of a plus sign surrounded by quotation marks (`U+0022`). Instead, the `\u0022` are converted into `"` before parsing, yielding `""+""`, or an empty string.

> [!WARNING]
> You must beware of `\u` inside comments, eg - `// \u000A is a newline` - yields a syntax error since `\u000A` is replaced with a newline when the program is read.

- You can use any number of `u` in a Unicode escape (e.g., `\u00E9` and `\uuu00E9` both mean `é`). 
  - This makes ASCII-only conversions reversible - a tool can add extra `u`'s to existing escapes and later restore them.

### `boolean` type

- The boolean type has two values - `true` and `false`. 
- Used for evaluating logical conditions. 
- You cannot convert between integers and boolean values, eg - `if (x = 0)` does not compile because the integer expression `x = 0` cannot be converted to a `boolean` value.

## Variables and Constants

- Variables are used to store values.
- Constants are variables whose values don’t change.

- Declaring a variable -
```
double salary;
int vacationDays;
long earthPopulation;
boolean done;
```

- Declaring multiple variables on a single line - `int i, j;`

> [!TIP]
> To check which Unicode characters are allowed in Java identifiers, use `Character.isJavaIdentifierStart` and `Character.isJavaIdentifierPart`.

> [!NOTE]
> Although `$` is allowed in identifiers, avoid using it in your code as it is reserved for compiler/tool-generated names.

- Initializing Variables -
```
int x;
int y = 10;
IO.println(x);     // ERROR  - variable not initialized
IO.println(y);     // 10

x = 5;
IO.println(x);     // 5
```

- Type inference - 
  - You do not need to declare the types of local variables if they can be inferred from the initial value -
  ```
  var x = 5;                // x is an int
  var greet = "Hello";      // greet is a string
  ```

- Use the keyword `final` to denote a constant - `final double PI = 3.14;`

> [!NOTE]
> Generally, we name the constants in all uppercase.

## Enum Types

- Used when a variable should hold only a _fixed set of values_.
- Example -
```
enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }

Size s = Size.MEDIUM;           // declare variables of the enum type
```

- A variable of an enum type (e.g., `Size`) can only hold one of its defined values or `null` if it’s not set.

## Operators 

### Arithmetic Operators

- Arithmetic Operators - `+`, `-`, `*`, `/`
- `/` operator denotes integer division if both operands are integers, and floating-point division otherwise. 
- Integer division by `0` raises an exception, whereas floating-point division by `0` yields an infinite or `NaN` result.

- When one operand of `%` is negative, the result is also negative -
  - Example - `n % 2` yields `0` for even `n`, `1` for odd positive `n`, and `-1` for odd negative `n`. 
  - This is because early computer designers chose a convenient but non-Euclidean rule, unlike the mathematical convention of always returning a non-negative remainder.
  - Better way is to use `Math.floorMod` instead of `%` to avoid negative remainders, eg - `Math.floorMod(position + adjustment, 12)` always returns a value `0–11`.
  - `floorMod` can still return negative results if the divisor is negative.

- __Legal conversions between numeric types__ -

  ![Legal conversions between numeric types](assets/numeric_types_conversions.png)

  - Solid arrows denote conversions without information loss. 
  - Dotted arrows denote conversions that may lose precision, eg -
    - `int n = 123456789; float f = n;`
    - `f` becomes `1.23456792E8` (magnitude correct, precision lost).
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

### Assignment Operators

- Compound assignment operators - `+=`, `-=`. `*=`, `/=`, `%=`
- Compound assignment operators perform an implicit cast to the type of the left-hand side - even if the conversion is narrowing.
- Example -

  ```
  int x = 0;
  x += 3.5;

  // equivalent to -
  x = (int)(x + 3.5);
  ```

  - The fractional part is discarded (`3.5` → `3`), so `x` becomes `3`.
  - No compile-time error, and no warning either.
  - Whereas, `x = x + 3.5;` does not compile and explicit cast is required to fix it.
  - Java 20+ can warn about such lossy conversions when linting is enabled. Enable such warnings with -
   ```
   javac -Xlint:lossy-conversions MyApp.java
   ```

- In Java, an assignment is an _expression_ and returns the assigned value, eg -

  ```
  int x = 1;
  int y = x += 4;
  ```

  - `x += 4` sets `x` to `5` and evaluates to `5`, which is then assigned to `y`.

### Increment & Decrement Operators

- `++` increases a variable by 1.
- `--` decreases a variable by 1.
- Both operators only work on variables, not on literals (e.g., `4++` is illegal).
- Two forms - 
  - Prefix (`++x` / `--x`) - Value is changed before being used in an expression.
  - Postfix - (`x++` / `x--`) - Value is used first, then changed afterward.

- Example -

```
int m = 7;
int n = 7;

int a = 2 * ++m;              // a = 16, m = 8
int b = 2 * n++;              // b = 14, n = 8
```

### Relational Operators

- Equality (`==`) and inequality (`!=`).
- Comparisons - `<`, `>`, `<=`, `>=`
- Logical operators -
  - Logical AND - `&&`
  - Logical OR - `||`
  - Logical NOT - `!`
- __Short-circuit Evaluation__ -
  - `&&` stops evaluating if the first operand is `false`.
  - `||` stops evaluating if the first operand is true.

- __Conditional Operator (`?:`)__ -
  - Syntax - `condition ? expression1 : expression2`
  - Returns `expression1` if condition is `true`, otherwise returns `expression2`.

### Bitwise Operators

- Work on bit patterns - `&` ("and"), `|` ("or"), `^` ("xor"), `~` ("not").
- `&` and `|` work on boolean values and return boolean - similar to `&&` and `||` - but they do not provide short-circuiting i.e. both operands are always evaluated before result is computed.

- __Bit Shift operators__ -
  - `<<` - left shift
  - `>>` - right shift (sign-extends the leftmost bit)
  - `>>>` - unsigned right shift (fills leftmost bits with 0)
  - No `<<<` operator exists

- __Integer Bit-level Methods__ -
  - `Integer.bitCount(n)` - number of 1 bits in binary form of `n`.
  - `Integer.reverse(n)` - reverses bits of `n`.


## `Math` class

```
double x = 4;
double y = 2;

Math.sqrt(x);       // square-root - 2.0
Math.pow(x, y)      // Power - 16

double x = 9.997;
Math.round(x);      // round a floating-point number to the nearest integer - 10
```

- Trigonometric functions -

```
Math.sin
Math.cos
Math.tan
Math.atan
Math.atan2
```

- Logarithmic functions -

```
Math.exp
Math.log
Math.log10
```

- `Math.clamp` that forces a number to fit within given bounds -

```
Math.clamp(-1, 0, 10)         // too small, yields lower bound 0
Math.clamp(11, 0, 10)         // too large, yields upper bound 10
Math.clamp(3, 0, 10)          // within bounds, yields value 3
```

- Mathematical constants -

```
Math.PI
Math.TAU
Math.E
```

- To avoid `Math` prefix - `import static java.lang.Math.*;`

> [!NOTE]
> `Math` class use the routines in the computer’s floating-point unit for fastest performance. If completely predictable results are more important than performance, use the `StrictMath` class instead.

- The `Math` class provides several methods to make integer arithmetic safer -
  - Integer operators silently overflow and give incorrect results, eg - `1000000000 * 3` becomes `-1294967296`.
  - Use `Math.multiplyExact` (and other `*Exact` methods) to detect overflow.
  - Overflow causes an exception instead of a wrong value.
  - Other methods include -
    - `addExact`, `subtractExact`
    - `incrementExact`, `decrementExact`
    - `negateExact`, `absExact`, `powExact`
  - These methods work for `int` and `long` types.


## Switch Expressions

- Used to choose among more than two values.
- Example -

```
String seasonName = switch (seasonCode) {
    case 0 -> "Spring";
    case 1 -> "Summer";
    case 2 -> "Fall";
    case 3 -> "Winter";
    default -> "???";
};
```

- Expression after switch is the `selector`.
- A case label must be a compile-time constant whose type matches the selector type.

> [!TIP]
> Java 23 Preview - Switch selector can be float, double, long, or boolean.

- Multiple labels in a `case` -

```
int numLetters = switch (seasonName) {
    case "Spring", "Summer", "Winter" -> 6;
    case "Fall" -> 4;
    default -> -1;
};
```

- __Enums in Switch__ -
  - Enum type labels can omit enum name.

  ```
  enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }

  String label = switch (itemSize) {
    case SMALL -> "S";                    // no need to use Size.SMALL
    case MEDIUM -> "M";
    case LARGE -> "L";
    case EXTRA_LARGE -> "XL";
  };
  ```

  - If all enum values covered, `default` is optional; otherwise required.
  - Switch with numeric or String selector must always have `default`.
  
- __Null Selector Handling__ -
  - If selector is `null`, `NullPointerException` is thrown.
  - To handle null, add - `case null -> "???";`
  - Note that `default` does NOT match `null`.

> [!TIP]
> If you cannot compute the result in a single expression, use braces and a `yield` statement -
> ```
> case "Spring" -> {
>    IO.println("spring time!");
>    yield 6;
> }
> ```

> [!NOTE]
> You cannot use `return`, `break`, or `continue` statements in a switch expression.

## Switch statements

```
switch (choice) {
    case 1:
        ...
        break;
    case 2:
        ...
        break;
    case 3:
        ...
        break;
    case 4:
        ...
        break;
    default:
        IO.println("Bad input");
}
```

- __Fallthrough behavior__ - if a case does not end with `break`, execution continues into the next `case`.

- To detect fallthrough mistakes - 
  - Compile with - `-Xlint:fallthrough`
  - The compiler will warn when a case does not end with break.
  - But if fallthrough is intentional, use - `@SuppressWarnings("fallthrough")` to suppress warnings for that method.

## Strings

- Java strings are sequences of `char` values.
- The JVM may store strings as byte sequences for single-byte characters and as char sequences for others, rather than always using char[].

- __Strings are immutable__ - 
  - You cannot change a character inside an existing string.
  - Immutable strings allow sharing, so the compiler can store strings in a common pool and multiple variables can reference the same characters without copying.

- __Concatenation__ -
  - Use `+` to concatenate two strings.
  - When you concatenate a string with a non-string value, Java converts the non-string value to a string.
  - Example -

  ```
  int age = 15;
  String rating = "PG" + age;           // "PG15"
  ```

> [!WARNING]
> String concatenation uses + and is evaluated left to right. Therefore, if you concatenate a number after a string, everything becomes a string from that point onward.
> Example -
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

- __`String` API__ -

| __Operation__                                             | __Description__                                                                |
| --------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `charAt(int index)`                                       | Returns the code unit at the specified index                                   |
| `length()`                                                | Returns the number of code units in the string                                 |
| `equals(Object other)`                                    | Returns `true` if the string equals `other`                                    |
| `equalsIgnoreCase(String other)`                          | Returns `true` if strings match ignoring case                                  |
| `compareTo(String other)`                                 | Negative if before, positive if after, 0 if equal                              |
| `isEmpty()`                                               | Returns `true` if the string is empty                                          |
| `isBlank()`                                               | Returns `true` if the string is empty or whitespace                            |
| `startsWith(String prefix)`                               | Returns `true` if the string starts with `prefix`                              |
| `endsWith(String suffix)`                                 | Returns `true` if the string ends with `suffix`                                |
| `indexOf(String str)`                                     | Index of first occurrence of `str`, or -1                                      |
| `indexOf(String str, int fromIndex)`                      | First occurrence starting from `fromIndex`, or -1                              |
| `indexOf(String str, int fromIndex, int toIndex)`         | First occurrence between `fromIndex` and `toIndex`, or -1                      |
| `lastIndexOf(String str)`                                 | Index of last occurrence of `str`, or -1                                       |
| `lastIndexOf(String str, int fromIndex)`                  | Last occurrence up to `fromIndex`, or -1                                       |
| `replace(CharSequence oldString, CharSequence newString)` | Returns a new string replacing all occurrences of `oldString` with `newString` |
| `substring(int beginIndex)`                               | Returns substring from `beginIndex` to end                                     |
| `substring(int beginIndex, int endIndex)`                 | Returns substring from `beginIndex` to `endIndex - 1`                          |
| `toLowerCase()`                                           | Returns a new string in lowercase                                              |
| `toUpperCase()`                                           | Returns a new string in uppercase                                              |
| `strip()`                                                 | Removes leading and trailing whitespace                                        |
| `stripLeading()`                                          | Removes leading whitespace                                                     |
| `stripTrailing()`                                         | Removes trailing whitespace                                                    |
| `join(CharSequence delimiter, CharSequence... elements)`  | Joins elements into a string separated by `delimiter`                          |
| `repeat(int count)`                                       | Returns a string repeated `count` times                                        |

> [!WARNING]
> Do not use the `==` operator to test whether two strings are equal! It only determines whether or not the strings are stored in the same location. 
>
> Only string literals are shared, not strings that are computed at runtime. Therefore, never use `==` to compare strings. Always use `equals` instead.

> [!TIP]
> `CharSequence` is the interface type to which all strings belong. 

### `StringBuilder`

- String concatenation is inefficient because each concatenation creates a new String object.
- `StringBuilder` avoids this by building a string in a _mutable buffer_.

- The `String` class doesn’t have a method to reverse the Unicode characters of a string, but `StringBuilder` does - 
```
String reversed = new StringBuilder(original).reverse().toString();
```

- __`StringBuilder` API__ -

| __Operation__                          | __Description__                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------- |
| `StringBuilder()`                      | Creates an empty StringBuilder                                               |
| `StringBuilder(CharSequence seq)`      | Creates a StringBuilder initialized with `seq`                               |
| `length()`                             | Returns the number of code units in the builder                              |
| `append(String str)`                   | Appends `str` and returns the builder                                        |
| `appendCodePoint(int cp)`              | Appends a Unicode code point and returns the builder                         |
| `insert(int offset, String str)`       | Inserts `str` at `offset` and returns the builder                            |
| `delete(int startIndex, int endIndex)` | Deletes characters from `startIndex` to `endIndex-1` and returns the builder |
| `repeat(CharSequence cs, int count)`   | Appends `count` copies of `cs` and returns the builder                       |
| `reverse()`                            | Reverses the code points in the builder and returns the builder              |
| `toString()`                           | Converts the builder contents to a `String`                                  |

## Input and Output

- `IO.readln()` reads one line of input and returns it as a String.
- You can pass a prompt string -
```
String name = IO.readln("What is your name? ");
```

- To read an integer or double, convert the input string -
```
int age = Integer.parseInt(IO.readln("How old are you? "));
double rate = Double.parseDouble(IO.readln("Interest rate: "));
```

> [!TIP]
> The `IO.readLine` method is not suitable for reading a password from a console since the input is plainly visible to anyone. Use the `readPassword` method of the `Console` class to read a password while hiding the user input -
> ```
> char[] passwd = System.console().readPassword("Password: ");
> Arrays.fill(passwd, '*');       // overwrite with *'s immediately after use
> ```

### `java.lang.IO` API

| __Method__                     | __Description__                                              | __Output Behavior__                     |
| ------------------------------ | ------------------------------------------------------------ | --------------------------------------- |
| `println(Object obj)`          | Converts the object to a string and prints it on the console | Prints followed by a __line separator__ |
| `print(Object obj)`            | Converts the object to a string and prints it on the console | Prints __without__ a line separator     |
| `println()`                    | Prints a line separator                                      | Prints __only a new line__              |
| `String readln(String prompt)` | Prints a prompt on the console and waits for user input      | Returns __one line of input__           |
| `String readln()`              | Waits for user input without printing a prompt               | Returns __one line of input__           |

### `java.lang.System` API

| __Method__  | __Signature__              | __Description__                                                                                                                                            |
| ----------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `console()` | `static Console console()` | Returns a `Console` object for interacting with the user through a console window if available. Returns __`null`__ if console interaction is not possible. |


### `java.io.Console` API

| __Method__     | __Signature__                                        | __Description__                                                     |
| -------------- | ---------------------------------------------------- | ------------------------------------------------------------------- |
| `readPassword` | `char[] readPassword(String prompt, Object... args)` | Displays the prompt and reads a password __without echoing__ input. |
| `readLine`     | `String readLine(String prompt, Object... args)`     | Displays the prompt and reads user input until the end of the line. |

### Formatting Output

- `IO.print(x)` prints a number with the maximum non-zero digits.
- Example -
```
double x = 10000.0 / 3.0;
IO.print(x);                  // 3333.3333333333335
```

- Using `formatted` method -
  - Uses C-style formatting - `IO.print("%8.2f".formatted(x));`
  - `%8.2f` means -
    - `8` → total width of the field
    - `2` → digits after decimal point
  - Output - ` 3333.33`

- Formatting with Multiple Arguments -
  ```
  IO.print("Hello, %s. Next year, you'll be %d.".formatted(name, age + 1));
  ```

  - `%s` → string placeholder
  - `%d` → integer placeholder

- __Conversions for `formatted`__ -

| **Conversion Character** | **Type**                                        | **Example**  |
| ------------------------ | ----------------------------------------------- | ------------ |
| `d`                      | Decimal integer                                 | `159`        |
| `x` or `X`               | Hexadecimal integer                             | —            |
| `o`                      | Octal integer                                   | `237`        |
| `f` or `F`               | Fixed-point floating-point                      | `15.9`       |
| `e` or `E`               | Exponential floating-point                      | `1.59e+01`   |
| `g` or `G`               | General floating-point (shorter of `e` and `f`) | —            |
| `a` or `A`               | Hexadecimal floating-point                      | `0x1.fccdp3` |
| `s` or `S`               | String                                          | `Hello`      |
| `c` or `C`               | Character                                       | `H`          |
| `b` or `B`               | Boolean                                         | `true`       |
| `h` or `H`               | Hash code                                       | `42628b2`    |
| `t` `x` or `T`           | Legacy date/time formatting                     | —            |
| `%`                      | The percent symbol                              | `%`          |
| `n`                      | Platform-dependent line separator               | —            |

> [!TIP]
> You can use the `%s` conversion to format any object. If it implements `Formattable` interface, its `formatTo` method is used, otherwise `toString()` is used.

- Also, you can specify _flags_ that control the appearance of the formatted output, eg - the comma flag adds group separators i.e. `IO.println("%,.2f".formatted(10000.0 / 3.0));` prints `3,333.33`.

- You can use multiple flags, for example `"%,(.2f"` to use group separators and enclose negative numbers in parentheses.

- __Flags for `printf`__ -

| **Flag**                    | **Purpose**                                    | **Example** |
| --------------------------- | ---------------------------------------------- | ----------- |
| `+`                         | Prints sign for positive and negative numbers  | `+3333.33`  |
| (space)                     | Adds a space before positive numbers           | ` 3333.33`  |
| `0`                         | Adds leading zeros                             | `003333.33` |
| `-`                         | Left-justifies the field                       | `3333.33 `  |
| `(`                         | Encloses negative numbers in parentheses       | `(3333.33)` |
| `,`                         | Adds group separators                          | `3,333.33`  |
| `#` (for `f` format)        | Always includes a decimal point                | `3,333.`    |
| `#` (for `x` or `o` format) | Adds `0x` or `0` prefix                        | `0xcafe`    |
| `$`                         | Specifies argument index (positional argument) | `159 9F`    |
| `<`                         | Reuses the previous argument                   | `159 9F`    |

> [!TIP]
> Formatting depends on the system locale (e.g., Germany uses `,` instead of `.`). To ensure consistent output for files, specify a fixed locale -
> ```
> IO.print(String.format(Locale.US, "%8.2f", x));
> ```

## Control Flow

### If-else
```
if (condition1) {
  statement1
} else if (condition2) {
  statement2
} else {
  statement3
}
```

### While loops
```
while (condition) {
  statement
}
```

- __do while loops__ -
```
do {
  statement
} while (condition)
```

### for loops
  
- Syntax -
```
for (initialization; condition; update) {
  statement
}
```
  
- Example -
```
for (int i = 1; i <= 10; i++) {
  statement
}
  ```

> [!TIP]
> Java lets you put almost anything in those three parts, but a good programming practice is to use the for loop only to control a single counter variable. Otherwise the code becomes confusing and hard to read, eg - 
> ```
> for (int i = 0; i < 10; System.out.println(i++)) { ... }
> ```

- Initialization can declare multiple variables, provided they are of the same type and the update expression can contain multiple comma-separated expressions -
```
for (int i = 1, j = 10; i <= 10; i++, j--) { . . . }
```

### Labeled `break`

- The labeled `break` statement lets you break out of multiple nested loops.
- Example with `while` loop -
```
int n;

read_data:
while ( ... ) {                       // this loop statement is tagged with the label
    ...
    for ( ... ) {                     // this inner loop is not labeled
        if (n < 0) {                  
            break read_data;          // break out of read_data loop
        }
        ...
    }
}

// this statement is executed immediately after the labeled break
statement
```

- Example with `if` statement -
```
label: {
    ...
    if (condition) break label;       // exits block
    ...
}
// jumps here when the break statement executes
```

### `continue` statement

- The `continue` statement transfers control to the header of the innermost enclosing loop.
- Example with `while` -
```
while (sum < goal) {
    n = Integer.parseInt(IO.readln("Enter a number: "));
    if (n < 0) continue;
    sum += n;                         // not executed if n < 0
}
```

- Example with `for` - jumps to the “update” part of the for loop -
```
for (count = 1; count <= 100; count++) {
    n = Integer.parseInt(IO.readln("Enter a number, -1 to quit: "));
    if (n < 0) continue;
    sum += n;                         // not executed if n < 0 - jumps to the count++ statement
}
```

> [!TIP]
> There is also a labeled form of the `continue` statement that jumps to the header of the loop with the matching label.

## Big Numbers

- `java.math.BigInteger` -
  - Used for very large integer arithmetic.
  - Can hold numbers with any number of digits.
  - From a normal number - `BigInteger.valueOf(100)`
  - From a long number (string) - `new BigInteger("123456")`
  - Constants - `BigInteger.ZERO`, `BigInteger.ONE`, `BigInteger.TWO`, `BigInteger.TEN`

- `java.math.BigDecimal` -
  - Used for very precise decimal numbers (money, financial calculations, etc.)
  - Always construct from integers or string.
  - Avoid `new BigDecimal(0.1)` - because it creates an imprecise value `0.1000000000000000055511151231257827021181583404541015625`

- Cannot use arithmetic operators like `+`, `-`, `*`, `/` - because Java does not support operator overloading.

> [!NOTE]
> Only exception - `+` is overloaded for string concatenation.

- Instead use methods -
```
BigInteger c = a.add(b);  // c = a + b
BigInteger d = c.multiply(b.add(BigInteger.valueOf(2)));      // d = c * (b + 2)
```

> [!TIP]
> Java 19 feature - `parallelMultiply()` - works like `multiply()` but may be faster using multiple CPU cores.

### `java.math.BigInteger` API

| Method                                  | Description                                    | Returns                                             |
| --------------------------------------- | ---------------------------------------------- | --------------------------------------------------- |
| `BigInteger add(BigInteger other)`      | Adds this BigInteger with `other`              | Sum                                                 |
| `BigInteger subtract(BigInteger other)` | Subtracts `other` from this BigInteger         | Difference                                          |
| `BigInteger multiply(BigInteger other)` | Multiplies this BigInteger with `other`        | Product                                             |
| `BigInteger divide(BigInteger other)`   | Divides this BigInteger by `other`             | Quotient                                            |
| `BigInteger mod(BigInteger other)`      | Computes remainder of division                 | Remainder                                           |
| `BigInteger pow(int exponent)`          | Raises this BigInteger to the power `exponent` | Power                                               |
| `BigInteger sqrt()`                     | Computes the square root                       | Square root                                         |
| `int compareTo(BigInteger other)`       | Compares this BigInteger with `other`          | `0` if equal, negative if less, positive if greater |
| `static BigInteger valueOf(long x)`     | Converts a long to BigInteger                  | BigInteger value of `x`                             |

### `java.math.BigDecimal` API

| Method                                                   | Description                                    | Returns                                             |
| -------------------------------------------------------- | ---------------------------------------------- | --------------------------------------------------- |
| `BigDecimal(String digits)`                              | Constructs a BigDecimal from the given digits  | BigDecimal                                          |
| `BigDecimal add(BigDecimal other)`                       | Adds this BigDecimal with `other`              | Sum                                                 |
| `BigDecimal subtract(BigDecimal other)`                  | Subtracts `other` from this BigDecimal         | Difference                                          |
| `BigDecimal multiply(BigDecimal other)`                  | Multiplies this BigDecimal with `other`        | Product                                             |
| `BigDecimal divide(BigDecimal other)`                    | Divides this BigDecimal by `other`             | Quotient (exact)                                    |
| `BigDecimal divide(BigDecimal other, RoundingMode mode)` | Divides with rounding using the specified mode | Rounded quotient                                    |
| `int compareTo(BigDecimal other)`                        | Compares this BigDecimal with `other`          | `0` if equal, negative if less, positive if greater |

> [!NOTE]
> Division -
>   - `divide(other)` - throws exception if result is not finite (repeating decimal).
>   - `divide(other, mode)` - allows rounding when the result is repeating.
>
> Example - `RoundingMode.HALF_UP`
>   - Round down - 0–4
>   - Round up - 5–9

## Arrays

- Declaring an array - 
  - Syntax - `elementType[] arrayName;`
  - Example - `int[] a;`
  - This only declares the variable, it does not create the array.

- Initialize an array -
  ```
  int[] a = new int[100];
  // or
  var a = new int[100];
  ```

  - This creates an array of 100 integers, all initialized to 0.
  - Array length does not have to be a constant - `var a = new int[n];`

- Array variables store a reference, not the array itself.
- Once created, the array size cannot be changed.
- To access an array element by its index - `a[i]`
- To modify individual elements - `a[0] = 10;`
- Length of the array - `a.length`
- For resizable collections, use `ArrayList` instead.

- Two valid ways to declare an array -
  - `int[] a` - preferred
  - `int a[]` - valid, but less common

- You can declare and initialize an array in one step and the array size is inferred automatically -
```
int[] smallPrimes = { 2, 3, 5, 7, 11, 13 };
```

- Creating an array of length 0 -
```
new elementType[0]
// or
new elementType[] {}
```

- Use `Arrays.toString(a)` to convert an array to a string and print it -
  ```
  int[] a = {2, 3, 5, 7, 11, 13};
  IO.println(Arrays.toString(a));
  ```

  - Output - `[2, 3, 5, 7, 11, 13]`

### Array Copying

- Assigning one array variable to another copies the reference, not the values i.e. both variables point to the same array in memory -
```
int[] luckyNumbers = smallPrimes;
luckyNumbers[5] = 12; // smallPrimes[5] is also 12
```

- To create a new array with the same values, use `Arrays.copyOf` -
  ```
  int[] copiedLuckyNumbers = Arrays.copyOf(luckyNumbers, luckyNumbers.length);
  ```

  - The second argument is the length of the new array.

- Can also use `copyOf` to resize an array -
```
luckyNumbers = Arrays.copyOf(luckyNumbers, 2 * luckyNumbers.length);
```

- Extra elements are filled with -
  - `0` for numeric arrays
  - `false` for boolean arrays
  - `null` for object references
- If new length < original length then only initial values are copied.

- Memory model -
  - Java arrays are heap-allocated.
  - Think of Java arrays like pointers to heap memory.

- To sort an array - `Arrays.sort(a)` - uses a tuned version of the QuickSort algorithm.

### `java.util.Arrays` API

| Method                                              | Description                                                                                                      | Notes                                                                                   |
| --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `static String toString(T[] a)`                     | Returns a string with the elements of `a`, enclosed in brackets and separated by commas.                         | `T` can be `int, long, short, char, byte, boolean, float, double`.                      |
| `static T[] copyOf(T[] a, int end)`                 | Returns a new array of the same type as `a`, length = `end`, filled with values from `a`.                        | If `end > a.length`, extra elements are padded with `0` (numbers) or `false` (boolean). |
| `static T[] copyOfRange(T[] a, int start, int end)` | Returns a new array of same type as `a`, length = `end–start`, filled with values from `a[start]` to `a[end-1]`. | If `end > a.length`, extra elements padded with `0` or `false`.                         |
| `static void sort(T[] a)`                           | Sorts the array using a tuned QuickSort algorithm.                                                               | —                                                                                       |
| `static void fill(T[] a, T v)`                      | Sets all elements of the array to value `v`.                                                                     | —                                                                                       |
| `static boolean equals(T[] a, T[] b)`               | Returns `true` if arrays `a` and `b` have the same length and corresponding elements match.                      | —                                                                                       |

## Multi-dimensional Arrays

- Declaring a 2D Array - `double[][] balances`
- Initializing a 2D Array -
  - Fixed size initialization - `balances = new double[NYEARS][NRATES]`
  - Shorthand initialization (if elements known) - 
  ```
  int[][] magicSquare = {
    {16, 3, 2, 13},
    {5, 10, 11, 8},
    {9, 6, 7, 12},
    {4, 15, 14, 1}
  };
  ```

- Accessing Elements - `balances[i][j]`
- Printing a 2D Array - `Arrays.deepToString(a)`

### Ragged Arrays (Arrays of Arrays)

- Java has no multidimensional arrays at all, only one-dimensional arrays. 
- Multidimensional arrays are faked as “arrays of arrays.”
- It is legal to construct multi-dimensional arrays where a dimension is zero, eg -
```
new int[3][0]             // 3 rows - each having length 0
new int[0][3]             // no rows
```

> [!TIP]
> In Java, each row is stored separately on the heap.

## for-each loop

- Syntax - `for (variable : collection) statement`
-  The collection expression must be an `array` or an object of a class that implements the `Iterable` interface, such as `ArrayList`.
- Example -
```
for (int element : a)
    IO.println(element);
```


