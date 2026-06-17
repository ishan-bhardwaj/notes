## Inheritance

- Expresses _is-a_ relationship - subclass inherits methods and fields from superclass -
  ```java
  public class Manager extends Employee { }
  ```

> [!NOTE]
> You cannot extend a record, and a record cannot extend another class

- `super` keyword -
  - Call superclass method - `super.method1()`
  - Call superclass constructor - must be first statement (before Java 25; after Java 25, code before `super` is called an _early execution context_) -
    ```java
    public Manager(String name, double salary) {
      super(name, salary);
      bonus = 0;
    }
    ```

> [!TIP]
> If subclass constructor doesn't explicitly call `super(...)`, the superclass must have a no-arg constructor - it is invoked automatically before subclass construction

- __Polymorphism__ - object reference can refer to multiple actual types -
  ```java
  Employee e = new Manager("John", 500);    // valid
  ```

- __Static vs Dynamic Binding__ -
  - __Dynamic__ - method selected at runtime based on actual object type
  - __Static__ - method resolved at compile time - applies to `private`, `static`, `final` methods and constructors

> [!NOTE]
> Java does not support multiple inheritance

- Subclass array can be assigned to superclass array without cast -
  ```java
  Manager[] managers = new Manager[10];
  Employee[] staff = managers;              // OK
  ```

> [!NOTE]
> Arrays remember their element type - storing an incompatible reference throws `ArrayStoreException`

- __vtable (method table)__ - built per class at load time, merges class and inherited methods (overrides replace parent entries) - method calls resolved via fast table lookup at runtime

> [!TIP]
> Every class in Java implicitly extends `Object`

> [!WARNING]
> Overriding method must be at least as visible as the superclass method - Eg - if superclass method is `protected`, subclass must declare it `protected` or `public`

## Preventing Inheritance (`final`)

- `final` class cannot be extended -
  ```java
  public final class Executive extends Manager { }
  ```

  - `final` class methods are automatically `final`, but not fields

> [!TIP]
> `String` is a `final` class. Enumerations and records are always `final`.

- If a constructor calls a method, declare it `final` or `private` - otherwise it can be overridden and access a partially constructed subclass instance
  - Java 21+ - compile with `-Xlint:this-escape` to warn on such cases

- __Inlining__ - compiler optimizes short, non-overridden method calls by replacing them with the method body directly - avoids branching, improving CPU pipeline efficiency
  - JIT compiler inlines short, frequently called, non-overridden methods
  - If a new subclass overrides an inlined method, JIT undoes the inlining (rare, but takes time)

## Casting

- Forcing conversion from one type to another -
  ```java
  Employee e = new Manager();
  Manager m = (Manager) e;        // explicit cast
  ```

- Failed cast at runtime throws `ClassCastException` - check first with `instanceof` -
  ```java
  if (e instanceof Manager) {
    m = (Manager) e;
  }
  ```

> [!NOTE]
> `x instanceof C` returns `false` if `x` is `null` - never throws an exception

- __Pattern matching for `instanceof`__ (Java 16+) - declare variable inline -
  ```java
  if (e instanceof Manager m) {
    m.setBonus(5000);
  }
  ```

- Can use pattern variable immediately in same expression -
  ```java
  if (e instanceof Executive exec && exec.getTitle().length() >= 20)   // OK
  if (e instanceof Executive exec || exec.getTitle().length() >= 20)   // ERROR - compile-time error
  ```

- Conditional operator -
  ```java
  String title = e instanceof Executive exec ? exec.getTitle() : "";
  ```

- __Record pattern__ -
  ```java
  if (p instanceof Point(var a, var b)) distance = Math.hypot(a, b);
  if (p instanceof Point(var a, _)) distance = Math.abs(a);           // underscore for unused (Java 22+)
  ```

- Nested record patterns -
  ```java
  if (c instanceof Circle(Point(var a, var b), var r)) { ... }
  ```

### `equals` Method

- Default `Object.equals` checks reference identity (`==`)
- Override to compare field values
- When overriding in subclass - call `super.equals` first

> [!TIP]
> `Objects.equals(a, b)` - `true` if both `null`, `false` if only one `null`, else `a.equals(b)`

> [!NOTE]
> Records auto-generate `equals` comparing all components

- `equals` contract -
  - __Reflexive__ - `x.equals(x)` is `true`
  - __Symmetric__ - `x.equals(y)` iff `y.equals(x)`
  - __Transitive__ - `x.equals(y)` and `y.equals(z)` → `x.equals(z)`
  - __Consistent__ - repeated calls return same value if objects unchanged
  - `x.equals(null)` is always `false`

> [!TIP]
> Array fields - use `Arrays.equals` for 1D, `Arrays.deepEquals` for multidimensional

### `hashCode` Method

- Integer derived from object - should be widely scattered across distinct objects
- Must override `hashCode` if you override `equals` - objects that are `equals` must have same `hashCode`
- Best practice -
  ```java
  public int hashCode() {
    return Objects.hash(name, salary, hireDay);   // null-safe, combines fields
  }
  ```
- Array fields - `Arrays.hashCode` for 1D, `Arrays.deepHashCode` for multidimensional

> [!NOTE]
> Zero-length array has hash code `1` to distinguish from `null`

> [!NOTE]
> Records auto-generate `hashCode` from component hash codes

### `toString` Method

- Returns string representation - auto-invoked when object concatenated with `+`
- Default `Object.toString` returns class name + hash code
- Arrays inherit `Object.toString` - use `Arrays.toString` for 1D, `Arrays.deepToString` for multidimensional instead -
  ```java
  int[] arr = { 2, 3, 5 };
  "" + arr;                       // "[I@1a46e30" - not useful
  Arrays.toString(arr);           // "[2, 3, 5]"
  ```

> [!NOTE]
> Records auto-generate `toString` listing class name and all field names/values

> [!NOTE]
> If you override `equals`, `hashCode`, or `toString` but need original behavior -
> - Use `==` instead of `equals`
> - Use `System.identityHashCode(obj)` or `Objects.identityToString(obj)`

## Autoboxing

- Primitive types have wrapper class counterparts - `Integer`, `Long`, `Float`, `Double`, `Short`, `Byte`, `Character`, `Boolean`
  - First six extend `Number`
  - Immutable and `final` - cannot subclass them

- __Autoboxing__ - primitive auto-converted to wrapper -
  ```java
  list.add(3);                    // compiled as list.add(Integer.valueOf(3))
  ```

- __Unboxing__ - wrapper auto-converted to primitive -
  ```java
  int n = list.get(i);            // compiled as list.get(i).intValue()
  ```

- Works in arithmetic too -
  ```java
  Integer n = 3;
  n++;                            // unbox → increment → rebox
  ```

> [!WARNING]
> Never compare wrapper objects with `==` and never use them as locks

> [!NOTE]
> Mixed `Integer` and `Double` in conditional - `Integer` is unboxed, promoted to `double`, reboxed as `Double` -
> ```java
> Integer n = 1; Double x = 2.0;
> true ? n : x;                    // prints 1.0
> ```

> [!TIP]
> Boxing/unboxing is handled by the compiler, not the JVM - compiler inserts the necessary bytecode calls

- `java.lang.Integer` key API -
  | Method | Description |
  | --- | --- |
  | `intValue()` | Returns primitive `int` |
  | `static parseInt(String s)` | Parses base-10 string to `int` |
  | `static parseInt(String s, int radix)` | Parses string in given base |
  | `static valueOf(String s)` | Returns `Integer` from base-10 string |
  | `static toString(int i)` | Returns base-10 string |
  | `static toString(int i, int radix)` | Returns string in given base |

## Varargs Methods

- Method accepting variable number of arguments -
  ```java
  public static double max(double... values)    // same as double[]
  double m = max(3.1, 40.4, -5);               // compiler passes new double[]{ 3.1, 40.4, -5 }
  ```

## Abstract Classes

- Class with one or more abstract methods must itself be `abstract` -
  ```java
  public abstract class Person {
    public abstract String getDescription();
  }
  ```
- Can have fields and concrete methods alongside abstract methods
- Can be declared `abstract` even with no abstract methods

## Enumeration Classes

  ```java
  public enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }
  ```

- Type is actually a class with a fixed number of instances - use `==` to compare, never `equals`
- All enums extend `Enum` (abstract class)
- Key inherited methods -
  - `name()` - returns constant name as string - `final`, cannot override
  - `toString()` - returns `name()` by default - can override
  - `ordinal()` - zero-based position in declaration
  - `static valueOf(Class, String)` - returns constant by name
  - `static values()` - returns array of all constants

- Can add constructors, fields, and methods - constructor is always `private` -
  ```java
  public enum Size {
    SMALL("S"), MEDIUM("M"), LARGE("L"), EXTRA_LARGE("XL");

    private final String abbreviation;
    Size(String abbreviation) { this.abbreviation = abbreviation; }
    public String getAbbreviation() { return abbreviation; }
  }
  ```

- Simple name usable without qualification in -
  - Methods inside the enum
  - `switch` cases where selector is that enum type
  - After `import static`

> [!WARNING]
> Cannot statically import constants from an enum in the default package (no `package` declaration)

- Enum with no instances - useful as a utility class -
  ```java
  public enum FileUtils {
    public static String extension(String filename) { ... }
  }
  ```

- Cannot use `switch (this)` inside enum constructor - ordinal array is built after instances are constructed, so it is only safe inside methods

## Sealed Classes

- Controls which classes may inherit from it -
  ```java
  public abstract sealed class JSONValue permits JSONArray, JSONNumber, JSONString { }
  ```

- Permitted subclasses must be -
  - Accessible (not private/nested)
  - In the same package (or same module if using modules)
  - Each must declare itself `sealed`, `final`, or `non-sealed`

- `permits` clause can be omitted - then all direct subclasses must be in the same file
- Key benefit - compiler checks exhaustiveness in `switch`, no `default` needed -
  ```java
  return switch (this) {
    case JSONArray _  -> "array";
    case JSONNumber _ -> "number";
    case JSONNull _   -> "null";
  };
  ```

- Records and enums can implement a sealed interface, but cannot extend them

## Pattern Matching

- `switch` with pattern matching on types -
  ```java
  String description = switch (e) {
    case Executive exec -> "Executive: " + exec.getTitle();
    case Manager _      -> "Manager";       // underscore if variable unused (Java 22+)
    default             -> "Employee: " + e.getSalary();
  };
  ```