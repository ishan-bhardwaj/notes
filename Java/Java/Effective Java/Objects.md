## Consider static factory methods instead of constructors

- Advantages -
  - They have names.
  - Can cache or reuse instances -
    - such classes are called _instance-controlled_ classes (strict control over instance creation).
    - Instance-controlled classes can guarantee -
      - Singleton
      - Non-instantiable class
      - No duplicate equal instances (`a.equals(b)` ⇔ `a == b`)
      - Enums naturally provide this guarantee.
  - Can return any subtype of the declared return type.
    - Static factory can return -
      - Subtypes
      - Hidden implementation classes
  - Returned type can vary based on input parameters or library/version changes.
  - Implementation class may not exist yet -
    - The return type exists (interface), but implementation may be added later.
    - Enables Service Provider Frameworks (SPF).
    - Eg - JDBC API - _Driver_ is the service provider interface.

- Disadvantages -
  - Classes without public or protected constructors cannot be subclassed -
    - Encourages composition over inheritance.
  - Harder to find in API documentation.

- Common naming conventions for static factories -

| Name              | Purpose                                   | Example                                              |
| ----------------- | ----------------------------------------- | ---------------------------------------------------- |
| **from()**        | Type conversion from another type         | `Date date = Date.from(instant);`                    |
| **of()**          | Aggregate multiple values into one object | `Set<Integer> set = Set.of(1,2,3);`                  |
| **valueOf()**     | Verbose alternative to from/of            | `Integer i = Integer.valueOf("42");`                 |
| **getInstance()** | Returns instance (may be cached/reused)   | `StackWalker walker = StackWalker.getInstance();`    |
| **instance()**    | Same as getInstance (shorter)             | `Logger logger = Logger.instance();`                 |
| **newInstance()** | Always creates new object                 | `Object arr = Array.newInstance(String.class, 10);`  |
| **create()**      | Always creates new object (readable alt)  | `User user = User.create("Ishan");`                  |
| **getType()**     | Factory in another class (may reuse)      | `FileStore fs = Files.getFileStore(path);`           |
| **newType()**     | Factory in another class (always new)     | `BufferedReader br = Files.newBufferedReader(path);` |
| **type()**        | Short alternative to getType/newType      | `List<String> list = Collections.list(enumeration);` |

## Consider a builder when faced with many constructor parameters

- Alternatives -
  - Telescopic constructor pattern -
    - one constructor with only the required parameters.
    - another with a single optional parameter.
    - a third with two optional parameters.
    - and so on.
  - JavaBeans pattern -
    - call a parameterless constructor to create the object.
    - call setter methods to set each parameter.

- __Builder pattern__ -
  - call the constructor or static factory with all the required parameters and get a _builder_ object.
  - call setter-like methods on the builder object to set each optional parameter.
  - finally, call a parameterless `build` method to generate the object.

- Example -

```
class User {
    private final String name;
    private final int age;
    private final String city;

    private User(Builder b) {
        this.name = b.name;
        this.age = b.age;
        this.city = b.city;
    }

    static class Builder {
        private final String name;                    // required
        private int age = 0;                          // optional
        private String city = "NA";                   // optional

        Builder(String name) { this.name = name; }

        Builder age(int age) { this.age = age; return this; }
        Builder city(String city) { this.city = city; return this; }

        User build() { return new User(this); }
    }
}
```

- Usage -

```
var user = new User.Builder("Ishan")
            .age(30)
            .city("London")
            .build();
```

- The Builder pattern is well suited to class hierarchies -
  - __Covariant return typing__ - A subclass method can return a more specific type than the superclass method, allowing clients to use builders without casting.
  - Base class with generic builder -
    ```
    class Pizza {
      abstract static class Builder {
        abstract Pizza build();

        Builder addCheese() {
          System.out.println("Cheese added");
          return this;
        }
      }
    }
    ```

  - Subclass returns subtype in `build()` -
    ```
    class NyPizza extends Pizza {

      static class Builder extends Pizza.Builder {
        @Override
        NyPizza build() {                               // covariant return (subtype)
            return new NyPizza();
        }

        @Override
        Builder addCheese() {                           // returns subclass Builder
            super.addCheese();
            return this;
        }
      }
    }
    ```

  - Usage -
    ```
    var pizza = new NyPizza.Builder()
                  .addCheese()
                  .build();                             // returns NyPizza directly - no casting needed
    ```

## Enforce the singleton property with a private constructor or an enum type

