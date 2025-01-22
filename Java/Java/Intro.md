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
