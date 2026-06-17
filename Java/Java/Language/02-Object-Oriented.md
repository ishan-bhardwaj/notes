# Object-oriented Programming

- Object properties -
  - Identity - its unique reference that distinguishes it from other objects
  - State - the data it holds (instance variables)
  - Behavior - what the object can do (its methods)

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
  Employee john;                                // declare reference to object
  john = new Employee("John Doe", 30);          // allocate an Employee object

  // or
  var john = new Employee("John", 30);          
  ```

- A source file can only contain one public class, and the names of the file & the public class must match

## Class Relationships

- Classes can interact with each other in _three_ common ways -
  - __Dependence (uses-a)__ -
    - A class depends on another class but does not model a part-of relationship
    - Example -
      ```java
      class PaymentGateway {
        public boolean charge(double amount) {
          IO.println("Charged: " + amount);
          return true;
        }
      }

      class OrderService {

        private var gateway = new PaymentGateway();             // dependency uses-a

        public void placeOrder(double amount) {
          if (gateway.charge(amount)) {
            IO.println("Order placed successfully");
          }
        }
      }
      ```

  - __Aggregation (has-a)__ -
    - Object _contains_ other objects (long-term relationship)
    - Example -
      ```java
      record Item(String name, double price) {}

      import java.util.List;

      class Order {

        private List<Item> items;                        // aggregation (has-a)

        public Order(List<Item> items) {
          this.items = items;
        }

        public double totalPrice() {
          return items.stream()
                 .mapToDouble(Item::price)
                 .sum();
        }
      }
      ```

  - __Inheritance (is-a)__ -
    - Inheritance expresses a relationship between a general class and a more specialized class
    - A subclass inherits methods and behavior from its superclass
    - Example -
      ```java
      class Order {
        public int deliveryDays() {
          return 5;
        }
      }

      class RushOrder extends Order {                  // is-a relationship
        @Override
        public int deliveryDays() {
          return 1;
        }
      }
      ```

## Constructor
  
- Special method whose purpose is to construct and initialize objects
  ```java
  class Order {
    int id;
    String name;

    Order(int id, String name) {
      this.id = id;
      this.name = name;
    }

    Order() {                           // can have more than one constructor.
      this(0, "unknown");
    }
  }
  ```

- A constructor -
  - Always have the same name as the class name
  - Has no return value
  - Always called with the `new` operator -
    ```java
    var order1 = new Order();                // return value is a reference
    var order2 = null;                       // refers to no object
    ```

- Constructors can be - 
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

- Final fields must be initialized in the constructor -
  ```java
  class Order {
    final int id;

    Order(int id) {
        this.id = id;           // mandatory, otherwise compiler error
    }
  }
  ```

> [!NOTE] 
> Compiler automatically adds a no-argument constructor, iff no constructor is provided explicitly

> [!NOTE]
> Every method has an implicit parameter `this` referring to the object -
>
> ```java
> void raiseSalary(double byPercent) {
>   double raise = this.salary * byPercent / 100;
>   this.salary += raise;
> }
> ```

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

- A code block inside a class (not inside a method)
- Executed every time an object is constructed, before the constructor

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
> Initialization blocks are rarely used and is good practise to place them after field declarations

## Static Initialization

- Runs once, when the class is first loaded
- Static fields can be initialized in two ways -
  - Inline Initialization - 
    - `private static int nextId = 1`

  - Static Initialization Block -
    - Example -
    ```java
    static {
      nextId = generator.nextInt(10000);
    }
    ```

## Handling Null Fields in Classes
  
- __Permissive Approach__ - 
  - Replace `null` with a default value -
    ```java
    if (n == null) 
      name = "unknown"; 
    else 
      name = n;
    ```

  - Using `Objects` utility class -
    ```java
    Employee(String n, double s) {
      name = Objects.requireNonNullElse(n, "unknown");
      // ... other initializations
    }
    ```

- “Tough Love” Approach -
  - Reject `null` arguments with an exception -
    ```java
    Employee(String n, double s) {
      name = Objects.requireNonNull(n, "The name cannot be null");
      // ... other initializations
    }
    ```

## Method parameters

- __Call by value__ - 
  - Always used in Java
  - Method gets a _copy_ of all arguments
- __Call by reference__ - 
  - Method gets the location of the variable that the caller provides
  - Thus, a method can modify the value stored in a variable passed by reference

> [!TIP]
> Method gets a copy of the object reference, so it is still using call by value.
> 
> But because both the original and the copy refer to the same object - method can update the states of the object.

> [!TIP]
> Local variables are not initialized with their default values - you must explicitly initialise them
>
> Instance variables are automatically initialised to their default value (`0`, `false`, `null` etc), if not initialised explicitly

## `LocalDate` class

- Represents a date in calendar notation (year, month, day)
- Objects are immutable
- Construction - _static factory methods_ are used instead of constructors -
  ```java
  LocalDate today = LocalDate.now();                    // current date
  LocalDate newYearsEve = LocalDate.of(1994, 11, 1);    // specific date

  // accessor methods
  newYearsEve.getYear();                                // 1994
  newYearsEve.getMonthValue();                          // NOVEMBER
  newYearsEve.getDayOfMonth();                          // 1
  ```

- Date Arithmetic -
  - `plusDays` -
    - Immutable method - original object remains unchanged - 
      ```java
      LocalDate aThousandDaysLater = newYearsEve.plusDays(1000)
      ```

    - Mutator example - `GregorianCalendar.add` - changes the state of the object -
      ```java
      GregorianCalendar someDay = new GregorianCalendar(1999, 11, 31); // month 0-11
      someDay.add(Calendar.DAY_OF_MONTH, 1000);
      ```

# Packages

- A class can use -
  - All classes in its own package
  - All public classes from other packages
  
- Importing classes -
  - Using fully qualified names -
    - `java.time.LocalDate today = java.time.LocalDate.now()`
    - Always works, but tedious
  - Using `import` -
    ```java
    import java.time.*;                   // imports all classes in java.time package
    import java.time.LocalDate;           // specific class import

    LocalDate today = LocalDate.now();
    ```

> [!TIP]
> `import` is purely for convenience. The bytecode always uses fully qualified names

> [!TIP]
> `java.lang`is imported automatically

- __Module Imports__ (Java 25+) -
  - Java packages can be organized into modules
  - Import all packages in a module -
    - Example - `import module java.xml`
    - Important module - 
      - `import module java.base`
      - Automatically imported in compact source files
      - Includes `java.lang`, `java.util`, `java.time`, `java.io` etc

- __Static Imports__ -
  - Allows importing static methods and fields -
    ```java
    import static java.lang.System.*;
    err.println("Error");
    exit(0);
    ```

  - Enum Constants -
    ```java
    import static java.time.DayOfWeek.*;

    var day = FRIDAY;
    ```

- If you don’t put a package statement in the source file - the classes will belong to the unnamed package

> [!TIP]
> The implicitly declared class of a compact compilation unit (with methods declared outside a class) is always in the unnamed package

- Package structure is important for JVM, not for compiler -
  - A class with `package com.mycompany` can compile even if it is not contained in a subdirectory `com/mycompany`
  - But, the JVM won't be able to execute it

## Class Based Access Privileges

- A method can access the `private` fields of the object it is invoked on.
- A method of a class can access `private` fields of any other object of the same class -
    ```java
    class Employee {
      private int id;
      // other fields

      public boolean equals(Employee other) {
          return id == other.id;                // accesses private field of 'other'
      }
    }

    if (harry.equals(boss)) { ... }               // valid
    ```

  - This enables methods like `equals`, `compareTo`, or `copy` constructors to work

> [!TIP]
> Variables with no access modifiers defaults to package access

## Private Methods

- Instance fields are made `private` to protect data
- Reasons to make methods private -
  - They are implementation details
  - May require a special protocol or calling order
  - Not meant to be used outside the class

## Static Members

- Methods that do not operate on objects, eg - `Math.pow`.
  - Has no implicit (`this`) parameter
  - A static method can access static field, but not an instance field
  - It is legal to call static methods on the objects, but is not recommended

- Classes such as `LocalDate` and `NumberFormat` use static factory methods that construct objects
- Reasons to prefer a factory method over constructor -
  - Can’t give names to constructors
  - Can’t vary the return type of the constructed object
  - Share instances, eg - the call `Set.of()` yields the same instance of an empty set when you call it twice

### __`main` method__
  
- Traditionally, `main` was a static method -
  - When a program starts, there aren’t any objects yet

- Java 25+ - the `main` method no longer needs to be `static` and `public`, and it need not have a parameter of type `String[]`

- Rules of `main` method -
  - If there is more than one `main` method - static `main` methods are preferred
  - Methods with a `String[]` parameter are preferred over those with no parameters
  - If `main` is not static, the class must have a non-private no-argument constructor - 
    - constructs an instance of the class and invokes the `main` method on it

- Java 25+ - `main` method no longer needs to be declared inside a class.
  - A source file with method declarations outside a class is called _compact compilation unit_ 
  - Implicitly declares a class whose name is derived from the source file
  - Eg - if file name is `Application.java` then class name is likely to be `Application`.

## Final Instance Fields

- Rules for `final` instance fields -
  - Must be initialized during object construction (in every constructor)
  - Cannot be reassigned after the constructor completes

- Example -
  ```java
  class Employee {
    private final String name; 
  }
  ```

- Benefits -
  - Guarantees immutability of the reference
  - Improves clarity and safety of your class design

- `final` applies to the reference, not the object itself -
  ```java
  private final StringBuilder evaluations;

  evaluations = new StringBuilder();          // initialized in constructor

  // cannot reassign evaluations to a new object, but the object itself can be mutated
  void giveGoldStar() {
    evaluations.append(LocalDate.now() + ": Gold star!\n");
  }
  ```

# Records

- A record is a special form of a class designed to model immutable data
- Its state -
  - is fixed at construction time
  - is publicly readable
- Records are ideal for data carriers whose entire state is represented by a fixed set of fields

- Declaring a Record - 
  - `record Point(double x, double y) { }`
  - This declaration automatically creates -
    - Instance Fields (called Components) -
      ```java
      private final double x;
      private final double y;
      ```

- Automatically Provided Members -
  - Canonical Constructor -
    - automatically defined constructor that sets all instance fields
    - `Point(double x, double y)`

  - Accessor Methods -
    ```java
    double x()
    double y()
    ```

    - Accessors use the component name, not `getX()` or `getY()` -
      ```java
      var p = new Point(3, 4);
      IO.println(p.x() + " " + p.y());
      ``` 

  - Automatically Generated Methods - `toString`, `equals`, `hashCode`

- Adding Methods to Records -
  ```java
  record Point(double x, double y) {
    
  }
  ```

- Can override generated methods if the signature matches (but strongly discouraged as it violates the principle of records as transparent data holders) -
  ```java
  record Point(double x, double y) {
    public double x() { return 2 * x; }     // Legal but misleading
  }
  ```

- No Additional Instance Fields -
  ```java
  record Point(double x, double y) {
    private double z;                       // ERROR
  }
  ```

- All components are implicitly `final`, but references may point to mutable objects -
  ```java
  record PointInTime(double x, double y, Date when) { }

  var pt = new PointInTime(0, 0, new Date());
  pt.when().setTime(0);                           // Mutates record state
  ```

- A record can be declared inside another class.
  - The enclosing class may access record fields directly
  - `p.x` instead of `p.x()`
  - This also applies to records in a compact compilation unit

- Records can have declare static fields and methods

- Compact Constructor -
  - Used to validate or normalize parameters
  - Cannot read or modify the instance fields in the body of the compact constructor
  - Example -
  ```java
  record Range(int from, int to) {
    Range {
        if (from > to) throw new IllegalArgumentException();
    }
  }
  ```

- Custom Constructors
  - Must delegate to another constructor
  - Ultimately must invoke the canonical constructor
  - Example - this record has two constructors: the canonical constructor and a no-argument constructor yielding the origin -
  ```java
  record Point(double x, double y) {
    Point() {
        this(0, 0);
    }
  }
  ```
