# Annotations

## What Annotations Are

- __Annotation__ — a tag inserted into source code for processing tools to consume
- Do not change compilation output — same bytecode with or without them
- Tools operate at three levels: source, class file (bytecode), or runtime (reflection)
- Eg - JUnit uses `@Test` to mark test methods; Jakarta Persistence uses annotations to map classes to DB tables

---

## Using Annotations

### Elements

- Annotations can have key/value pairs called __elements__: `@RepeatedTest(value=10, failureThreshold=3)`
- Element value types allowed:
    - Primitive type
    - `String`
    - `Class` object
    - Enum instance
    - Another annotation
    - Array of any of the above (not array of arrays)
- All element values must be compile-time constants
- Element values can never be `null`
- Single-element array shorthand: `@BugReport(reportedBy="Harry")` instead of `reportedBy={"Harry"}`
- Elements can have default values — omitting the element uses the default
- If the only element is named `value`, it can be omitted: `@RepeatedTest(10)` = `@RepeatedTest(value=10)`

### Multiple and Repeated Annotations

- Multiple different annotations can be applied to the same item
- If an annotation is declared `@Repeatable`, the same annotation can be applied multiple times:

```java
@Tag("localized")
@Tag("showstopper")
void testHello() { ... }
```

### Declaration Annotations — Where They Can Appear

- Classes/interfaces/enums/modules — before the keyword and modifiers
- Methods, constructors, fields (including enum constants and record components)
- Local variables (in `for`, `try`-with-resources)
- Parameter variables and `catch` parameters
- Type parameters
- Packages — in `package-info.java`, before the package statement

> [!NOTE]
> Annotations on local variables and packages are discarded at compile time — only processable at source level.

### Type Use Annotations

- Annotations on types, not just declarations:
    - Generic type arguments: `List<@NonNull String>`
    - Array brackets: `String[] @NonNull []` — `words[i]` not null; `String @NonNull [][]` — `words` not null
    - Superclass/interfaces: `class Warning extends @Localized Message`
    - Constructor invocations: `new @Localized String(...)`
    - Nested types: `Map.@Localized Entry`
    - Casts and `instanceof`: `(@Localized String) text`, `text instanceof @Localized String`
    - Exception specs: `throws @Localized IOException`
    - Wildcard bounds: `List<@Localized ? extends Message>`
    - Method/constructor references: `@Localized Message::getText`
- Cannot annotate: class literals (`@NonNull String.class`), imports
- If an annotation is valid for both variable and type use, both are annotated when used in a declaration

### Receiver Parameters

- `this` is not declared, so cannot be annotated directly
- Rarely used syntax to annotate the implicit `this`:

```java
public boolean equals(@NonNull Point this, @Nullable Object other) { ... }
```

- Only for methods, not constructors
- For inner class constructors, the enclosing class reference can be made explicit:

```java
public Iterator(@NonNull Sequence Sequence.this) { ... }
```

---

## Defining Annotations

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RepeatedTest {
    int failureThreshold() default Integer.MAX_VALUE;
}
```

- `@interface` creates a real Java interface — tools receive proxy objects implementing it
- Methods of annotation interface = elements; no parameters, no `throws`, not generic
- `@Target` and `@Retention` are __meta-annotations__

### `@Target` Values (`ElementType`)

| Value | Applies To |
|---|---|
| `ANNOTATION_TYPE` | Annotation type declarations |
| `MODULE` | Modules |
| `PACKAGE` | Packages |
| `TYPE` | Classes, interfaces (including annotation types), enums |
| `METHOD` | Methods |
| `CONSTRUCTOR` | Constructors |
| `FIELD` | Fields, enum constants |
| `PARAMETER` | Method/constructor parameters |
| `RECORD_COMPONENT` | Record components |
| `LOCAL_VARIABLE` | Local variables |
| `TYPE_PARAMETER` | Type parameters |
| `TYPE_USE` | Uses of a type |

- No `@Target` = can be used on any declaration (but not type parameters or type uses)
- Record component annotations propagate to generated fields/methods if their `@Target` permits it

### `@Retention` Values

- `RetentionPolicy.SOURCE` — available to source processors only; not in class files
- `RetentionPolicy.CLASS` — included in class files; JVM does not load them (default)
- `RetentionPolicy.RUNTIME` — available at runtime via reflection API

### Default Values

```java
public @interface BugReport {
    String[] reportedBy() default {};
    Reference ref() default @Reference(id=0);
}
```

- Defaults are not stored with annotations — computed dynamically
- Changing a default and recompiling takes effect in all annotated elements, even previously compiled ones

### Notes on Annotation Interfaces

- All annotation interfaces implicitly extend `java.lang.annotation.Annotation`
- Cannot extend annotation interfaces manually
- Never provide implementing classes — JVM and tools generate proxies
- `annotationType()` returns the annotation interface `Class`, unlike `getClass()` which returns the proxy class

---

## Annotations in the Java API

### Compilation Annotations

- `@Deprecated` — marks items as no longer encouraged; compiler warns on use; persists until runtime
- `@Override` — causes compile error if annotated method does not actually override a superclass method
- `@Serial` — checks that serialization methods have correct signatures
- `@SuppressWarnings("unchecked")` — suppresses specific compiler warnings
- `@SafeVarargs` — asserts varargs parameter is not corrupted; only on `static`, `final`, or `private` methods/constructors
- `@FunctionalInterface` — causes compiler error if interface has more than one abstract method
- `@Generated` — marks code generated by a tool; includes generator ID, optional ISO 8601 date and comment

### Meta-Annotations

- `@Target` — specifies where annotation can appear (see above)
- `@Retention` — specifies where annotation can be accessed (see above)
- `@Documented` — instructs JavaDoc to include annotation in docs (like `private`, `static`)
- `@Inherited` — applies to class annotations; subclasses automatically inherit the annotation
- `@Repeatable(ContainerAnnotation.class)` — allows annotation to be applied multiple times; requires a container annotation holding an array:

```java
@Repeatable(Tags.class)
@interface Tag { String value(); }

@interface Tags { Tag[] value(); }
```

---

## Processing Annotations at Runtime

- Use `RetentionPolicy.RUNTIME` so annotations are accessible via reflection
- Reflection classes implementing `AnnotatedElement`: `Class`, `Field`, `Parameter`, `Method`, `Constructor`, `Package`

### Key `AnnotatedElement` Methods

- `getAnnotation(Class<T>)` — returns annotation or `null`; returns `null` for repeated annotations (wrapped in container)
- `getAnnotationsByType(Class<T>)` — returns array, handles repeatable annotations transparently (preferred)
- `getDeclaredAnnotation(Class<T>)` / `getDeclaredAnnotationsByType(Class<T>)` — excludes inherited annotations
- `getAnnotations()` — all annotations including inherited, repeated ones wrapped in containers
- `getDeclaredAnnotations()` — all annotations excluding inherited
- `isAnnotationPresent(Class<? extends Annotation>)` — boolean check

```java
Class<?> cl = obj.getClass();
ToString ts = cl.getAnnotation(ToString.class);
if (ts != null && ts.includeName()) { ... }
```

> [!NOTE]
> For repeatable annotations, always use `getAnnotationsByType` — `getAnnotation` returns `null` if the annotation was repeated (because they are wrapped in a container).

---

## Source-Level Annotation Processing

### Annotation Processors

- Invoked during compilation: `javac -processor ProcessorClassName1,... sourceFiles`
- Processors can only generate new source files — cannot modify existing ones
- Processing is iterative: if new source files are produced, another round runs; continues until no new files are generated
- Implement `Processor` interface, typically by extending `AbstractProcessor`:

```java
@SupportedAnnotationTypes("com.example.MyAnnotation")
@SupportedSourceVersion(SourceVersion.RELEASE_21)
public class MyProcessor extends AbstractProcessor {
    public boolean process(Set<? extends TypeElement> annotations,
            RoundEnvironment currentRound) { ... }
}
```

- `@SupportedAnnotationTypes` accepts specific names, wildcards (`"com.myco.*"`), or `"*"` (all)
- `process` is called once per round; when no new files are generated, called with empty annotations set — return immediately to avoid duplicating output
- Flag `-XprintRounds` shows all processing rounds

### Language Model API

- Analyse source-level annotations using the language model API (not reflection — reflection uses JVM representation)
- Key types: `TypeElement` (class/interface), `VariableElement` (field/param), `ExecutableElement` (method/constructor)
- `RoundEnvironment.getElementsAnnotatedWith(Class<? extends Annotation>)` — get all elements with a given annotation
- `RoundEnvironment.getElementsAnnotatedWithAny(Set<...>)` — for repeated annotations
- `AnnotatedConstruct.getAnnotation(Class<A>)` / `getAnnotationsByType(Class<A>)` — source-level equivalents
- `TypeElement.getEnclosedElements()` — fields and methods of a class
- `Element.getSimpleName()` / `TypeElement.getQualifiedName()` — returns `Name`, call `toString()`

### Generating Source Files

```java
JavaFileObject sourceFile = processingEnv.getFiler().createSourceFile("pkg.ClassName");
try (var out = new PrintWriter(sourceFile.openWriter())) {
    out.println("package pkg;");
    out.println("public class ClassName { ... }");
}
```

- Generated files can be Java source, XML, properties, scripts, HTML — anything
- Annotation processors cannot add methods to existing classes — that would require modifying source files
    - Project Lombok works around this by modifying internal compiler data structures (non-standard)

---

## Bytecode Engineering

### Modifying Class Files

- Class files retain annotation information (with `RetentionPolicy.CLASS` or `RUNTIME`)
- JDK class file API (since Java 24) — `java.lang.classfile` package
- Key types: `ClassFile`, `ClassModel`, `MethodModel`, `CodeModel`, `MethodTransform`, `ClassTransform`
- General pattern:

```java
ClassFile cf = ClassFile.of();
ClassModel classModel = cf.parse(classFileBytes);
ClassTransform transform = ClassTransform.transformingMethods(filter, methodTransform);
byte[] newBytes = cf.transformClass(classModel, transform);
```

- Inspect annotations on methods: `mm.findAttribute(Attributes.runtimeVisibleAnnotations())`
- Insert bytecode instructions at start of method body using `CodeBuilder` — prepend before iterating existing `CodeModel` elements

### Modifying Bytecodes at Load Time (Java Agents)

- Avoids adding a separate build step — modification happens at class load time
- Uses the instrumentation API (`java.lang.instrument`)
- Steps:
    - Implement `premain(String arg, Instrumentation instr)` — called before `main`
    - Create manifest with `Premain-Class: fully.qualified.AgentClass`
    - Package into a JAR: `jar cvfm Agent.jar Agent.mf Agent*.class`
    - Run with: `java -javaagent:Agent.jar=agentArg MainClass`
- Register a `ClassFileTransformer` via `instr.addTransformer(...)`:
    - `transform(...)` receives class bytes; return `null` to leave unchanged, or return modified bytes
    - Modification is "just in time" — modified bytes loaded into JVM but not written to disk

```java
public static void premain(String arg, Instrumentation instr) {
    instr.addTransformer((loader, className, cl, pd, data) -> {
        if (!className.replace("/", ".").equals(arg)) return null;
        return modifyBytes(data);
    });
}
```

### Three Levels of Annotation Processing

| Level | When | Tools |
|---|---|---|
| Source | During compilation | `javac -processor`, `AbstractProcessor`, language model API |
| Bytecode | Post-compile or at load time | Class file API, Java agents, `Instrumentation` |
| Runtime | During execution | Reflection API, `AnnotatedElement` methods |