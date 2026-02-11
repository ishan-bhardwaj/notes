# Inheritance

- Inheritance signifies _"is-a"_ relationship, eg - `Manager` _is-a_ `Employee` -
```
public class Manager extends Employee {
    // added methods and fields
}
```

- The existing class (`Employee`) is called the superclass, base class, or parent class. 
- The new class (`Manager`) is called the subclass, derived class, or child class. 

> [!NOTE]
> You cannot extend a record, and a record cannot extend another class.

- `super` keyword -
  - To call super class methods and non-private fields, use `super` keyword - `super.method1()` - calls the `method1()` of superclass.
  - To invoke superclass constructor in subclass constructor -
    ```
    public Manager(String name, double salary) {
      super(name, salary);              // calls the constructor of `Employee` superclass.
      bonus = 0;
    }
    ```

    - Before Java 25, the call to super had to be the first statement in the constructor for the subclass. Now, the code between the start of the constructor and the call to super is an _early execution context_.

> [!TIP]
> When a subclass object is constructed without an explicit invocation of a superclass constructor, the superclass must have a no-argument constructor. That constructor is invoked prior to the subclass construction.

- Polymorphism is the ability of an object reference to refer to multiple actual types, eg - 
```
Manager m = new Manager("John", 500);

// or
Employee e = new Manager("John", 500);
```

- _Dynamic binding_ is the process of automatically selecting the appropriate method to execute at runtime based on the actual object.

- If the method is `private`, `static`, `final`, or a constructor, then the compiler knows exactly which method to call. This is called _static binding_.

> [!NOTE]
>  Java does not support multiple inheritance.

- In Java, arrays of subclass references can be converted to arrays of superclass references without a cast. For example, consider this array of managers -
```
Manager[] managers = new Manager[10];
Employee[] staff = managers;              // OK  
```

> [!NOTE]
> All arrays remember the element type with which they were created, and they monitor that only compatible references are stored into them. For example, the array created as `new Manager[10]` remembers that it is an array of managers. Attempting to store an `Employee` reference causes an `ArrayStoreException`.

- When the program runs and uses dynamic binding to call a method, the virtual machine must call the version of the method that is appropriate for the actual type of the object to which `x` refers. It would be time-consuming to carry out this search every time a method is called.
- Instead, the virtual machine precomputes a method table for each class. The method table lists all method signatures and the actual methods to be called.
- The virtual machine can build the method table after loading a class, by combining the methods that it finds in the class file with the method table of the superclass.
- When a method is actually called, the virtual machine simply makes a table lookup.

> [!TIP]
> Every class in Java has a superclass `Object`.

> [!WARNING]
> When you override a method, the subclass method must be at least as visible as the superclass method. In particular, if the superclass method is `public`, the subclass method must also be declared `public`. 
>
> It is a common error to accidentally omit the `public` specifier for the subclass method. The compiler then complains that you try to supply a more restrictive access privilege.

## Preventing Inheritance

- Classes that cannot be extended are called final classes, and you use the _final_ modifier in the definition of the class to indicate this. 
- Example -
```
public final class Executive extends Manager {
  ...
}
```

- All methods in a `final` class are automatically `final`.
- Make specific method `final` -
```
public final String getName() {
  return name;
}
```

- Fields can also be declared as `final`. A `final` field cannot be changed after the object has been constructed. However, if a class is declared `final`, only the methods, not the fields, are automatically `final`.

> [!TIP]
> The `String` class is a `final` class.

- If you call a method in a constructor, you should declare it as `final` or `private`. Otherwise, it can be overridden in a subclass, and it can access a partially constructed subclass instance.

> [!TIP]
> Since Java 21, if you compile with the `-Xlint:this-escape` option, the compiler issues a warning when the constructor of a public class calls a method that is not `final` or `private`.

- Some programmers used the `final` keyword hoping to avoid the overhead of dynamic binding -
  - If a method is not overridden, and it is short, then a compiler can optimize the method call away— a process called _inlining_. 
  - For example, inlining the call `e.getName()` replaces it with the field access `e.name`.
  - This is a worthwhile improvement — CPUs hate branching because it interferes with their strategy of prefetching instructions while processing the current one. 
  - However, if `getName` can be overridden in another class, then the compiler cannot inline it because it has no way of knowing what the overriding code may do.
  - The just-in-time compiler knows exactly which classes extend a given class, and it can check whether any class actually overrides a given method. 
  - If a method is short, frequently called, and not actually overridden, the just-in-time compiler can inline it. 
  - But if the virtual machine loads another subclass that overrides an inlined method, then the optimizer must undo the inlining - which takes time, but it happens rarely.

> [!TIP]
> Enumerations and records are always `final` — you cannot extend them.

## Casting

- Castin is the process of forcing a conversion from one type to another - you may need to convert an object reference from one class to another. 
- Example -
```
Employee e = new Manager();
Manager m = (Manager) e;
```

- If the cast is not possible then when the program runs, the Java runtime system generates a `ClassCastException`.
- Thus, it is good programming practice to find out whether a cast will succeed before attempting it - 
```
if (e instanceof Manager) {
  m = (Manager) e
}
```

- Finally, the compiler will not let you make a cast if there is no chance for the cast to succeed, eg - `String c = (String) e` - is a compile-time error because `String` is not a subclass of `Employee`.

> [!NOTE]
> The test `x instanceof C` does not generate an exception if `x` is `null` - it simply returns `false`.

- __Pattern Matching for `instanceof`__ - 
  - As of Java 16, you can declare the subclass variable right in the `instanceof` test -
  ```
  if (e instanceof Manager m) {
    m.setBonus(5000);
  }
  ```

> [!NOTE]
> In most situations in which you use instanceof, you need to apply a subclass method. Then use this “pattern-matching” form of instanceof instead of a cast.
>
> Example -
> ```
> Manager m = ... ;
> if (m instanceof Employee e)       // ERROR - Of course it's an Employee
> ```
>
> However, The equally useless `if (m instanceof Employee)` is allowed, for backward compatibility with Java 1.0.

- When an `instanceof` pattern introduces a variable, you can use it right away, in the same expression -
  ```
  if (e instanceof Executive exec && exec.getTitle().getLength() >= 20)
  ```

  - This works because the right-hand side of an `&&` expression is only evaluated if the left-hand side is `true`.
  - However, the following is a compile-time error -
  ```
  if (e instanceof Manager exec || exec.getTitle().getLength() >= 20)     // ERROR
  ```

- Another example with the conditional operator -
```
String title = e instanceof Executive exec ? exec.getTitle() : "";
```

- Record pattern -
```
record Point(double x, double y) {}
Point p = ... ;
if (p instanceof Point(var a, var b)) distance = Math.hypot(a, b);
```

- You can also specify explicit types for the introduced variables -
```
if (p instanceof Point(double a, double b)) ... ;
```

- Record patterns can also be nested -
```
record Circle(Point center, double radius) {}
Circle c = ... ;
if (c instanceof Circle(Point(var a, var b), var r)) ... ;
```

- Since Java 22, you can denote unmatched parts in a record pattern with an underscore -
```
if (p instanceof Point(var a, _)) distance = Math.abs(a);
```

> [!TIP]
> Protected - accessible in the package and all subclasses.

## `Object` Superclass

- Every class in Java extends `Object`, however, you never have to write - `public class Employee extends Object`.
- Only the values of primitive types (numbers, characters, and boolean values) are not objects.

### The `equals` Method
  - The `equals` method, as implemented in the `Object` class, determines whether two object references are identical.

> [!TIP]
> `Objects.equals(a, b)` is `true` if both arguments are `null`, `false` if only one is `null`, and `a.equals(b)` otherwise.

- When you define the equals method for a subclass, first call `equals` on the superclass. If that test doesn’t pass, then the objects can’t be equal - `super.equals` checked that `this` and `otherObject` belong to the same class.

> [!NOTE]
> Records automatically define an `equals` method that compares the components. Two record instances are equal when the corresponding component values are equal.

- The Java Language Specification requires that the equals method has the following properties -
  - It is reflexive - For any non-`null` reference `x`, `x.equals(x)` should return `true`.
  - It is symmetric - For any references `x` and `y`, `x.equals(y)` should return `true` if and only if `y.equals(x)` returns `true`.
  - It is transitive - For any references `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` should return `true`.
  - It is consistent - If the objects to which `x` and `y` refer haven’t changed, then repeated calls to `x.equals(y)` return the same value.
  - For any non-`null` reference `x`, `x.equals(null)` should return `false`.

> [!TIP]
> If you have fields of array type, you can use the static `Arrays.equals` method to check that the corresponding array elements are equal. Use the `Arrays.deepEquals` method for multidimensional arrays.

### The `hashCode` Method

- A hash code is an integer that is derived from an object. 
- Hash codes should be scrambled — if x and y are two distinct objects, there should be a high probability that `x.hashCode()` and `y.hashCode()` are different.
- The `String` class uses the following algorithm to compute the hash code -
```
int hash = 0;
for (int i = 0; i < length(); i++)
    hash = 31 * hash + charAt(i);
```

- The `hashCode` method is defined in the `Object` class. Therefore, every object has a default hash code, called the _identity hash code_. How that hash code is determined depends on the virtual machine.

- If you redefine the `equals` method, you will also need to redefine the `hashCode` method for objects that users might insert into a hash table.
- The `hashCode` method should return an integer (which can be negative). Just combine the hash codes of the instance fields so that the hash codes for different objects are likely to be widely scattered.
- However, you can do better -
  - First, use the `null`-safe method `Objects.hashCode` - it returns `0` if its argument is `null` and the result of calling `hashCode` on the argument otherwise. 
  - Also, use the static `Double.hashCode` method to avoid creating a `Double` object.

- In many cases, you can simply call `Objects.hash` with all fields. It will combine the hash codes of its arguments. Then the `Employee.hashCode` method is simply -
```
public int hashCode() {
    return Objects.hash(name, salary, hireDay);
}
```

- In case of arrays, use the static `Arrays.hashCode` method which combines the hash codes of the array elements. There are overloads for primitive type arrays and arrays of objects. For multi-dimensional arrays, use `Arrays.deepHashCode`.

> [!NOTE]
> A zero-length array has hash code `1`, to distinguish it from `null`.

- A record type automatically provides a `hashCode` method that derives a hash code from the hash codes of the component values.
- Your definitions of equals and hashCode must be compatible - If `x.equals(y)` is `true`, then `x.hashCode()` must return the same value as `y.hashCode()`.

### The `toString` Method

- Returns a string representing the value of the object.
- Whenever an object is concatenated with a string by the “+” operator, the Java compiler automatically invokes the `toString` method to obtain a string representation of the object.
- The `Object` class defines the toString method to print the class name and the hash code of the object.
- Arrays inherit the `toString` method from `Object` that adds array type -
  ```
  int[] luckyNumbers = { 2, 3, 5, 7, 11, 13 };
  String s = "" + luckyNumbers;
  ```

  - yields  the string `"[I@1a46e30"` - the prefix `[I` denotes an array of integers. 
  - The remedy is to call the static `Arrays.toString` method instead - 
  ```
  String s = Arrays.toString(luckyNumbers);
  ```

  - yields the string `"[2, 3, 5, 7, 11, 13]"`.

  - To correctly print multidimensional arrays, use `Arrays.deepToString`.

- For record types, a `toString` method is already provided - it simply lists the class name and the names and stringified values of the fields.

> [!NOTE]
> If you override `equals`, `hashCode` or `toString` methods, but still need to access the unmodified behavior, use - 
>
>   - Compare with `==` instead of `equals`.
>   - Call `System.indentityHashCode(obj)` or `Objects.identityToString(obj)` to get what would have been returned if you had not overriden `hashCode` or `toString`.

## Autoboxing

- All primitive types have class counterparts. These kinds of classes are usually called wrappers.
- Example - a class `Integer` corresponds to the primitive type `int`.
- The wrapper classes have obvious names  `Integer`, `Long`, `Float`, `Double`, `Short`, `Byte`, `Character`, and `Boolean`. The first six inherit from the common superclass `Number`.
- The wrapper classes are immutable — you cannot change a wrapped value after the wrapper has been constructed. 
- They are also `final`, so you cannot subclass them.

- Suppose we want an array list of integers -
  - The type argument inside the angle brackets cannot be a primitive type i.e. tt is not possible to form an `ArrayList<int>`.
  - But we can declare an array list of `Integer` objects - `var list = new ArrayList<Integer>()`
  - Now, to add an element of type `int` - `list.add(3)` - is automatically translated to - `list.add(Integer.valueOf(3))` - this conversion is called _autoboxing_.
  - Similarly, when you assign an `Integer` object to an `int` value, it is automatically unboxed i.e. the compiler translates `int n = list.get(i)` into `int n = list.get(i).intValue()`.

- Automatic boxing and unboxing also takes place in arithmetic expressions, for eg, you can apply the increment operator to a wrapper reference -
  ```
  Integer n = 3;
  n++;
  ```

  - The compiler automatically inserts instructions to unbox the object, increment the resulting value, and box it back.

> [!WARNING]
> Never rely on the identity of wrapper objects. Don’t compare them with `==` and don’t use them as locks.

> [!NOTE]
> For the special values `-0.0` and `Double.NaN`, the `equals` method of the `Double` class is more convenient than the `==` operator. 
>
> While `0.0 == -0.0`, the result of `Double.valueOf(-0.0).equals(Double.valueOf(0.0))` is `false`. 
>
> Moreover, `Double.NaN != Double.NaN`, but `Double.valueOf(Double.NaN).equals(Double.valueOf(Double.NaN))` is `true`.

- If you mix `Integer` and `Double` types in a conditional expression, then the `Integer` value is unboxed, promoted to `double`, and boxed into a `Double` -
```
Integer n = 1;
Double x = 2.0;
IO.println(true ? n : x);                   // prints 1.0
```

> [!TIP]
> The compiler, not the virtual machine, is in charge of boxing and unboxing. The compiler inserts the necessary calls when it generates the bytecodes of a class. The virtual machine simply executes those bytecodes.

- __`java.lang.Integer` API -

| Method                                        | Type     | Description                                                                               |
| --------------------------------------------- | -------- | ----------------------------------------------------------------------------------------- |
| `int intValue()`                              | Instance | Returns the value of this `Integer` as a primitive `int` (overrides `Number.intValue()`). |
| `static String toString(int i)`               | Static   | Returns a `String` representation of `i` in base 10.                                      |
| `static String toString(int i, int radix)`    | Static   | Returns a `String` representation of `i` in the specified base (`radix`).                 |
| `static int parseInt(String s)`               | Static   | Parses the string `s` as a **base-10** integer and returns a primitive `int`.             |
| `static int parseInt(String s, int radix)`    | Static   | Parses the string `s` as an integer in the specified base (`radix`).                      |
| `static Integer valueOf(String s)`            | Static   | Returns an `Integer` object representing the **base-10** value of `s`.                    |
| `static Integer valueOf(String s, int radix)` | Static   | Returns an `Integer` object representing the value of `s` in the specified base.          |

- `java.text.NumberFormat` API -

| Method                   | Type     | Description                                                                                       |
| ------------------------ | -------- | ------------------------------------------------------------------------------------------------- |
| `Number parse(String s)` | Instance | Parses the string `s` and returns a `Number` (can be `Long`, `Double`, etc., depending on input). |

## Varargs Methods

- Methods that can be called with a variable number of arguments.
- For example -
  ```
  public static double max(double... values)
  ```

  - The ellipsis `...` is part of the Java code. It denotes that the method can receive an arbitrary number of objects.
  - The `double...` parameter type is exactly the same as `double[]`.
  - Call the function - `double m = max(3.1, 40.4, -5)`
  - The compiler passes a `new double[] { 3.1, 40.4, -5 }` to the `max` function.

- You can even declare the main method as -
```
public static void main(String... args)
```

## Abstract Classes

- A class with one or more abstract methods must itself be declared abstract -
```
public abstract class Person {
  public abstract String getDescription();
}
```

- In addition to abstract methods, abstract classes can have fields and concrete methods. 
- A class can even be declared as `abstract` though it has no abstract methods.

## Enumeration Classes

```
public enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }
```

- Enumerations can also have methods.
- When you refer to these constants, qualify them with the class name, such as `Size.SMALL`.
- You can use the simple name in three situations -
  - Inside the methods of an enumeration, you can use the simple name.
  - In a `case` of a `switch` whose selector has an enumerated type, you don’t need the qualification -
    ```
    Size s = ... ;
    String abbrev = switch (s) {
      case SMALL -> "S";
      ...
    };
    ```

  - You can statically import all constants of an enumeration -
    ```
    import com.example.util.Size;
    import static com.example.util.Size.*;

    Size s = SMALL;
    ```

> [!WARNING]
> You cannot statically import constants from an enumeration in the default package. For example, if `Size` is in the default package, you cannot use - `import static Size.*`

- The type defined by an `enum` declaration is actually a class. The class has exactly four instances — it is not possible to construct new objects. Therefore, you never need to use `equals` for values of enumerated types. Simply use `==` to compare them.

- All enumerated types are subclasses of the abstract class Enum. They inherit a number of methods from that class.
  - The most useful one is `name`, which returns the name of the enumerated constant, for eg - `Size.SMALL.toString()` returns the string `"SMALL"`.
  - By default, the `toString` method returns `name`. However, you can override `toString`, whereas `name` is `final`.
  - The converse of `name` is the static `valueOf` method, for eg - `Size s = Enum.valueOf(Size.class, "SMALL")` - sets `s` to `Size.SMALL`.
  
- Each enumerated type has a static `values` method that returns an array of all values of the enumeration, for eg - `Size[] values = Size.values()` - returns the array with elements `Size.SMALL`, `Size.MEDIUM`, `Size.LARGE`, and `Size.EXTRA_LARGE`.

- The `ordinal` method yields the position of an enumerated constant in the `enum` declaration, counting from zero. For example, `Size.MEDIUM.ordinal()` returns `1`.

- You add constructors, methods, and fields to an enumerated type, for eg -
```
public enum Size {
  SMALL("S"), MEDIUM("M"), LARGE("L"), EXTRA_LARGE("XL");

  private final String abbreviation;

  Size(String abbreviation) { this.abbreviation = abbreviation; }
    // automatically private

  public String getAbbreviation() { return abbreviation; }
}
```

> [!NOTE]
> When an `enum` has fields or methods, you must terminate the list of constants with a semicolon.

- The constructor of an enumeration is always `private`. You can omit the `private` modifier. It is a syntax error to declare an `enum` constructor as `public` or `protected`.

- An enumeration with no instances cannot be instantiated. This is useful for indicating that a class is a utility class, with only static methods -
```
public enum FileUtils {
    ; // No instances

    public static String extension(String filename) {
        int n = filename.lastIndexOf(".");
        return n >= 0 ? filename.substring(n + 1) : "";
    }
}
```

- The constructor of an enumeration class cannot invoke a `switch` on the instance that is being constructed -
```
public enum Size {
    SMALL, MEDIUM, LARGE, EXTRA_LARGE;
    private final boolean largish;
    Size() {
        largish = switch (this) {               // ERROR
            case LARGE, EXTRA_LARGE -> true;
            default -> false;
        };
    }
}
```

  - This limitation is caused by the implementation strategy for `switch` on `enum`. 
  - Each `switch` consults an array that maps the ordinal values of the `enum` constants, as currently defined, to the ordinal values of the time when the switch was compiled.
  - These arrays can only be built after the enum instances are constructed.
  - In a method, such a `switch` works fine.

> [!NOTE]
> The `Enum` class has a type parameter. For example, the enumerated type `Size` actually extends `Enum<Size>`. The type parameter is used in the `compareTo` method.

- __`java.lang.Enum<E>` API__ -

| Method                                                                  | Type     | Description                                                                                                                |
| ----------------------------------------------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------- |
| `static <T extends Enum<T>> T valueOf(Class<T> enumClass, String name)` | Static   | Returns the enum constant of the specified enum class with the given name. Throws `IllegalArgumentException` if not found. |
| `String name()`                                                         | Instance | Returns the **exact name** of this enum constant as declared in the enum.                                                  |
| `int ordinal()`                                                         | Instance | Returns the **zero-based position** of this enum constant in the enum declaration.                                         |
| `int compareTo(E other)`                                                | Instance | Compares this enum with another based on their **declaration order**.                                                      |

## Sealed Classes

- A sealed class controls which classes may inherit from it.
- Example -
```
public abstract sealed class JSONValue
        permits JSONArray, JSONNumber, JSONString, JSONBoolean, JSONObject, JSONNull {
    ...
}
```

- It is an error to define a nonpermitted subclass -
```
public class JSONComment extends JSONValue { ... }            // Error
```

- The permitted subclasses of a sealed class must be accessible. They cannot be private classes that are nested in another class, or package-visible classes from another package.
- For permitted subclasses that are public, they must be in the same package as the sealed class. However, if you use modules, then they must only be in the same module.

- A sealed class can be declared without a permits clause. Then all of its direct subclasses must be declared in the same file.
- A file can have at most one public class, so this arrangement appears to be only useful if the subclasses are not for use by the public.

- An important motivation for sealed classes is compile-time checking. Consider this method of the `JSONValue` class, which uses a `switch` expression with pattern matching -
  ```
  public String type() {
    return switch (this) {
      case JSONArray _ -> "array";
      case JSONNumber _ -> "number";
      case JSONString _ -> "string";
      case JSONBoolean _ -> "boolean";
      case JSONObject _ -> "object";
      case JSONNull _ -> "null";
      // No default needed here
    };
  }
  ```

  - The compiler can check that no default clause is needed since all direct subclasses of `JSONValue` occur as cases.

- A subclass of a sealed class must specify whether it is `sealed`, `final`, or open for subclassing. In the latter case, it must be declared as `non-sealed`.

- Sealed interfaces work exactly the same as sealed classes, controlling the direct subtypes.
- Records and enumerations can implement interfaces, but they cannot extend classes.

## Pattern Matching

```
Employee e = ... ;
String description = switch (e) {
    case Executive exec -> "An executive with a title of " + exec.getTitle();
    case Manager _ -> "A manager who deserves a bonus";
    default -> "A lowly employee with a salary of " + e.getSalary();
};
```

- Starting with Java 22, you can use an underscore if you don’t need the variable, eg - `case Manager _`.
