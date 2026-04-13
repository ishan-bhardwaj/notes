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

> [!TIP]
> Every class can have a `main` method. That can be handy for adding demonstration code to a class. If that class is part of another program, its `main` method is not executed.
