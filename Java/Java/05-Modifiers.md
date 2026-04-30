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

  - This enables methods like `equals`, `compareTo`, or `copy` constructors to work.

> [!TIP]
> Variables with no access modifiers defaults to package access.

## Private Methods

- Instance fields are made `private` to protect data.
- Reasons to make methods private -
  - They are implementation details.
  - May require a special protocol or calling order.
  - Not meant to be used outside the class.

## Static Members

- Methods that do not operate on objects, eg - `Math.pow`.
  - Has no implicit (`this`) parameter.
  - A static method can access static field, but not an instance field.
  - It is legal to call static methods on the objects, but is not recommended.

- Classes such as `LocalDate` and `NumberFormat` use static factory methods that construct objects.
- Reasons to prefer a factory method over constructor -
  - Can’t give names to constructors
  - Can’t vary the return type of the constructed object
  - Share instances, eg - the call `Set.of()` yields the same instance of an empty set when you call it twice.

### __`main` method__
  
- Traditionally, `main` was a static method -
  - When a program starts, there aren’t any objects yet.

- Java 25+ - the `main` method no longer needs to be `static` and `public`, and it need not have a parameter of type `String[]`.

- Rules of `main` method -
  - If there is more than one `main` method - static `main` methods are preferred.
  - Methods with a `String[]` parameter are preferred over those with no parameters.
  - If `main` is not static, the class must have a non-private no-argument constructor - 
    - constructs an instance of the class and invokes the `main` method on it.

- Java 25+ - `main` method no longer needs to be declared inside a class. 
  - A source file with method declarations outside a class is called _compact compilation unit_. 
  - Implicitly declares a class whose name is derived from the source file.
  - Eg - if file name is `Application.java` then class name is likely to be `Application`.

## Final Instance Fields

- Rules for `final` instance fields -
  - Must be initialized during object construction (in every constructor).
  - Cannot be reassigned after the constructor completes.

- Example -
  ```
  class Employee {
    private final String name; 
  }
  ```

- Benefits -
  - Guarantees immutability of the reference.
  - Improves clarity and safety of your class design.

- `final` applies to the reference, not the object itself -
  ```
  private final StringBuilder evaluations;

  evaluations = new StringBuilder();          // initialized in constructor

  // cannot reassign evaluations to a new object, but the object itself can be mutated
  void giveGoldStar() {
    evaluations.append(LocalDate.now() + ": Gold star!\n");
  }
  ```
