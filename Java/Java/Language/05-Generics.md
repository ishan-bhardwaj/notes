# Generic Programming

## Why Generics

- Pre-generics: collections stored `Object` references — required casts on retrieval, no type safety at compile time
- Generics add __type parameters__ — compiler knows element types, inserts casts automatically, catches type errors at compile time not runtime
- Diamond syntax: `ArrayList<String> files = new ArrayList<>()` — type inferred from variable declaration

---

## Defining Generic Classes and Methods

### Generic Classes

```java
public class Pair<T, U> {
    private T first;
    private U second;
    ...
}
```

- Type variable conventions: `E` (element), `K`/`V` (key/value), `T`/`U`/`S` (any type)
- Instantiate by substituting a concrete type: `Pair<String>` — acts as a factory for ordinary classes

### Generic Methods

```java
<T> T getMiddle(T... a) { return a[a.length / 2]; }
```

- Type variables go after modifiers, before return type
- Compiler infers type from arguments — explicit call: `this.<String>getMiddle(...)`
- Type inference can fail with mixed numeric types — remedy: make all arguments the same type
- Inferring large numbers of types is slow — help compiler with explicit type args for large `List.of(...)` calls

### Bounds for Type Variables

```java
<T extends Comparable> T min(T[] a)
<T extends Comparable & Serializable> T min(T[] a)
```

- `extends` is used for both class and interface bounds — chosen as approximation of "subtype"
- Multiple bounds separated by `&`; commas separate multiple type variables
- At most one class bound; if present, it must be first in the list

### Generic Exceptions

```java
interface Task<T extends Throwable> { void run() throws T; }
<T extends Throwable> void repeat(Task<T> t, int n) throws T { ... }
```

- Type variable allowed in `throws` clause
- Cannot use a type variable in a `catch` clause — compile error
- Cannot throw or catch objects of a generic class
- A generic class cannot extend `Throwable`

---

## Type Erasure

- JVM has no generic types — all type parameters are erased at compile time
- __Raw type__ — the erased form: `Pair<T>` becomes `Pair` with `Object` fields (or first bound if bounded)
- `Pair<String>` and `Pair<Employee>` both erase to raw `Pair`

### What Erasure Produces

- Unbounded `<T>` → replaced with `Object`
- Bounded `<T extends Comparable & Serializable>` → replaced with `Comparable` (first bound)
    - Switching bound order to `<T extends Serializable & Comparable>` changes raw type fields to `Serializable`; compiler inserts casts to `Comparable` where needed

### Compiler Insertions

- Casts inserted on field access and method return when raw type is `Object` but caller expects a specific type
    - Eg - `buddies.getFirst()` on `Pair<Employee>` → JVM calls raw `Pair.getFirst()` returning `Object`, compiler inserts cast to `Employee`

### Bridge Methods

- Required to preserve polymorphism when subclassing a generic type

```java
class DateInterval extends Pair<LocalDate> {
    public void setSecond(LocalDate second) { ... }
}
```

- After erasure, `Pair.setSecond(Object)` is inherited — different signature from `DateInterval.setSecond(LocalDate)`
- Compiler synthesises a bridge: `public void setSecond(Object second) { setSecond((LocalDate) second); }`
- Bridge methods also synthesised for covariant return types

### Summary of Generic Translation

- No generics in JVM — only ordinary classes and methods
- All type parameters replaced by bounds
- Bridge methods synthesised to preserve polymorphism
- Casts inserted as necessary to preserve type safety

### Legacy Code Interoperability

- `Pair<Employee>` is a subtype of raw `Pair` — allows passing to legacy APIs
- Passing a parameterised type to a raw-typed API parameter produces an __unchecked warning__
- Receiving a raw type and assigning to a parameterised variable also produces an unchecked warning
- Suppress with `@SuppressWarnings("unchecked")` on the variable or containing method

---

## Inheritance Rules for Generic Types

- `Pair<Manager>` is NOT a subtype of `Pair<Employee>` — no relationship regardless of how `S` and `T` are related
- This is required for type safety:
    - If it were allowed: `Pair<Employee> e = new Pair<Manager>(...)` then `e.setFirst(lowlyEmployee)` would corrupt the `Manager` pair
- Contrast with arrays: `Manager[]` IS a subtype of `Employee[]` but protected at runtime by `ArrayStoreException`
- `Pair<Employee>` IS a subtype of raw `Pair` — necessary for legacy interop
- `ArrayList<Manager>` IS a subtype of `List<Manager>` — generic classes can extend/implement other generic classes normally

---

## Wildcard Types

### Subtype Bound (`? extends T`)

```java
void printBuddies(Pair<? extends Employee> p)
```

- `Pair<Manager>` is a subtype of `Pair<? extends Employee>`
- __Can read__ — `getFirst()` returns `? extends Employee`, safe to assign to `Employee`
- __Cannot write__ — `setFirst(? extends Employee)` cannot be called; compiler cannot know the specific type

### Supertype Bound (`? super T`)

```java
void minmaxTitle(Executive[] a, Pair<? super Executive> result)
```

- Accepts `Pair<Executive>`, `Pair<Manager>`, `Pair<Employee>`, `Pair<Object>`
- __Can write__ — `setFirst(? super Executive)`: safe to pass an `Executive` (or subtype) since the actual type is a supertype of `Executive`
- __Cannot read usefully__ — `getFirst()` returns `? super Executive`; can only assign to `Object`
- Rule of thumb: __subtype bound = read (producer); supertype bound = write (consumer)__

### Use with `Comparable`

```java
<T extends Comparable<? super T>> Pair<T> minmax(T[] a)
```

- Needed because `LocalDate` implements `Comparable<ChronoLocalDate>`, not `Comparable<LocalDate>`
- `? super T` allows `compareTo` to accept a supertype of `T`

### Unbounded Wildcard (`?`)

```java
boolean hasNulls(Pair<?> p)
```

- `getFirst()` returns `?` — can only assign to `Object`
- `setFirst(?)` cannot be called at all (except with `null`)
- Use when actual type is irrelevant — simpler than making the method generic

### Wildcard Capture

- Cannot use `?` as a type variable directly — `? t = p.getFirst()` is illegal
- Solution: delegate to a generic helper method that captures the wildcard:

```java
void swap(Pair<?> p) { swapHelper(p); }
<T> void swapHelper(Pair<T> p) {
    T t = p.getFirst();
    p.setFirst(p.getSecond());
    p.setSecond(t);
}
```

- Capture is only legal when compiler can guarantee the wildcard represents a single definite type

---

## Restrictions and Limitations

### Cannot Use Primitive Types as Type Parameters

- No `Pair<double>` — only `Pair<Double>`; reason: type erasure leaves `Object` fields which cannot hold primitives

### Casts and `instanceof` with Generic Types

- `(Pair<String>) obj` — compiles with warning; only checks that `obj` is some `Pair` at runtime
- `obj instanceof Pair<String>` — compile error
- `obj instanceof Pair<T>` — compile error
- `obj instanceof T` — compile error
- Cast `(T) obj` — does nothing at runtime; compile warning only

### No Arrays of Parameterised Types

- `new Pair<String>[10]` — compile error
- Reason: array stores are type-checked at runtime using the erased type; `Pair<Employee>` stored in a `Pair<String>[]` would not be caught
- Can declare `Pair<String>[]` variable but cannot initialise with `new Pair<String>[n]`
- Cast `(Pair<String>[]) new Pair<?>[10]` — compiles but unsafe
- Preferred alternative: `ArrayList<Pair<String>>`

### Varargs with Generic Types

- Passing generic instances to varargs produces a warning (not error) at the call site
- Annotate the varargs method with `@SafeVarargs` to suppress warnings at both declaration and call sites
- `@SafeVarargs` only allowed on `static`, `final`, or `private` methods/constructors

### Generic Varargs Do Not Spread Primitive Arrays

- `addAll(strings, new String[]{"a", "b"})` — works; array is passed directly
- `addAll(numbers, new int[]{1, 2, 3})` — compile error; `int[]` cannot match `Integer`
- `List.of(new int[]{1,2,3})` — produces `List<int[]>` of one element, not `List<Integer>` of three

### Cannot Instantiate Type Variables

- `new T()` is illegal — erasure would make it `new Object()`
- Workaround 1 — supplier: `<T> Pair<T> makePair(Supplier<T> constr)` called as `makePair(String::new)`
- Workaround 2 — `Class<T>` parameter: `<T> Pair<T> makePair(Class<T> cl)` called as `makePair(String.class)`

### Cannot Construct Generic Arrays

- `new T[2]` — illegal; erases to `new Comparable[2]`
- Private field workaround — use `Object[]` internally, cast on get:

```java
private Object[] elements;
@SuppressWarnings("unchecked") public E get(int n) { return (E) elements[n]; }
```

- Return-value workaround — pass array constructor expression:

```java
<T extends Comparable> T[] minmax(IntFunction<T[]> constr, T... a) {
    T[] result = constr.apply(2);
    ...
}
// called as: minmax(String[]::new, "Tom", "Dick", "Harry")
```

- Reflection workaround: `Array.newInstance(a.getClass().getComponentType(), 2)`

### No Static Fields or Methods with Type Variables

- `static T field` and `static T method()` inside `Singleton<T>` — illegal
- After erasure only one class exists — one static field, regardless of type parameter

### Defeating Checked Exception Checking

- `@SuppressWarnings("unchecked") static <T extends Throwable> void throwAs(Throwable t) throws T { throw (T) t; }`
- The cast `(T) t` is erased — no runtime check occurs; exception stays checked but compiler is fooled
- Enables throwing checked exceptions from `Runnable.run()` or other methods that declare no checked exceptions

### Clashes after Erasure

- Adding `boolean equals(T value)` to `Pair<T>` — erases to `boolean equals(Object)` — clashes with `Object.equals`
- A class cannot implement two different parameterisations of the same interface:
    - `class Manager extends Employee implements Comparable<Manager>` — illegal if `Employee implements Comparable<Employee>`
    - Reason: two bridge methods `compareTo(Object)` would be generated with different casts — conflict

---

## Reflection and Generics

### `Class<T>`

- `String.class` is of type `Class<String>` — single instance
- Methods that benefit from the type parameter:
    - `T cast(Object obj)` — casts and returns, or throws `ClassCastException`
    - `T[] getEnumConstants()` — returns enum values or `null`
    - `Class<? super T> getSuperclass()`
    - `Constructor<T> getConstructor(Class<?>... parameterTypes)`

### `Type` Interface and Subtypes

Used to represent generic type declarations via reflection:

| Subtype | Represents |
|---|---|
| `Class` | Concrete types |
| `TypeVariable` | Type variables (`T extends Comparable<? super T>`) |
| `WildcardType` | Wildcards (`? super T`) |
| `ParameterizedType` | Generic class/interface types (`Comparable<? super T>`) |
| `GenericArrayType` | Generic arrays (`T[]`) |

- Access via: `Method.getGenericReturnType()`, `Method.getGenericParameterTypes()`, `Class.getTypeParameters()`, `Class.getGenericSuperclass()`, `Class.getGenericInterfaces()`
- Erased objects at runtime don't carry their type arguments — but field and method parameter types survive in class files

### Type Literals

```java
var type = new TypeLiteral<ArrayList<Integer>>(){};
```

- Anonymous subclass trick: `getClass().getGenericSuperclass()` on the anonymous subclass gives the parameterised type, including type arguments
- Used by injection frameworks (CDI, Guice) to control injection of generic types
- Enables mapping different parameterisations of the same raw type to different behaviours at runtime