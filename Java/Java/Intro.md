## Java

- Hello World Program -

```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

- Where -

  - `public class HelloWorld` - In Java, all code is defined inside classes.
  - `public static void main(String[] args)` - main method is the first method that is called when the program runs. `static` indicates that the method does not operate on any objects and `void` indicates that it does not return any value.
  - `System.out.println("Hello, World!");` - print "Hello, World!" to the console.

- Compiling - `javac` command compiles the Java source code into an intermediate machine-independent representation, called `byte codes`, and saves them in class files.
- Running - `java` command launches a _virtual machine_ that loads the class files and executes the byte codes.

> [!NOTE]
> Java supports **Write Once, Run Anywhere (WORA)** i.e. once compiled, byte codes can run on any Java virtual machine.

> [!TIP]
> If your program consists of a single source file, then you can skip the compilation step and run the program directly. Behind the scenes, the program is compiled before it runs, but no class files are produced.

> [!TIP]
> On Unix-like operating systems, you can turn a Java file into an executable program -
>
> - Remove `.java` extension from the file - `mv HelloWorld.java hello`
> - Make the file executable - `chmod +x hello`
> - Add a “shebang” line at the top of the file - `#!/path/to/jdk/bin/java --source 21`
> - Run the program - `./hello`

- In Java 21 _preview_, the Hello World program can be written as -

```
class HelloWorld {
    void main() {
        System.out.println("Hello, World!");
    }
}
```

- To run - `java --enable-preview --source 21 HelloWorld.java`

- We can even omit the class if there is only one -

```
void main() {
   System.out.println("Hello, World!");
}
```

## JShell

- Provides a **read-evaluate-print loop (REPL)** that allows you to experiment with Java code without compiling and running a program.
- To start JShell - `jshell`.
- Supports tab completion.
- To get import suggestions, type the class name (eg - `Duration`) followed by `Shift`+`Tab`+`I`.

- Comments -
  - Single line comments - `//`
  - Multi-line comments - `/* comments */`
  - Documentation comments - `/** comments */`

## Primitive Types

### Signed Integer

- Java provides the four signed integer types -
  | Type   | Storage Requirement | Range (inclusive)                                       |
  |--------|---------------------|---------------------------------------------------------|
  | `byte` | 1 byte              | -128 to 127                                             |
  | `short`  | 2 bytes           | –32,768 to 32,767                                       |
  | `int`    | 4 bytes           | –2,147,483,648 to 2,147,483,647 (just over 2 billion)   |  
  | `long`   | 8 bytes           | –9,223,372,036,854,775,808 to 9,223,372,036,854,775,807 |

- `long` integers are written with suffix `L` eg - `10L`, but there is no syntax for literals of type `byte` or `short`. Use the cast notation instead - `(byte) 127`.
- Hexadecimal literals have a prefix `0x`, eg - `0xCAFEBABE`. 
- Binary values have a prefix `0b`, eg - `0b1001` is 9.

> [!WARNING]
> Octal numbers have a prefix `0`, eg - `011` is 9. This can be confusing, so it is better to stay away from octal literals and leading zeroes.

- You can add underscores to number literals, such as `1_000_000` (or `0b1111_0100_0010_0100_0000`) to denote one million. The Java compiler simply removes them.
- The constants `Integer.MIN_VALUE` and `Integer.MAX_VALUE` are the smallest and largest int values. The Long, Short, and Byte classes also have `MIN_VALUE` and `MAX_VALUE` constants.

> [!TIP]
> If the `long` type is not sufficient, use the `BigInteger` class.

> [!TIP]
> In Java, the ranges of the integer types do not depend on the machine on which you will be running your program, whereas the integer types in C and C++ programs depend on the processor for which a program is compiled.

- `Byte.toUnsignedInt(b)` returns an int value between 0 and 255. This can be useful if you want to work with integer values that can never be negative and you really need an additional bit.
- The `Integer` and `Long` classes have methods for unsigned division and remainder.


### Floating-Point Types

- Java supports two floating-point types -
  | Type     | Storage Requirement | Range (inclusive)                                   |
  |----------|---------------------|-----------------------------------------------------|
  | `float`  | 4 byte              | Approx ±3.402E+38F (6–7 significant decimal digits) |
  | `double` | 8 bytes             | Approx ±1.797E+308 (15 significant decimal digits)  |

- `Double` is the default type. You can optionally suffix with `D`, eg - `3.14D`. To define a float, use suffix `F`, eg - `3.14F`.

- Special floating-point values -
  - `Double.POSITIVE_INFINITY` = `∞`
  - `Double.NEGATIVE_INFINITY` = `–∞`
  - `Double.NaN` - _not a number_

> [!WARNING]
> All “not a number” values are considered to be distinct from each other. Therefore, `x == Double.NaN` will always return `false`.
> Instead, use - `Double.isNaN(x)`

> [!TIP]
> `Double.isInfinite` tests for `± ∞` and `Double.isFinite` to check that a floating-point number is neither infinite nor a `NaN`.

- Floating-point numbers are not suitable for financial calculations in which roundoff errors cannot be tolerated, eg - `System.out.println(2.0 - 1.7)` prints `0.30000000000000004`. There is no precise binary representation of the fraction `3/10`.

> [!TIP]
> If you need precise numerical computations with arbitrary precision and without roundoff errors, use the `BigDecimal` class.

### The `char` Type

- Describes “code units” in the UTF-16 character encoding used by Java.
- Eg - `'J'` is a character literal with value `74` (or hexadecimal `4A`)
- A code unit can be expressed in hexadecimal, with the `\u` prefix, eg - `'\u004A'` is the same as `'J'`.
- Special codes - 
  - `\n` - new line
  - `\r` - carriage return
  - `\t` - tab
  - `\b` - backspace
- Use a backslash to escape a single quote `'\''` and a backslash `'\\'`.

### The `boolean` Type

- Has two values - `false` and `true`.
- The `boolean` type is not a number type. There is no relationship between `boolean` values and the integers `0` and `1`.

## Variables

- Defining a variable - `int count = 0;`
- Defining multiple variables of same type in single line - `int total = 0, count;` - where `count` is uninitialized variable.
- Can also be initialized as - `var random = new Random();`. The type of the variable is the type of the expression with which the variable is initialized.

> [!TIP]
> A variable must be initialized before use, eg -
> ```
> int count;
> count++;        // Error—count might not be initialized
> ```
> or
> ```
> int count;
> if (total == 0) {
>     count = 0;
> } else {
>     count++;    // Error—count might not be initialized
> }
> ```

> [!TIP]
> It is considered good style to declare a variable as late as possible, just before you need it for the first time.

### Identifier
- The name of a variable, method, or class is called an **identifier**.
- Must begin with a letter. 
- Can consist of any letters, digits, currency symbols, and “punctuation connectors” such as `_` and a few other underscore-like characters such as the wavy low line `﹏`.
- Letters and digits can be from any alphabet, not just the Latin alphabet, eg - `π` and `élévation` are valid identifiers.

## Constants

- `final` keyword denotes a value that cannot be changed once it has been assigned.
```
final int DAYS_PER_WEEK = 7;
```

- By convention, uppercase letters are used for names of constants.
- Declare a constant outside a method, using the `static` keyword to be used as - `Calendar.DAYS_PER_WEEK`

> [!TIP]
> The `System` class declares a constant - `public static final PrintStream out` that you can use anywhere as `System.out`

- We can defer the initialization of a final variable, but it has to be initialized exactly once before it is used for the first time -
```
final int DAYS_IN_FEBRUARY;
if (leapYear) {
    DAYS_IN_FEBRUARY = 29;
} else {
    DAYS_IN_FEBRUARY = 28;
}
```

## Operators

- `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `<<=`, `>>=`, `>>>=`, `&=`, `^=`, `|=` operators are right-associative i.e. `i -= j -= k` means `i -= (j -= k)`.

- In `/` operator, if both operands are integer types, it denotes integer division, discarding the remainder, eg - `17 / 5` is `3`, whereas `17.0 / 5` is `3.4`.

- `&` and `|` when applied to boolean values, force evaluation of both operands before combining the results.

> [!TIP]
> An integer division by zero gives rise to an exception which, if not caught, will terminate your program.
> A floating-point division by zero yields an infinite value or `NaN`, without causing an exception.

> [!TIP]
> The `Math` class provides several methods to make integer arithmetic safer. The mathematical operators quietly return wrong results when a computation overflows, eg - `1000000000 * 3` evaluates to `-1294967296` because the largest int value is just over two billion. 
> If you call `Math.multiplyExact(1000000000, 3)` instead, an exception is generated. You can catch that exception or let the program terminate rather than quietly continue with a wrong result. 
> There are also methods `addExact`, `subtractExact`, `incrementExact`, `decrementExact`, `negateExact`, all with `int` and `long` parameters.

> [!WARNING]
> If you worry that a cast can silently throw away important parts of a number, use the `Math.toIntExact` method instead which throws an exception if the number cannot be cast.

- **Conditional operator** - `time < 12 ? "am" : "pm"` - yields the string `"am"` if `time < 12` and the string `"pm"` otherwise.

## `BigInteger` & `BigDecimal`

- The `java.math.BigInteger` class implements arbitrary-precision integer arithmetic, and `java.math.BigDecimal` does the same for floating-point numbers.
- Convert `long` to `BigInteger` - `BigInteger.valueOf(876543210123456789L)`
- Convert a string of digits to `BigInteger` - `new BigInteger("9876543210123456789")`
- Java does not permit the use of operators with objects, so you must use method calls to work with big numbers -
```
BigInteger.valueOf(5).multiply(n.add(k)); // r = 5 * (n + k)
```

- The `parallelMultiply` method yields the same result as `multiply` but can potentially compute the result faster by using multiple processor cores.

- `BigDecimal.valueOf(n, e)` returns a BigDecimal instance with value `n × 10–e`.

- The result of the floating-point subtraction `2.0 - 1.7` is `0.30000000000000004`. The BigDecimal class can compute the result accurately -
```
BigDecimal.valueOf(2, 0).subtract(BigDecimal.valueOf(17, 1))    // Exactly equal to 0.3
```

## Strings

- A string is a sequence of characters. 
- **Immutable**
- To concatenate two strings - `+`
- To combine several strings, separated with a delimiter, use the `join` method -
```
String.join(" | ", "Peter", "Paul", "Mary")   // returns "Peter | Paul | Mary"
```

- It is inefficient to concatenate a large number of strings if all you need is the final result. In that case, use a `StringBuilder` instead -
```
var builder = new StringBuilder();
while (<more_strings>) {
    builder.append(<next_string>);
}
String result = builder.toString();
```

- To extract all substrings from a string that are separated by a delimiter -
```
String names = "Peter, Paul, Mary";
String[] result = names.split(", ");  // returns an array of three strings ["Peter", "Paul", "Mary"]
```

> [!TIP]
> The separator can be any regular expression, eg - `input.split("\\s+")` splits input at white space.

- `"Hello World!".substring(7, 12)` - slices the string from index 0 (inclusive) to 12 (exclusive), therefore returning `World`. Note that `12-7` returns the length of the substring, that's why the ending index is exclusive.

- String equality - `"Hello".equals("Hello")` - returns True.

- The `==` (String object equality) operator can be used for the `null` checks, eg -
```
String middleName = null;
middleName == null          // true
```

> [!WARNING]
> Never use the `==` operator to compare strings. The comparison returns true only if `location` and `"World"` are the same object in memory. In the virtual machine, there is only one instance of each literal string, so `"World" == "World"` will be `true`.
> But if location was computed like - `String location = "Hello World!".substring(7, 12)` - then the result is placed into a separate `String` object, and the comparison `location == "World"` will return `false`.

> [!TIP]
> When comparing a string against a literal string, it is a good idea to put the literal string first -
> ```
> "World".equals(location)
> ```
> This test works correctly even when location is null.

- `equalsIgnoreCase` for String equality ignoring the cases.

- `str1.compare(str2)` tells whether `str1` comes before `str2` in dictionary order. It returns -
  - a negative integer (not necessarily -1) if first comes before second
  - a positive integer (not necessarily 1) if first comes after second
  - 0 if they are equal.

> [!TIP]
> The strings are compared a character at a time, until one of them runs out of characters or a mismatch is found. For eg - when comparing `"word"` and `"world"`, the first three characters match. Since `d` has a Unicode value that is less than that of `l`, `"word"` comes first. 
> The call `"word".compareTo("world")` returns `-8`, the difference between the Unicode values of `d` and `l`.

> [!TIP]
> When sorting human-readable strings, use a `Collator` object that knows about language-specific sorting rules -
> ```
> Collator.getInstance(Locale.GERMAN).compare("café", "caffeine")
> ```

## Number to String Conversions

- Integer to String conversion - `Integer.toString(42)` - returns `"42"`.
- With radix - `Integer.toString(42, 2)` - returns `101010`.
- String to Integer coversion - `Integer.parseInt("10")` - returns `10`.
- With radix - `Integer.parseInt("101010", 2)` - returns `42`.
- For floating-point numbers, use `Double.toString` and `Double.parseDouble`.

> [!NOTE]
> `CharSequence` is a common supertype of `String`, `StringBuilder`, and other sequences of characters.




