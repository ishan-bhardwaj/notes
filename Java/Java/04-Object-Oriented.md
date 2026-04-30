# Object-oriented Programming

- __Encapsulation / Information Hiding__ - bundle data and methods inside an object and hide internal implementation from the outside.

## Classes & Objects

- _Class_ defines a new data type which can be used to create _objects_ of that type.
- Methods and variables defined within a class are called _members_ of the class.
- Three object characteristics -
  - Identity - its unique reference that distinguishes it from other objects.
  - State - the data it holds (instance variables).
  - Behavior - what the object can do (its methods).

- Example -
  ```
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
  ```
  Employee john;                                // declare reference to object
  john = new Employee("John Doe", 30);          // allocate an Employee object

  // or
  var john = new Employee("John", 30);
  ```

- A source file can only contain one public class, and the names of the file & the public class must match.

## Class Relationships

- Classes can interact with each other in _three_ common ways -
  - __Dependence (uses-a)__ -
    - A class depends on another class but does not model a part-of relationship.
    - Example -
      ```
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
    - Object _contains_ other objects (long-term relationship).
    - Example -
      ```
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
    - Inheritance expresses a relationship between a general class and a more specialized class.
    - A subclass inherits methods and behavior from its superclass.
    - Example -
      ```
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
  
- Special method whose purpose is to construct and initialize objects.
  ```
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
  - Always have the same name as the class name.
  - Has no return value.
  - Always called with the `new` operator -
    ```
    var order1 = new Order();                // return value is a reference
    var order2 = null;                       // refers to no object
    ```

- Constructors can be - 
  | Modifier        | Access            |
  | --------------- | ----------------- |
  | public          | Anywhere          |
  | protected       | Same package      |
  | package-private | Same package      |
  | private         | Only inside class |

- Final fields must be initialized in the constructor -
  ```
  class Order {
    final int id;

    Order(int id) {
        this.id = id;           // mandatory, otherwise compiler error
    }
  }
  ```

- JVM Bytecode -
  - Constructor becomes method named `<init>`
  - Example bytecode - 
    ```
    0:aload_0
    1:invokespecial java/lang/Object.<init>
    ```

  - Constructors are _invoked_, not called like methods.

> [!NOTE] 
> Compiler automatically adds a no-argument constructor, iff no constructor is provided explicitly.

> [!NOTE]
> Every method has an implicit parameter `this` referring to the object -
>
> ```
> void raiseSalary(double byPercent) {
>   double raise = this.salary * byPercent / 100;
>   this.salary += raise;
> }
> ```

> [!TIP]
> `jdeprscan` (Java Deprecated API Scanner) - Java tool for detecting deprecated API usage in your code.
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

## Initialization Blocks

- A code block inside a class (not inside a method).
- Executed every time an object is constructed, before the constructor.

- Example -
```
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
  - Used to share common initialization logic across all constructors.
  - Ensures the same code runs regardless of which constructor is called.

> [!TIP]
> Initialization blocks are rarely used and is good practise to place them after field declarations.

## Static Initialization

- Runs once, when the class is first loaded.
- Static fields can be initialized in two ways -
  - Inline Initialization - 
    - `private static int nextId = 1`

  - Static Initialization Block -
    - Example -
    ```
    static {
      nextId = generator.nextInt(10000);
    }
    ```

## Handling Null Fields in Classes
  
- __Permissive Approach__ - 
  - Replace `null` with a default value -
    ```
    if (n == null) 
      name = "unknown"; 
    else 
      name = n;
    ```

  - Using `Objects` utility class -
    ```
    Employee(String n, double s) {
      name = Objects.requireNonNullElse(n, "unknown");
      // ... other initializations
    }
    ```

- “Tough Love” Approach -
  - Reject `null` arguments with an exception -
    ```
    Employee(String n, double s) {
      name = Objects.requireNonNull(n, "The name cannot be null");
      // ... other initializations
    }
    ```

## Method parameters

- __Call by value__ - 
  - Always used in Java.
  - Method gets a _copy_ of all arguments.
- __Call by reference__ - 
  - Method gets the location of the variable that the caller provides. 
  - Thus, a method can modify the value stored in a variable passed by reference.

> [!TIP]
> Method gets a copy of the object reference, so it is still using call by value.
> 
> But because both the original and the copy refer to the same object - method can update the states of the object.

> [!TIP]
> Local variables are not initialized with their default values - you must explicitly initialise them.
>
> Instance variables are automatically initialised to their default value (`0`, `false`, `null` etc), if not initialised explicitly.

## `LocalDate` class

- Represents a date in calendar notation (year, month, day).
- Objects are immutable.
- Construction - _static factory methods_ are used instead of constructors -
  ```
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
      ```
      LocalDate aThousandDaysLater = newYearsEve.plusDays(1000)
      ```

    - Mutator example - `GregorianCalendar.add` - changes the state of the object -
      ```
      GregorianCalendar someDay = new GregorianCalendar(1999, 11, 31); // month 0-11
      someDay.add(Calendar.DAY_OF_MONTH, 1000);
      ```

# Packages

- Java allows you to group related classes into a package.
- Packages serve three main purposes -
  - Organization of large programs
  - Name uniqueness
  - Encapsulation at a higher level than classes

- A class can use -
  - All classes in its own package
  - All public classes from other packages
  
- Importing classes -
  - Using fully qualified names -
    - `java.time.LocalDate today = java.time.LocalDate.now()`
    - Always works, but tedious.
  - Using `import` -
    ```
    import java.time.*;                   // imports all classes in java.time package
    import java.time.LocalDate;           // specific class import

    LocalDate today = LocalDate.now();
    ```

> [!TIP]
> `import` is purely for convenience. The bytecode always uses fully qualified names.

> [!TIP]
> `java.lang`is imported automatically.

- __Module Imports__ (Java 25+) -
  - Java packages can be organized into modules.
  - Import all packages in a module -
    - Example - `import module java.xml`
    - Important module - 
      - `import module java.base`
      - Automatically imported in compact source files.
      - Includes `java.lang`, `java.util`, `java.time`, `java.io` etc.

- __Static Imports__ -
  - Allows importing static methods and fields -
    ```
    import static java.lang.System.*;
    err.println("Error");
    exit(0);
    ```

  - Enum Constants -
    ```
    import static java.time.DayOfWeek.*;

    var day = FRIDAY;
    ```

- If you don’t put a package statement in the source file - the classes will belong to the unnamed package (no package name).

> [!TIP]
> The implicitly declared class of a compact compilation unit (with methods declared outside a class) is always in the unnamed package.

- Package structure is important for JVM, not for compiler -
  - A class with `package com.mycompany` can compile even if it is not contained in a subdirectory `com/mycompany`
  - But, the JVM won't be able to execute it.
