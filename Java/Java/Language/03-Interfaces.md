# Interfaces

### The Interface Concept

- __Interface__ — a set of requirements (method signatures) for classes that want to conform to it; not a class
- All interface methods are automatically `public`; all fields are automatically `public static final`
- Interfaces never have instance fields
- Before Java 8, all methods were abstract — since Java 8, static, private, and default methods are allowed

### Implementing an Interface

- Two steps:
    - Declare intent with `implements`
    - Supply definitions for all abstract methods
- Implementation must explicitly declare methods `public` (else compiler assumes package access — more restrictive than interface)
- Eg - `class Employee implements Comparable<Employee>`

### `Comparable<T>`

- `int compareTo(T other)` — returns negative if `this < other`, 0 if equal, positive if `this > other`
- Antisymmetry contract: `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))` must hold
- `compareTo` should be consistent with `equals` — notable exceptions: `BigDecimal`, `StringBuilder`
- Use `Double.compare(x, y)` for floats — subtraction trick unsafe due to rounding and `-0.0`/`NaN` edge cases:
    - `-0.0 < 0.0` evaluates `false` with `<` but `Double.compare(-0.0, 0.0)` returns `-1`
    - `Double.NaN == Double.NaN` is `false`; `Double.compare(NaN, NaN)` returns `0`
    - `Double.compare(POSITIVE_INFINITY, NaN)` returns `-1` — `NaN` is deemed largest

### Inheritance and `compareTo`

- `Manager extends Employee` inherits `Comparable<Employee>`, not `Comparable<Manager>`
- If subclasses have different comparison notions — guard with `if (getClass() != other.getClass()) throw new ClassCastException()`
- If a common algorithm exists — declare a single `final compareTo` in superclass

### Interface Properties

- Cannot instantiate an interface with `new` — but can declare interface variables
- `instanceof` works: `if (obj instanceof Comparable)`
- Interfaces can extend other interfaces (single or multiple)
- Constants in interfaces are `public static final` implicitly
- A class can implement multiple interfaces; `class Employee implements Cloneable, Comparable`
- Records and enums can implement interfaces but cannot extend classes
- Interfaces can be `sealed` — permitted subtypes must be declared in `permits` or same source file

### Interfaces vs. Abstract Classes

- A class can only extend one class — so abstract class as a generic contract limits flexibility
- Interfaces allow multiple implementation — `class Employee extends Person implements Comparable` is legal
- C++ supports multiple inheritance with all its complexity (virtual base classes, dominance rules) — Java uses interfaces as a cleaner alternative

### Static and Private Methods (Java 8+)

- Interfaces can have `static` methods — removes need for separate companion/utility classes
    - Eg - `Path.of(URI)` is a static method on the `Path` interface; no separate `Paths` class needed
- `private` methods allowed — can be static or instance; only usable as helpers within the interface itself

### Default Methods

- `default` keyword supplies a fallback implementation for an interface method
- Key use case: __interface evolution__ — adding a `default` method to an existing interface is both source-compatible and binary-compatible; adding a non-default method breaks both
- Default methods can call other methods in the interface
    - Eg - `default boolean isEmpty() { return size() == 0; }`

### Resolving Default Method Conflicts

- __Superclass wins__ — concrete method from a superclass always takes priority over a default method with the same signature from an interface
- __Interface clash__ — if two interfaces provide conflicting defaults, the class must override the method explicitly
    - Can delegate to a specific interface: `return Person.super.getName()`
- Cannot define a default method that redefines `Object` methods (`toString`, `equals`) — class wins rule means they can never override `Object`

### Callbacks with Interfaces

- __Callback pattern__ — specify action to occur when an event happens by passing an object implementing a functional interface
- `ActionListener` has single method `void actionPerformed(ActionEvent event)`
- `javax.swing.Timer(int interval, ActionListener listener)` — calls `actionPerformed` every `interval` ms

### `Comparator<T>`

- Use when sorting by a criterion other than the natural order, or when class isn't modifiable
- `int compare(T first, T second)` — called on the comparator object, not the elements
- Three rules:
    - Reflexivity — equal objects yield `0`
    - Antisymmetry — swapping args flips sign
    - Transitivity — `x < y` and `y < z` implies `x < z`
- Violation of rules may cause `Arrays.sort` (Timsort) to throw `"Comparison method violates its general contract!"`

### Object Cloning

- Default assignment copies the reference — both variables point to same object
- `clone()` creates a new object with same field values — defined as `protected` in `Object`
- __Shallow copy__ — copies field values; object references inside point to the same subobjects
- __Deep copy__ — also clones mutable subobjects (Eg - `cloned.hireDay = (Date) hireDay.clone()`)
- To enable cloning:
    - Implement `Cloneable` (a __tagging/marker interface__ — no methods, just enables `instanceof` check and lifts `Object.clone`'s paranoia)
    - Override `clone()` as `public`, return correct type (covariant return), call `super.clone()`
- If a class is `final`, catching `CloneNotSupportedException` internally is appropriate; otherwise propagate it to allow subclasses to opt out
- All array types have a `public clone()` method

> [!NOTE]
> `Cloneable` does not specify the `clone` method — that comes from `Object`. The interface is purely a tag to signal that the designer understands cloning semantics.

> [!TIP]
> Less than 5% of standard library classes implement `clone`. Prefer alternative copy mechanisms where possible.

---

## Lambda Expressions

### Why Lambdas

- Block of code passed to be executed later — previously required creating a class implementing a functional interface
- Lambdas allow passing behaviour directly, concisely, without boilerplate class definitions

### Syntax

```java
(String first, String second) -> first.length() - second.length()
```

- Multi-statement body uses `{}` with explicit `return`
- No parameters: `() -> { ... }`
- Inferred parameter types: `(first, second) -> first.length() - second.length()`
- Single inferred parameter: `event -> IO.println(event)`
- Unused parameter: `_ -> IO.println("action occurred")`
- Multiple unused: `(_, _) -> 0`
- `var` for annotated inferred types: `(@NonNull var first, @NonNull var second) -> ...`
- Return type always inferred from context — never declared
- Must return consistently across all branches — `(int x) -> { if (x >= 0) return 1; }` is illegal

### Functional Interfaces

- __Functional interface__ — interface with exactly one abstract method
- A lambda expression can be used wherever a functional interface instance is expected
- Common functional interfaces from `java.util.function`:

| Interface | Parameters | Return | Abstract Method |
|---|---|---|---|
| `Runnable` | none | `void` | `run` |
| `Supplier<T>` | none | `T` | `get` |
| `Consumer<T>` | `T` | `void` | `accept` |
| `BiConsumer<T,U>` | `T, U` | `void` | `accept` |
| `Function<T,R>` | `T` | `R` | `apply` |
| `BiFunction<T,U,R>` | `T, U` | `R` | `apply` |
| `UnaryOperator<T>` | `T` | `T` | `apply` |
| `BinaryOperator<T>` | `T, T` | `T` | `apply` |
| `Predicate<T>` | `T` | `boolean` | `test` |
| `BiPredicate<T,U>` | `T, U` | `boolean` | `test` |

- Primitive specialisations avoid boxing: `IntConsumer`, `IntUnaryOperator`, `ToIntFunction<T>`, `IntFunction<T>`, etc.
- `@FunctionalInterface` annotation — causes compiler error if >1 abstract method added; documents intent in javadoc

### Method References

- Shorthand when lambda body is a single method call
- Three variants:

| Syntax | Equivalent Lambda | Notes |
|---|---|---|
| `Class::staticMethod` | `x -> Class.staticMethod(x)` | Lambda param passed to static method |
| `Class::instanceMethod` | `x -> x.instanceMethod()` | Lambda param becomes implicit receiver |
| `object::instanceMethod` | `x -> object.instanceMethod(x)` | Lambda param passed as explicit arg |

- Compiler resolves overloads from context (target functional interface)
- `separator::equals` — throws `NullPointerException` immediately if `separator` is null; equivalent lambda defers the NPE to invocation
- `this::method` and `super::method` are valid

### Constructor References

- `ClassName::new` — resolves to whichever constructor matches the context
- `int[]::new` equivalent to `n -> new int[n]`
- Solves generic array creation problem: `stream.toArray(Person[]::new)` yields `Person[]` not `Object[]`

> [!NOTE]
> `Arrays.setAll(dates, Date::new)` calls `new Date(i)` (the milliseconds constructor), not `new Date()` — the `IntFunction` context determines which constructor is invoked.

### Variable Scope and Closures

- Lambda can capture variables from the enclosing scope
- __Captured variable must be effectively final__ — never reassigned after initialisation
- Mutating a captured variable is illegal; referring to a variable mutated outside is also illegal
- __Closure__ — a block of code plus the values of its free variables; Java lambdas are closures
- Lambda body has same scope as a nested block — cannot shadow local variables from the enclosing method
- Lambda has no `this` — `this` inside a lambda refers to the enclosing method's `this`
- Fields of the enclosing class are accessible (via implicit `this`) — no effectively-final restriction on fields

### Processing Lambda Expressions

- Design API methods to accept functional interfaces; call the abstract method inside
    - Eg - `public static void repeat(int n, Runnable action) { for (int i = 0; i < n; i++) action.run(); }`
    - Eg - `public static void repeat(int n, IntConsumer action) { for (int i = 0; i < n; i++) action.accept(i); }`
- Use primitive functional interfaces (`IntConsumer` over `Consumer<Integer>`) to avoid boxing

### Creating Comparators

- `Comparator.comparing(keyExtractor)` — compares by extracted key's natural order
- `Comparator.comparing(keyExtractor, keyComparator)` — custom comparator for the key
- `.thenComparing(keyExtractor)` — chained tie-breaker
- Primitive-specific: `Comparator.comparingInt`, `comparingLong`, `comparingDouble` — avoids boxing
- `Comparator.nullsFirst(comparator)` / `nullsLast(comparator)` — wraps a comparator to handle nulls
- `Comparator.naturalOrder()` — comparator for `Comparable` types
- `Comparator.reverseOrder()` / `comparator.reversed()` — reverse ordering

---

## Inner Classes

### Inner Class Access to Outer State

- __Inner class__ — class defined inside another class
- Inner class methods can access both their own fields and the private fields of the outer class
- Each inner class object holds an implicit reference to the outer class object that created it (`OuterClass.this`)
- Compiler adds an outer reference parameter to every inner class constructor automatically
- Inner class can be declared `private` — only outer class methods can then construct it
- Since Java 18: the `this$0` outer reference field is only generated when actually accessed

### Special Syntax

- Explicit outer reference: `OuterClass.this.field`
- Construct inner object with explicit outer: `outerObject.new InnerClass(args)`
- Refer to inner class from outside: `OuterClass.InnerClass`
- Since Java 16: inner classes can have `static` members

### Inner Class Translation

- Compiler translates inner class to `OuterClass$InnerClass.class`
- Synthesises `this$0` field for outer reference; adds outer class parameter to constructor
- Inner class has genuine access to outer private fields at the JVM level (via nest-mate access, Java 11+; earlier: compiler-generated bridge methods)

### Local Inner Classes

- Defined inside a method — no access modifier; scope limited to the enclosing block
- Completely hidden from outside, including other methods of the outer class
- Can access effectively-final local variables of the enclosing method
    - Compiler copies captured variables into synthesised fields (Eg - `val$beep`) in the local class

### Anonymous Inner Classes

```java
new SuperTypeOrInterface(args) {
    // methods and fields
}
```

- No name, no constructor — construction args go to superclass constructor
- Interface anonymous classes must use `()` with no args
- Can use object initialisation block `{ }` instead of constructor
- `var` with anonymous class — variable gets the nondenotable anonymous type, enabling access to added members
- Double-brace initialisation (`{{ add("x"); }}`) — outer braces create anonymous subclass, inner are init block; rarely useful
- Prefer lambdas over anonymous classes for functional interfaces
- `new Object(){}.getClass().getEnclosingClass()` — idiom to get current class in a static method

### Static Nested Classes

- __Static class__ — nested class declared `static`; no implicit reference to outer class object
- Use when the nested class does not need access to outer instance fields
- Required when nested class is instantiated inside a static method
- Classes declared inside interfaces are implicitly static
- Nested __records__ are automatically static
- Nested __enumerations__ are automatically static
- Nested __interfaces__ are implicitly static

> [!NOTE]
> Prior to Java 16, static classes could not be declared inside inner classes. This restriction is lifted.

---

## Service Loaders

- __Service architecture__ — define an interface; let providers supply implementations independently
- `ServiceLoader<S>.load(S.class)` — initialise once per program
- Provider classes must have a no-argument constructor
- Register implementations in `META-INF/services/<fully.qualified.InterfaceName>` — one class name per line (UTF-8)
- Iterate via enhanced for loop or `stream()` to pick an implementation
- `stream()` yields `ServiceLoader.Provider<S>` — call `type()` to inspect without instantiating, `get()` to instantiate
- `findFirst()` returns `Optional<S>` of the first available provider

---

## Proxies

### When to Use

- Dynamically create a class at runtime that implements one or more interfaces unknown at compile time
- Useful for: method call tracing, routing to remote servers, UI event association

### Creating a Proxy

```java
Object proxy = Proxy.newProxyInstance(
    ClassLoader.getSystemClassLoader(),
    new Class[]{ SomeInterface.class },
    handler  // InvocationHandler
);
```

- `InvocationHandler.invoke(Object proxy, Method method, Object[] args)` — called for every method invocation on the proxy
- Call `method.invoke(target, args)` inside handler to delegate to the real object

### Proxy Properties

- All proxy classes extend `Proxy`
- Only one instance field: the invocation handler (defined in `Proxy` superclass)
- `toString`, `equals`, `hashCode` from `Object` are proxied — other `Object` methods are not
- One proxy class per class loader + ordered set of interfaces
- Proxy classes are always `public` and `final`
- `Proxy.isProxyClass(cls)` — tests if a class is a proxy class
- Calling a `default` interface method on a proxy triggers the handler — use `InvocationHandler.invokeDefault(proxy, method, args)` to actually execute it