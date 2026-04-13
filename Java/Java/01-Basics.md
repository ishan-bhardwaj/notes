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

| Text                                                          | Sub-Part     | Meaning                                                                                                             |
| ------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------- |
| **`java 25.0.1 2025-10-21 LTS`**                              |              | Java version information (as printed by a specific JDK build)                                                       |
|                                                               | `25`         | Major Java feature release (Java 25)                                                                                |
|                                                               | `0.1`        | Update / patch level of Java 25                                                                                     |
|                                                               | `2025-10-21` | Vendor build/release date *(not part of official Java versioning spec)*                                             |
|                                                               | `LTS`        | Long-Term Support designation for this release line (vendor-defined)                                                |
| **`Java(TM) SE Runtime Environment (build 25.0.1+8-LTS-27)`** |              | Java runtime environment (JRE) build information                                                                    |
|                                                               | `Java SE`    | Java Standard Edition runtime                                                                                       |
|                                                               | `25.0.1+8`   | Java version `25.0.1` with build number `+8`                                                                        |
|                                                               | `LTS-27`     | Vendor-specific internal packaging/build identifier (not a standard Java version field; does not define LTS itself) |
| **`Java HotSpot(TM) 64-Bit Server VM`**                       |              | JVM implementation details                                                                                          |
|                                                               | `HotSpot`    | JVM implementation used in OpenJDK ecosystem (originally Sun/Oracle, now widely used across vendors)                |
|                                                               | `64-Bit`     | JVM running in 64-bit architecture mode                                                                             |
|                                                               | `Server VM`  | JVM mode optimized for long-running server/backend workloads with aggressive JIT optimizations                      |
| **`mixed mode, sharing`**                                     |              | JVM execution and runtime optimizations                                                                             |
|                                                               | `mixed mode` | Execution uses interpreter + tiered JIT compilation (adaptive optimization based on runtime profiling)              |
|                                                               | `sharing`    | Class Data Sharing (CDS) enabled: preloaded/shared class metadata to reduce startup time and memory usage           |

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

> [!TIP]
> Check which Unicode characters are allowed in Java identifiers - `Character.isJavaIdentifierStart` and `Character.isJavaIdentifierPart`

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

> [!TIP]
> Use `System.console().readPassword` to read a password which hiding the user input -
> ```
> char[] passwd = System.console().readPassword("Password: ");
> Arrays.fill(passwd, '*');                     // overwrite with *'s immediately after use
> ```

- Formatting output -
  ```
  IO.print("%8.2f".formatted(x));                                             // 3333.33
  IO.print("Hello, %s. Next year, you'll be %d.".formatted(name, age + 1));   // multiple arguments
  ```

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
> You can use the `%s` conversion to format any object. 
> 
> If it implements `Formattable` interface, its `formatTo` method is used, otherwise `toString()` is used.

- Specify flags - 
  - The comma flag adds group separators, eg - 
    ```
    IO.println("%,.2f".formatted(10000.0 / 3.0));           // prints 3,333.33
    ```

  - Use multiple flags, eg - `"%,(.2f"` uses group separators and enclose negative numbers in parentheses.

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
> Formatting depends on the system locale (e.g., Germany uses `,` instead of `.`). 
> 
> Specifying a fixed locale -
> ```
> IO.print(String.format(Locale.US, "%8.2f", x));
> ```
