# Object-oriented Programming

- __Encapsulation / Information Hiding__ - bundle data and methods inside an object and hide internal implementation from the outside.

## Classes & Objects

- _Class_ defines a new data type which can be used to create _objects_ of that type.
- Methods and variables defined within a class are called _members_ of the class.
- Three object characteristics -
  - Behavior - what the object can do (its methods).
  - State - the data it holds (instance variables).
  - Identity - its unique reference that distinguishes it from other objects.

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

  // or,
  Employee john = new Employee("John", 30);

  // or,
  var john = new Employee("John", 30);
  ```

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

- Before Java 25, the call to the other constructor had to be the first statement of the constructor body. 
  - This restriction has now been removed.
  - However there are some restrictions on what can happen between the start of a constructor and the call of another constructor. This phase is called the _early construction context_.

- In the early construction context, you may not -
  - Read any instance variable.
  - Write any instance variable that has an explicit initialization.
  - Invoke any methods on this.
  - Pass this to any other methods.

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

- Happens when the class is loaded, not when objects are created.

## Java Object Creation - Execution Order

- __Once per class__ -
  - Load class bytecode
  - Verify bytecode
  - Prepare static memory (assign default values to static fields)
  - Resolve symbolic references
  - Execute static field initializers
  - Execute static initializer blocks

- Every time an object is created (`new`) -
  - Allocate memory on heap
  - Assign default values to all instance fields
  - Execute instance field initializers (in textual order)
  - Execute instance initializer blocks (in textual order)
  - Execute constructor body
  - Return object reference

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

## Handling Null Fields in Classes
  
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
      Employee(String n, double s) {
        name = Objects.requireNonNullElse(n, "unknown");
        // ... other initializations
      }
      ```

  - “Tough Love” Approach -
    - Reject `null` arguments with an exception.
    - Example - using `Objects` utility class -
      ```
      Employee(String n, double s) {
        name = Objects.requireNonNull(n, "The name cannot be null");
        // ... other initializations
      }
      ```

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

> [!TIP]
> Local variables are not initialized with their default values - you must explicitly initialise them.
> Instance variables are automatically initialised to their default value (`0`, `false`, `null` etc), if not initialised explicitly.