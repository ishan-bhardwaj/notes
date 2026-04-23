# Packages

- Java allows you to group related classes into a package.
- Packages serve three main purposes -
  - Organization of large programs
  - Name uniqueness
  - Encapsulation at a higher level than classe

- Encapsulation with Packages -
  - Packages provide default encapsulation.
  - Only public classes can be accessed from other packages.
  - Only public methods and fields are callable from other packages.

- __Access Rules__ -

| Modifier        | Accessible From                     |
| --------------- | ----------------------------------- |
| `public`        | Anywhere                            |
| `private`       | Same class only                     |
| _(no modifier)_ | Same package only (package-private) |

>[!NOTE] 
> Packages are not hierarchical i.e. `java.util` and `java.util.random` are unrelated.

- A class can use -
  - All classes in its own package
  - All public classes from other packages
  
- Importing classes -
  - Using fully qualified names -
    - `java.time.LocalDate today = java.time.LocalDate.now()`
    - Always works, but tedious.
  - Using `import` -
    - Example -
    ```
    import java.time.*;
    LocalDate today = LocalDate.now();

    // or
    import java.time.LocalDate;
    ```

> [!TIP]
> `java.lang`is always imported automatically.

> [!TIP]
> `import` is purely for convenience. The bytecode always uses fully qualified names.

- __Module Imports__ (Java 25+) -
  - Java packages can be organized into modules.
  - You can import all packages in a module -
    - Example - `import module java.xml`
    - Important module - 
      - `import module java.base`
      - Automatically imported in compact source files.
      - Includes `java.lang`, `java.util`, `java.time`, `java.io`, and many more.

- __Static Imports__ -
  - Allows importing static methods and fields.
  - Example -
  ```
  import static java.lang.System.*;
  err.println("Error");
  exit(0);
  ```

- If you don’t put a package statement in the source file, then the classes in that source file belong to the unnamed package. The unnamed package has no package name.

> [!TIP]
> The implicitly declared class of a compact compilation unit (with methods declared outside a class) is always in the unnamed package.

  - Good Use Case -
  ```
  import static java.lang.Math.*;
  sqrt(pow(x, 2) + pow(y, 2))
  ```

  - Improves readability for math-heavy code.

  - Enum Constants -
  ```
  import java.time.DayOfWeek;
  import static java.time.DayOfWeek.*;

  DayOfWeek d = FRIDAY;
  ```

- The compiler does not check the directory structure when it compiles source files. 
  - For eg - suppose you have a source file that starts with the directive - `package com.mycompany`.
  - You can compile the file even if it is not contained in a subdirectory `com/mycompany`. 
  - The source file will compile without errors if it doesn’t depend on other packages. 
  - However, the resulting program will not run unless you first move all class files to the right place. 
  - The virtual machine won’t find the classes if the packages don’t match the directories.

> [!NOTE]
> Variables with no access modifiers defaults to package access.

> [!NOTE]
> A source file can only contain one public class, and the names of the file and the public class must match.
