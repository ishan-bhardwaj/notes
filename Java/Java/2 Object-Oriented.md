# Object-oriented Programming

- __Encapsulation / Information Hiding__ - Encapsulation is the concept of combining data (attributes) and behavior (methods) into a single unit — the object — while hiding the internal implementation details from the outside world.
- __Key Principle__ - Methods should never directly access the instance fields of other objects. Instead, interaction with an object’s data should happen only through its public methods.

- Three fundamental characteristics of objects -
  - Behavior - An object’s behavior defines what operations can be performed on it. This is represented by the methods that can be invoked on the object.
  - State - An object’s state represents the data it holds at a given time. The state determines how the object behaves when its methods are executed and is typically stored in instance variables.
  - Identity - An object’s identity distinguishes it from other objects, even if they share the same behavior and state. Each object has a unique identity, usually represented by its memory reference.

- Classes are interact with each other in three common ways -
  - __Dependence (uses-a)__ -
    - A class depends on another class if its methods use or manipulate objects of that class.
    - The dependency exists only when objects of one class need to access objects of another class.
    - Example - an `Order` class depends on an `Account` class to check a customer’s credit status.
    - If a class does not use another class, no dependency relationship exists.
    - Dependencies should be minimized to reduce coupling between classes.
    - Low coupling ensures that changes in one class are less likely to affect other classes.

  - __Aggregation (has-a)__ -
    - Aggregation represents a containment relationship between classes.
    - Objects of one class contain objects of another class.
    - Example - an `Order` object contains multiple `Item` objects.
    - Aggregation models real-world “part-of” relationships.
    - It helps build complex objects from simpler components.

  - __Inheritance (is-a)__ -
    - Inheritance expresses a relationship between a general class and a more specialized class.
    - A subclass inherits methods and behavior from its superclass.
    - The subclass may add new behavior or modify inherited behavior.
    - Example - a `RushOrder` class is a specialized form of `Order` with priority handling.
    - Inheritance promotes code reuse and enables polymorphism.

- __Constructor__ -
  - Special method whose purpose is to construct and initialize objects.
  - A constructor -
    - Always have the same name as the class name. 
    - A class can have more than one constructor.
    - Can have zero, one, or more parameters.
    - Has no return value.
    - Always called with the `new` operator.
  - Example - to construct a `Date` object, combine the constructor with the `new` operator - `new Date()`
  - The return value of the `new` operation is a reference.
  - You can explicitly set an object variable to `null` to indicate that it currently refers to no object.

## `LocalDate` class

- __`Date` vs `LocalDate`__ -
  - `Date` class -
    - Represents a specific point in time.
    - Older class; includes methods like `getDay()`, `getMonth()`, `getYear()`.
  - `LocalDate` class -
    - Represents a date in calendar notation (year, month, day).
    - Part of modern Java date-time API (Java 8+).
    - Encourages good OO design by separating time measurement from calendar representation.
    - Objects are immutable; methods return new objects rather than modifying the original.

- Constructing `LocalDate` Objects -
  - _Static factory methods_ are used instead of constructors.
  - Current date - `LocalDate today = LocalDate.now()`
  - Specific date - `LocalDate newYearsEve = LocalDate.of(1999, 12, 31)`

- Accessor Methods -
  - Useful for computed dates or operations on existing objects.
  - Get year - `newYearsEve.getYear()`
  - Get month - `newYearsEve.getMonthValue()`
  - Get day - `newYearsEve.getDayOfMonth()`

- Date Arithmetic -
  - `plusDays` -
    - Immutable method - original object remains unchanged - `LocalDate aThousandDaysLater = newYearsEve.plusDays(1000)`
    - Mutator example - `GregorianCalendar.add` - changes the state of the object -
    ```
    GregorianCalendar someDay = new GregorianCalendar(1999, 11, 31); // month 0-11
    someDay.add(Calendar.DAY_OF_MONTH, 1000);
    ```

> [!TIP]
> `jdeprscan` (Java Deprecated API Scanner) is a Java tool for detecting deprecated API usage in your code. It analyzes Java class files or JARs to report usage of APIs that are marked as deprecated in a particular JDK version.
>
> Usage - `jdeprscan --release <version> <file-or-directory>`
>
>   - `--release <version>` - Target Java version for deprecation checks (e.g., 17, 21).
>
>   - `<file-or-directory>` - The `.class` files or `.jar` files to scan.
>
> Other common options -
>   - `--class-path <path>` - Specify classpath for dependent classes.
>
>   - `--log <file>` - Write output to a file instead of console.

## Classes

- Syntax -
```
class ClassName {
    field1
    field2
    . . .
    constructor1
    constructor2
    . . .
    method1
    method2
    . . .
}
```

- Example -
```
class Employee {
    // instance fields
    private String name;
    private double salary;

    // constructor
    Employee(String n, double s) {
        name = n;
        salary = s;
    }

    // a method
    String getName() {
        return name;
    }
}
```

- Constructing instance - `new Employee("John Doe", 75000)`

- The `private` keyword makes sure that the only methods that can access these instance fields are the methods of the `Employee` class itself. No outside method can read or write to these fields.

> [!TIP]
> It is best to make all your instance fields `private`.

> [!TIP]
> It is common to declare classes and methods as `public`.

- You can declare local variables with the `var` keyword instead of specifying their type, provided their type can be inferred from the initial value, eg - 
```
// instead of declaring 
Employee harry = new Employee("Harry Hacker", 50000, 1989, 10, 1);

// you simply write
var harry = new Employee("Harry Hacker", 50000, 1989, 10, 1);
```

- __Handling Null Fields in Classes__ -
  - Fields can be null if not properly initialized.
  - Example - in `Employee` class -
    - `name` → can be `null` if constructor argument is `null`.
    - `salary` → primitive type, cannot be `null`.

  - Strategies for Null Arguments -
    - Permissive Approach - 
      - Replace `null` with a default value.
      - Example -
      ```
      if (n == null) 
        name = "unknown"; 
      else 
        name = n;
      ```

      - Using `Objects` utility class -
      ```
      Employee(String n, double s, int year, int month, int day) {
         name = Objects.requireNonNullElse(n, "unknown");
        // ... other initializations
      }
      ```

      - Effect - No nulls, object always has a usable value.

    - “Tough Love” Approach -
      - Reject `null` arguments with an exception.
      - Example - using `Objects` utility class
      ```
      Employee(String n, double s, int year, int month, int day) {
        name = Objects.requireNonNull(n, "The name cannot be null");
        // ... other initializations
      }
      ```

      - Effect - Forces the caller to provide valid data.
      - Throws `NullPointerException` if argument is `null`.

- Methods Operate on Objects -
  - Methods access and modify instance fields of objects.
  - Example -
  ```
  void raiseSalary(double byPercent) {
    double raise = salary * byPercent / 100;
    salary += raise;
  }
  ```

  - Usage - `number007.raiseSalary(5)`
  - Explicit argument - the value inside parentheses (`5` in the example).
  - Implicit argument - the object before the method name (`number007`).
  - Every method has an implicit parameter this referring to the object -
  ```
  void raiseSalary(double byPercent) {
    double raise = this.salary * byPercent / 100;
    this.salary += raise;
  }
  ```

### `java.util.Objects` API 

| Method                                                      | Return Type | Behavior                                                               | Notes                                 |
| ----------------------------------------------------------- | ----------- | ---------------------------------------------------------------------- | ------------------------------------- |
| `requireNonNull(T obj)`                                     | `void`      | Throws `NullPointerException` if `obj` is null                         | No error message                      |
| `requireNonNull(T obj, String message)`                     | `void`      | Throws `NullPointerException` if `obj` is null                         | Uses provided message                 |
| `requireNonNull(T obj, Supplier<String> messageSupplier)`   | `void`      | Throws `NullPointerException` if `obj` is null                         | Message generated lazily via supplier |
| `requireNonNullElse(T obj, T defaultObj)`                   | `T`         | Returns `obj` if non-null, otherwise returns `defaultObj`              | Default value is immediate            |
| `requireNonNullElseGet(T obj, Supplier<T> defaultSupplier)` | `T`         | Returns `obj` if non-null, otherwise calls supplier and returns result | Default value created lazily          |


## Class Based Access Privileges

- A method can access the `private` fields of the object it is invoked on.
- A method of a class can access `private` fields of any other object of the same class -
  ```
  class Employee {
      private int id;
      // other fields

      public boolean equals(Employee other) {
          return id == other.id;                // accesses private field of 'other'
      }
  }

  if (harry.equals(boss)) { ... }               // valid
  ```

  - Legal because `boss` is an `Employee` and the method belongs to the same class.
  - This enables methods like `equals`, `compareTo`, or `copy` constructors to work efficiently.

## Private Methods

- Instance fields are made `private` to protect data.
- Similarly, helper methods (internal computations) are often not part of the public interface.
- Reasons to make methods private -
  - They are implementation details.
  - May require a special protocol or calling order.
  - Not meant to be used outside the class.

## Final Instance Fields

- An instance field can be declared as `final`.
- Rules for `final` fields -
  - Must be initialized during object construction (in every constructor).
  - Cannot be reassigned after the constructor completes.

- Example -
```
class Employee {
    private final String name; // cannot change after construction
}
```

- Use-cases -
  - Useful for fields that never change -
    - Primitive types (int, double, etc.)
    - Immutable classes (e.g., String, LocalDate)
- Benefits -
  - Guarantees immutability of the reference.
  - Improves clarity and safety of your class design.

- Final Fields in Mutable Classes -
  - `final` applies to the reference, not the object itself.
  - Example -
  ```
  private final StringBuilder evaluations;

  evaluations = new StringBuilder(); // initialized in constructor
  ```

  - You cannot reassign evaluations to a new object, but the object itself can be mutated -
  ```
  void giveGoldStar() {
    evaluations.append(LocalDate.now() + ": Gold star!\n");
  }
  ```

- - A `final` field can be `null`, but once set, it cannot change -
```
name = n != null && n.length() == 0 ? null : n
```

## Static Members

- Static methods are methods that do not operate on objects, eg - `Math.pow`.
  - It does not use any `Math` object to carry out its task i.e. it has no implicit parameter.
  - Therefore, static methods do not have `this` parameter.
  - A static method can access static field, but not an instance field.
  - It is legal to call static methods on the objects, but is not recommended.

- Use static methods in two situations when -
  - When a method doesn’t need to access the object state because all needed parameters are supplied as explicit parameters.
  - When a method only needs to access static fields of the class.

- Classes such as `LocalDate` and `NumberFormat` use static factory methods that construct objects. Reasons to prefer a factory method over constructor -
  - You can’t give names to constructors. The constructor name is always the same as the class name. But we want two different names to get the _currency_ instance and the _percent_ instance.
  - When you use a constructor, you can’t vary the type of the constructed object. But the factory methods actually return objects of the class `DecimalFormat`, a more specialized class that inherits from `NumberFormat`.
  - A constructor always constructs a new object, but you may want to share instances. For example, the call `Set.of()` yields the same instance of an empty set when you call it twice.

### __`main` method__
  
- Traditionally, `main` was a static method -
  - The static `main` method does not operate on any objects. 
  - When a program starts, there aren’t any objects yet. 
  - The `main` method executes and constructs the objects that the program needs.

- As of Java 25, the `main` method no longer needs to be `static` and `public`, and it need not have a parameter of type `String[]`.

- Rules of `main` method -
  - If there is more than one `main` method, static main methods are preferred over instance methods.
  - Methods with a `String[]` parameter are preferred over those with no parameters.
  - Private main methods are not considered.
  - If `main` is not static, the class must have a non-private no-argument constructor. Then the launcher constructs an instance of the class and invokes the `main` method on it.

- Also as of Java 25, a `main` method no longer needs to be declared inside a class. 
  - A source file with method declarations outside a class is a _compact compilation unit_. 
  - It implicitly declares a class whose name is derived from the source file.
  - Technically, the class name can depend on the host name but is usually the file name with its extension removed i.e. if file name is `Application.java` then class name is likely to be `Application`.

> [!NOTE]
> For an implicitly declared class -
>   - you can declare a constructor because you cannot rely on the name of the class, which is also the name of any constructor.
>   - It doesn’t make sense to implicitly declare classes that you want to use in other classes. You need a reliable name to use the class.
>   - The practical use for a compact compilation unit is a class with a `main` method, which you launch as a source file.

> [!TIP]
> Every class can have a `main` method. That can be handy for adding demonstration code to a class. If that class is part of another program, its `main` method is not executed.

## Method parameters

- __Call by value__ - 
  - Always used in Java.
  - Method gets just the value that the caller provides.
  - Thus, the method gets a _copy_ of all arguments i.e. the method cannot modify the contents of any variables in the method call.
- __Call by reference__ - 
  - Method gets the location of the variable that the caller provides. 
  - Thus, a method can modify the value stored in a variable passed by reference.

> [!TIP]
> Whenever we pass an object reference to the methods, the method gets a copy of the object reference (hence, still using call by value), but because both the original and the copy refer to the same object, therefore, method can update the states of the object.

## Object Construction

- __Overloading__ -
  - Occurs if several methods have the same name but different parameters. 
  - Example - `new StringBuilder()` and `new StringBuilder("To do!")`
  - The compiler must sort out which method to call. 
    - It picks the correct method by matching the parameter types in the declarations of the various methods with the types of the arguments used in the specific method call.
  - The process of finding a match is called _overloading resolution_.

- Java allows you to overload any method—not just constructor methods based on the method signature.

> [!NOTE]
> The return type is not part of the method signature i.e. you cannot have two methods with the same names and parameter types but different return types.

> [!TIP]
> Local variables are not initialized with their default values - you must explicitly initialise them.
> Instance variables are automatically initialised to their default value (`0`, `false`, `null` etc), if not initialised explicitly.

- If you write a class with no constructors, then a no-argument constructor is automatically added by the compiler which sets all the instance fields to their default values.

- If a class supplies at least one constructor but does not supply a no-argument constructor, it is illegal to construct objects without supplying arguments.

- A constructor can call another constructor of the same class using `this`, eg -
```
Employee(double s) {
    this("Employee #" + nextId, s);           // calls Employee(String, double)
    nextId++;
}
```

> [!NOTE]
> The Object class has a `finalize` method that classes can override for cleanup any resources which is intended to be called before the garbage collector sweeps away an object. However, you simply cannot know when this method will be called, and it is now deprecated for removal.

- Before Java 25, the call to the other constructor had to be the first statement of the constructor body. 
  - This restriction has now been removed.
  - However there are some restrictions on what can happen between the start of a constructor and the call of another constructor. This phase is called the _early construction context_.

- In the early construction context, you may not -
  - Read any instance variable.
  - Write any instance variable that has an explicit initialization.
  - Invoke any methods on this.
  - Pass this to any other methods.

## Initialization Blocks

- A code block inside a class (not inside a method).
- Executed every time an object is constructed.
- Runs before the constructor body.

- Example -
```
class Employee {
    private static int nextId;

    private int id;
    private String name;
    private double salary;

    // object initialization block
    {
        id = nextId;
        nextId++;
    }
}
```

- Purpose -
  - Used to share common initialization logic across all constructors.
  - Ensures the same code runs regardless of which constructor is called.

> [!TIP]
> Initialization blocks are rarely used and is good practise to place them after field declarations.

## Static Initialization

- Static fields can be initialized in two ways -
  - Inline Initialization - 
    - `private static int nextId = 1`
    - Simple and preferred when initialization logic is trivial

  - Static Initialization Block -
    - Used when initialization requires multiple statements or complex logic.
    - Marked with the keyword static.
    - Runs once, when the class is first loaded.
    - Example -
    ```
    static {
      nextId = generator.nextInt(10000);
    }
    ```

- Execution order -
  - Static fields default to - `0`, `false`, or `null`.
  - Then -
    - Static field initializers run
    - Static initialization blocks run
  - Executed in the order they appear in the class.

- Happens when the class is loaded, not when objects are created.
- Triggered by -
  - First object creation
  - First access to a static field or method
  - Explicit class loading

- It is possible to have cycles in static initialization -
  - Example -
  ```
  class Config {
    static final Config DEFAULT = new Config();
    String get(String key) { . . . }
  }

  class Logger {
    static final Logger DEFAULT = new Logger(Config.DEFAULT.get("logger.default.file"));
    void log(String message) { . . . }
  }
  ```

  - Now suppose the Config constructor adds a logging message -
  ```
  Config() {
    // read configuration
    Logger.DEFAULT.log("Config read successfully");
  }
  ```

  - The first time you use a `Logger`, the static initializization of the `Config` class invokes the `Config` constructor. 
  - It calls the `log` method on the `Logger.DEFAULT` variable, which has not yet been set. 
  - A `NullPointerException` occurs, which causes a fatal `ExceptionInInitializerError`.

## Records

- A record is a special form of a class designed to model immutable data.
- Its state -
  - is fixed at construction time
  - is publicly readable
- Records are ideal for data carriers whose entire state is represented by a fixed set of fields.

- Declaring a Record - 
  - `record Point(double x, double y) { }`
  - This declaration automatically creates -
    - Instance Fields (called Components) -
    ```
    private final double x;
    private final double y;
    ```

- Automatically Provided Members -
  - Canonical Constructor -
    - automatically defined constructor that sets all instance fields.
    - `Point(double x, double y)`

  - Accessor Methods -
    ```
    double x()
    double y()
    ```

    - Accessors use the component name, not `getX()` or `getY()` -
    ```
    var p = new Point(3, 4);
    IO.println(p.x() + " " + p.y());
    ``` 

  - Automatically Generated Methods - `toString`, `equals`, `hashCode`

- Adding Methods to Records -
```
record Point(double x, double y) {
    double distanceFromOrigin() {
        return Math.hypot(x, y);
    }
}
```

- Can override generated methods if the signature matches (but strongly discouraged as it violates the principle of records as transparent data holders.) -
```
record Point(double x, double y) {
    public double x() { return 2 * x; }     // Legal but misleading
}
```

- No Additional Instance Fields -
```
record Point(double x, double y) {
    private double z;                       // ERROR
}
```

- All components are implicitly `final`, but references may point to mutable objects -
```
record PointInTime(double x, double y, Date when) { }

var pt = new PointInTime(0, 0, new Date());
pt.when().setTime(0);                       // Mutates record state
```

- A record can be declared inside another class. 
  - The enclosing class may access record fields directly.
  - `p.x   // instead of p.x()`
  - This also applies to records in a compact compilation unit.

- Records may declare static fields and methods -
```
record Point(double x, double y) {
    static Point ORIGIN = new Point(0, 0);

    static double distance(Point p, Point q) {
        return Math.hypot(p.x - q.x, p.y - q.y);
    }
}
```

- Compact Constructor -
  - Used to validate or normalize parameters.
  - You cannot read or modify the instance fields in the body of the compact constructor.
  - Example -
  ```
  record Range(int from, int to) {
    Range {
        if (from > to) throw new IllegalArgumentException();
    }
  }
  ```

- Custom Constructors
  - Must delegate to another constructor.
  - Ultimately must invoke the canonical constructor.
  - Example - this record has two constructors: the canonical constructor and a no-argument constructor yielding the origin -
  ```
  record Point(double x, double y) {
    Point() {
        this(0, 0);
    }
  }
  ```

## Packages

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

## The Class Path

- The class path tells Java where to find `.class` and `.jar` files.
- Examples -  
  - Linux / macOS - `/home/user/classdir:.:/home/user/archives/archive.jar`
  - Windows - `c:\classdir;.;c:\archives\archive.jar`
  - where `.` = current directory

- Wildcard for JARs -
  - `/home/user/archives/*`
  - Includes all JARs
  - Does NOT include `.class` files

- Search Order -
  - Java API
  - Class path directories
  - JAR files

- Setting the Class Path -
  - Use command-line option - 
    - `java -classpath path MyProgram`
    - Aliases -
      - `-cp`
      - `--class-path`

  - Using `CLASSPATH` environment variable -
    - Unix shell - `export CLASSPATH=/home/user/classdir:.:/home/user/archives/archive.jar`
    - Windows shell - `set CLASSPATH=c:\classdir;.;c:\archives\archive.jar`
    - The class path is set until the shell exits.

> [!TIP]
> When using a JAR in jshell, launch it with - `CLASSPATH=archive.jar jshell`.

> [!TIP]
> Classes can also be loaded from the _module path_.

## JAR Files
  
- JAR = compressed archive.
- Contains -
  - `.class` files
  - Resources (images, sounds, config)
- Used to distribute applications and libraries.
- Built on ZIP format.

- Creating jar files -
  - Using `jar` tool - 
    - In the default JDK installation, it’s in the `$JAVA_HOME/bin` directory.
    - Syntax - `jar cvf jarFileName file1 file2 . . .`
    - Example - `jar cvf CalculatorClasses.jar *.class icon.png`

- `jar` program options -

| Option | Description                                                                                               |
| ------ | --------------------------------------------------------------------------------------------------------- |
| `c`    | Creates a new (or empty) JAR archive and adds files to it. Directories are processed __recursively__.     |
| `C`    | Temporarily changes the directory when adding files. <br>Example:<br>`jar cvf app.jar -C classes *.class` |
| `e`    | Specifies the __entry point__ (main class) in the manifest.                                               |
| `f`    | Specifies the __JAR file name__ as the next argument. <br>If omitted, `jar` uses standard input/output.   |
| `i`    | Creates an __index file__ for faster class lookup in large archives.                                      |
| `m`    | Adds a __custom manifest__ file to the JAR.                                                               |
| `M`    | Prevents creation of a manifest file.                                                                     |
| `t`    | Displays the __table of contents__ of the JAR.                                                            |
| `u`    | __Updates__ an existing JAR file.                                                                         |
| `v`    | Enables __verbose output__.                                                                               |
| `x`    | __Extracts__ files from the JAR. <br>If no file names are given, extracts everything.                     |
| `0`    | Stores files __without ZIP compression__.                                                                 |


## The Manifest

- Each JAR file contains a manifest file that describes special features of the archive.
- The manifest file is called `MANIFEST.MF` and is located in a special `META-INF` subdirectory of the JAR file.
- Minimum legal manifest - `Manifest-Version: 1.0`
- The manifest entries are grouped into sections -
  - The first section in the manifest is called the main section. It applies to the whole JAR file. 
  - Subsequent entries can specify properties of named entities such as individual files, packages, or URLs. 
  - Those entries must begin with a Name entry. 
  - Sections are separated by blank lines.
  - Example -
  ```
  Manifest-Version: 1.0
  lines describing this archive

  Name: Woozle.class
  lines describing this file
  Name: com/mycompany/mypkg/
  lines describing this package
  ```

- To edit the manifest - 
  - Place the lines that you want to add to the manifest into a text file. 
  - Then run - `jar cfm jarFileName manifestFileName . . .`
  - For example, to make a new JAR file with a manifest, run - 
  ```
  jar cfm MyArchive.jar manifest.mf com/mycompany/mypkg/*.class
  ```

- To update the manifest of an existing JAR file, place the additions into a text file and use a command such as - 
```
jar ufm MyArchive.jar manifest-additions.mf
```

- Executable jar files -
  - Use the `e` option of the jar command to specify the entry point of your program — the class that you would normally specify when invoking the java program launcher -
    - `jar cvfe MyProgram.jar com.mycompany.mypkg.MainAppClass files to add`
  - Alternatively, you can specify the main class of your program in the manifest, including a statement of the form -
    - `Main-Class: com.mycompany.mypkg.MainAppClass`
    - Do not add a `.class` extension to the main class name.

> [!WARNING]
> The last line in the manifest must end with a newline character. Otherwise, the manifest will not be read correctly. It is a common error to produce a text file containing just the Main-Class line without a line terminator.

- For backward compatibility, version-specific class files are placed in the `META-INF/versions` directory.
- To add versioned class files, use the `--release` flag - 
```
jar uf MyProgram.jar --release 9 Application.class
```

- To build a multi-release JAR file from scratch, use the `-C` option and switch to a different class file directory for each version - 
```
jar cf MyProgram.jar -C bin/8 . --release 9 -C bin/9 Application.class
```

- When compiling for different releases, use the `--release` flag and the `-d` flag to specify the output directory - 
  - `javac -d bin/8 --release 8 . . . `
  - The `-d` option creates the directory if it doesn’t exist.

> [!TIP]
> The sole purpose of multi-release JARs is to enable a particular version of your program or library to work with multiple JDK releases. If you add functionality or change an API, you should provide a new version of the JAR instead.

- Tools such as `javap` are not retrofitted to handle multi-release JAR files. 
  - If you call - `javap -classpath MyProgram.jar Application.class` - you get the base version of the class (which, after all, is supposed to have the same public API as the newer version). 
  - If you must look at the newer version, call - 
  ```
  javap -classpath MyProgram.jar\!/META-INF/versions/9/Application.class
  ```

> [!TIP]
> You can use the `JDK_JAVA_OPTIONS` environment variable to pass command-line options to the java launcher - `export JDK_JAVA_OPTIONS='--class-path /home/user/classdir -enableassertions'`

## Documentation Comments

- `javadoc` tool generates HTML documentation from your source files from the comments that start with a ¬ and ends with a `*/`.
- Each comment is placed immediately above the member it describes. 
- The `javadoc` utility extracts information for the following items -
  - Modules
  - Packages
  - Public classes and interfaces
  - Public and protected fields
  - Public and protected constructors and methods
- Each `/** . . . */` documentation comment contains free-form text followed by tags. 
  - A tag starts with an `@`, such as `@since` or `@param`.
  - The first sentence of the free-form text should be a summary statement. The javadoc utility automatically generates summary pages that extract these sentences.
  - The most common javadoc tags are block tags. They must appear at the beginning of a line and start with `@`, optionally preceded by whitespace, the comment delimiter `/**`, or leading `*` which are often used for multiline comments. 
  - In contrast, inline tags are enclosed in braces - `{@tagname contents}`. 
  - The contents may contain braces, but they must be balanced. Examples are the `@code` and `@link` tags.

- JavaDoc text can contain HTML tags such as `<em>` or `ul` - they are passed through to the generated HTML pages.

> [!NOTE]
> Starting with Java 23, you can author JavaDoc comments in Markdown instead of plain text and HTML. Markdown comments are delimited with `///` in front of each line, instead of `/**` and `*/`. JavaDoc follows the CommonMarc specification, and in addition supports the Github Flavored Markdown extension for tables.

- __Class Comments__ - must be placed after any import statements, directly before the class definition.
- __Method Comments__ -
  - Each method comment must immediately precede the method that it describes.
  - In addition to the general-purpose tags, you can use the following tags -
    - `@param` variable description - 
      - This tag adds an entry to the “parameters” section of the current method. 
      - The description can span multiple lines and can use HTML tags. 
      - All `@param` tags for one method must be kept together.
    - `@return` description -
      - This tag adds a “returns” section to the current method. 
      - The description can span multiple lines and can use HTML tags.
    - `@throws` class description -
      - This tag adds a note that this method may throw an exception. 

- It can be tedious to write comments for methods whose description and return value are identical, such as -
  ```
  /**
  * Returns the name of the employee.
  * @return  the name of the employee
  */
  ```

  - In such cases, consider the inline form of @return introduced in Java 16 -
  ```
  /**
  * {@return the name of the employee}
  */
  ```

  - The description section becomes “Returns the name of the employee.”, and a “Returns” section with the same contents is added.

> [!NOTE]
> If you add `{@inheritDoc}` into a method description or the `@param`, `@return`, or `@throws` tag bodies, then the documentation from the method in a superclass or interface is copied verbatim.

- __Field Comments__ -
  - You only need to document public fields—generally that means static constants. 
  - For example -
  ```
  /**
  * The "Hearts" card suit
  */
  public static final int HEARTS = 1;
  ```

- __Package Comments__ -
  - To generate package comments, you need to add a separate file in each package directory.
  - Two choices -
    - Supply a Java file named `package-info.java` -
      - The file must contain an initial documentation comment, delimited with `/**` and `*/`, followed by a `package` statement. 
      - It should contain no further code or comments.
    - Supply an HTML file named `package.html` -
      - All text between the tags `<body>. . .</body>` is extracted.
  
- For multiline code displays in an HTML `pre` tag, you can use -
  ```
  /**
  * . . .
  * <pre>{@code
  *     . . .
  *     . . .
  * }</pre>
  */
  ```

  - or, since Java 18 -
  ```
  /**
  * . . .
  * {@snippet :
  *     . . .
  *     . . .
  * }
  */
  ```

- __Links__ -
  - You can use hyperlinks to other relevant parts of the javadoc documentation, or to external documents, with the `@see` and `@link` tags.
  - The tag `@see` reference adds a hyperlink in the “see also” section.
    - Where `reference` can be one of the following -
    ```
    package.class#member label
    <a href=". . .">label</a>
    "text"
    ```

  - Example - `@see com.example.Employee#raiseSalary(double)`
    - makes a link to the `raiseSalary(double)` method in the `com.example.Employee` class.
    - You can omit the name of the package, or both the package and class names. Then, the member will be located in the current package or class.
    - Note that you must use a #, not a period, to separate the class from the method or variable name.
  
  - Constructors have the special name `<init>`, not the name of the class, such as - `@see com.example.Employee#<init>()`

  - You can specify an optional `label` after the member that will appear as the link anchor. If you omit the label, the user will see the member name.

  - If the `@see` tag is followed by a `<` character, then you need to specify a hyperlink -
    - You can link to any URL you like, eg - `@see <a href="horstmann.com/corejava.html">The Core Java home page</a>`

  - If the `@see` tag is followed by a `"` character, then the text is displayed in the “see also” section. 
    - For example - `@see "Core Java Volume 2"`

  - You can add multiple `@see` tags for one member, but you must keep them all together.

  - You can place hyperlinks to other classes or methods anywhere in any of your documentation comments. 
    - Insert a tag of the form - `{@link package.class#member}` - anywhere in a comment. 
    - The member reference follows the same rules as for the `@see` tag.

  - In a code snippet, place the `@link` tag in the comment, so that it doesn’t interfere with the code -
  ```
  {@snippet
    IO.println(); // @link substring=println target=java.lang.IO#println()
  }
  ```

  - Since Java 20, ids are automatically generated for level 2 and level 3 headings.
    - Example - `<h2>General Principles</h2>`
    - Gets an id general-principles-heading, which you can refer from `@see` and `@link` tags. 
    - You need two `#` symbols to link to an id -
    ```
    {@link com.horstmann.corejava.Employee##general-principles-heading}
    ```

  - Use `@linkplain` instead of `@link` if a link should be displayed in the plain font instead of the code font.

> [!NOTE]
> If your comments contain links to other files, such as images (for example, diagrams or images of user interface components), place those files into a subdirectory, named `doc-files`, of the directory containing the source file. 
>
> The javadoc utility will copy the `doc-files` directories and their contents from the source directory to the documentation directory. 
>
> You need to use the `doc-files` directory in your `link`, for example` <img src="doc-files/uml.png" alt="UML diagram"/>`.

- __General Comments__ -
  
| Tag                | Purpose                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `@since text`      | Specifies the version when the feature was introduced (e.g. `@since 1.7.1`).             |
| `@author name`     | Lists the author(s) of the class (optional; VCS is preferred today).                     |
| `@version text`    | Specifies the current version of the class.                                              |
| `@deprecated text` | Explains why the feature is deprecated and what to use instead. Used with `@Deprecated`. |

- __Inline Tags__ -

| Tag                        | Purpose                                             |
| -------------------------- | --------------------------------------------------- |
| `{@value constant}`        | Inserts the value of a constant field.              |
| `{@value format constant}` | Inserts formatted constant value (Java 20+).        |
| `{@index entry}`           | Adds an entry to the generated documentation index. |

  - Example - `{@value %X Integer#MAX_VALUE}   // → 7FFFFFFF`

- __Code Snippets (Java 18+)__ -

| Syntax                                                 | Description                                 |
| ------------------------------------------------------ | ------------------------------------------- |
| `{@snippet file=EmployeeDemo.java}`                    | Includes an entire source file              |
| `{@snippet class=com.horstmann.corejava.EmployeeDemo}` | Includes a class                            |
| 📁                                                     | Files must be in `snippet-files/` directory |

- __Snippet Regions__ -
```
// @start region=default-employee
var e = new Employee();
String name = e.getName();
// @end
```

- __Snippet Decorations__ - 

| Decoration            | Example                                                 |
| --------------------- | ------------------------------------------------------- |
| Highlight (substring) | `// @highlight substring=new`                           |
| Highlight (regex)     | `// @highlight regex=get[A-Z][a-z]+`                    |
| Replace               | `// @replace regex=([^)]+) replacement="(..., ...)"`    |
| Link                  | `// @link substring=Card target=package#Class.<init>()` |

- __Comment Extraction (javadoc tool)__ -

| Task              | Command                       |
| ----------------- | ----------------------------- |
| Single package    | `javadoc -d docs packageName` |
| Multiple packages | `javadoc -d docs pkg1 pkg2`   |
| Unnamed package   | `javadoc -d docs *.java`      |

- __Important Options__ -

| Option           | Effect                           |
| ---------------- | -------------------------------- |
| `-d dir`         | Output directory                 |
| `-author`        | Includes `@author` tags          |
| `-version`       | Includes `@version` tags         |
| `-link URL`      | Links to external API docs       |
| `-linksource`    | Generates HTML source with links |
| `-overview file` | Adds project-wide overview       |

- __Overview File__ -

| File              | Purpose                          |
| ----------------- | -------------------------------- |
| `overview.html`   | Project-wide documentation       |
| Extracted content | Text inside `<body>...</body>`   |
| Shown as          | “Overview” tab in generated docs |
