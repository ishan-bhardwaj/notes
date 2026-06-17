# Object-oriented Programming

- Object properties -
    - __Identity__ — unique reference distinguishing it from other objects
    - __State__ — data it holds (instance variables)
    - __Behavior__ — what it can do (methods)

- Example -
  ```java
  class Employee {
    // instance variables
    String name;
    int age;

    // constructors
    Employee(String n, int a) {
        name = n;
        age = a;
    }

    // methods
    String getName() {
        return name;
    }
  }
  ```

- Constructing objects -
  ```java
  Employee john = new Employee("John Doe", 30);
  var john = new Employee("John", 30);                  // shorthand      
  ```

- A source file can only contain one `public` class, and the file name must match the class name

## Class Relationships

- __Dependence (uses-a)__ — class depends on another, no part-of relationship — Eg - `OrderService` uses `PaymentGateway`
- __Aggregation (has-a)__ — object contains other objects — Eg - `Order` has `List<Item>`
- __Inheritance (is-a)__ — subclass inherits from superclass — Eg - `RushOrder extends Order`

## Constructor
  
- Same name as class, no return type, called with `new`
- Can be overloaded, can delegate with `this(...)` -
  ```java
  Order(int id, String name) { this.id = id; this.name = name; }
  Order() { this(0, "unknown"); }
  ```

- Access modifiers - 
  | Modifier        | Access                       |
  | --------------- | ---------------------------- |
  | `public`        | Anywhere                     |
  | `protected`     | Same package + subclass      |
  | package-private | Same package                 |
  | `private`       | Only inside class            |

  ```java
  public class MyClass {
    public    MyClass() {}          // can be instantiated anywhere
    protected MyClass() {}          // same package + subclasses
              MyClass() {}          // package-private — omit modifier
    private   MyClass() {}          // only instantiated within this class
  }
  ```

- `final` fields must be initialized in the constructor, else compiler error

> [!NOTE]
> Compiler auto-adds a no-arg constructor iff no constructor is explicitly defined

> [!NOTE]
> Every method has an implicit `this` parameter referring to the object

> [!TIP]
> `jdeprscan` (Java Deprecated API Scanner) - Java tool for detecting deprecated API usage in your code
>
> Usage - `jdeprscan --release <version> <file-or-directory>`
>
>   - `--release <version>` - Target Java version for deprecation checks (e.g., 17, 21)
>
>   - `<file-or-directory>` - The `.class` files or `.jar` files to scan
>
> Other common options -
>   - `--class-path <path>` - Specify classpath for dependent classes
>
>   - `--log <file>` - Write output to a file instead of console

## Initialization Blocks

- Code block inside class, outside method
- Runs before every constructor

- Example -
  ```java
  class Employee {
    private static int nextId;
    private int id;

    // object initialization block
    {
        id = nextId;
        nextId++;
    }
  }
  ```

- Purpose -
  - Used to share common initialization logic across all constructors
  - Ensures the same code runs regardless of which constructor is called

> [!TIP]
> Rarely used — place after field declarations

## Static Initialization

- Runs once when class is first loaded
- Inline Initialization - `private static int nextId = 1`
- Static Initialization Block -
  ```java
  static {
    nextId = generator.nextInt(10000);
  }
  ```

## Handling Null Fields in Classes
  
- __Permissive__ — `Objects.requireNonNullElse(n, "unknown")`
- __Tough love__ — `Objects.requireNonNull(n, "name cannot be null")` — throws exception

## Method parameters

- Java always uses __call by value__ — method gets a copy of all arguments
- For object references — copy of reference is passed, so object state can still be mutated

> [!TIP]
> Local variables must be explicitly initialized — no defaults
> Instance variables auto-initialize to `0`, `false`, `null` etc

## `LocalDate` Class

- Immutable, uses static factory methods -
  ```java
    LocalDate.now();
    LocalDate.of(1994, 11, 1);
    d.getYear(); d.getMonthValue(); d.getDayOfMonth();
    d.plusDays(1000);                                     // returns new object, original unchanged
  ```

# Packages

- Import styles -
  ```java
    import java.time.*;               // all classes
    import java.time.LocalDate;       // specific
    import module java.base;          // all packages in module (Java 25+)
    import static java.lang.System.*; // static members
  ```

> [!TIP]
> `import` is for convenience only — bytecode always uses fully qualified names

> [!TIP]
> `java.lang` is auto-imported

- No `package` statement → class belongs to unnamed package
- Package directory structure matters to JVM, not compiler -
  - A class with `package com.mycompany` can compile even if it is not contained in a subdirectory `com/mycompany`
  - But, the JVM won't be able to execute it

## Class-Based Access Privileges

- A method can access `private` fields of any object of the same class -
  ```java
    public boolean equals(Employee other) {
        return id == other.id;        // valid
    }
  ```

> [!TIP]
> No modifier → defaults to package access

## Static Members

- No implicit `this` — can access static fields only, not instance fields
- Prefer static factory methods over constructors when —
    - Descriptive names are needed
    - Return type needs to vary
    - Instances should be shared — Eg - `Set.of()`

### `main` Method

- Java 25+ — no longer needs to be `static`, `public`, or take `String[]`
- If both static and instance `main` exist — static is preferred
- `String[]` variant preferred over no-parameter variant
- Can be declared outside a class in a compact compilation unit — class name derived from file name

## Final Instance Fields

- Must be initialized in every constructor, cannot be reassigned after
- `final` applies to the reference, not the object — object itself can still be mutated

## Records

- Immutable data carrier — state fixed at construction, publicly readable
  ```java
    record Point(double x, double y) { }
  ```

- Auto-provided —
    - `private final` fields for each component
    - Canonical constructor
    - Accessors — `p.x()`, `p.y()` (not `getX()`/`getY()`)
    - `toString`, `equals`, `hashCode`

- No additional instance fields allowed
- Components are `final` but referenced objects can be mutable — be careful
- Can have static fields and methods
- Declared inside a class — enclosing class can access as `p.x` instead of `p.x()`

- __Compact constructor__ — validate/normalize only, cannot read/assign fields directly -
  ```java
    record Range(int from, int to) {
        Range { if (from > to) throw new IllegalArgumentException(); }
    }
  ```

- __Custom constructor__ — must delegate to canonical constructor -
  ```java
    record Point(double x, double y) {
        Point() { this(0, 0); }
    }
  ```