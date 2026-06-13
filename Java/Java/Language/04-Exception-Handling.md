# Exception Handling

## Exception Hierarchy

- All exceptions descend from `Throwable`, which splits into two branches:
    - `Error` — internal JVM errors and resource exhaustion; do not throw or catch these
    - `Exception` — splits further into:
        - `RuntimeException` — programming errors; unchecked
        - Everything else — environmental failures (I/O, network); checked

### Checked vs. Unchecked

- __Checked exception__ — any exception not deriving from `Error` or `RuntimeException`; compiler enforces handling or declaration
- __Unchecked exception__ — derives from `Error` or `RuntimeException`; compiler does not enforce handling
- Rule: if it's a `RuntimeException`, it was your fault (bad cast, null deref, out-of-bounds)
- Common `RuntimeException` subclasses: `ClassCastException`, `ArrayIndexOutOfBoundsException`, `NullPointerException`
- Common checked exceptions: `IOException`, `FileNotFoundException`, `EOFException`

---

## Declaring Checked Exceptions

- Methods must advertise checked exceptions they can throw via `throws` in the method header
- Do NOT declare `Error` or `RuntimeException` subclasses in `throws` — either beyond control or your fault
- Multiple exceptions: `throws FileNotFoundException, EOFException`
- Subclass override rule: subclass method cannot declare more general checked exceptions than superclass; if superclass declares none, subclass must catch all checked exceptions internally

```java
public Image loadImage(String s) throws IOException { ... }
```

---

## Throwing Exceptions

```java
throw new EOFException("Content-length: " + len + ", Received: " + n);
```

- Find an appropriate existing class or create a custom one
- Once thrown, method exits immediately — no return value needed

### Custom Exception Classes

- Extend `Exception` or a subclass (Eg - `IOException`)
- Provide both a no-arg constructor and a `String message` constructor

```java
class FileFormatException extends IOException {
    public FileFormatException() {}
    public FileFormatException(String gripe) { super(gripe); }
}
```

---

## Catching Exceptions

### Basic `try/catch`

```java
try {
    // code
} catch (IOException e) {
    e.printStackTrace();
}
```

- If no exception: `catch` block skipped
- If wrong exception type: method exits immediately, caller must handle

### Multiple `catch` Clauses

- Order from most specific to most general — otherwise more specific clauses are unreachable
- Multi-catch (Java 7+): `catch (FileNotFoundException | UnknownHostException e)`
    - Variable `e` is implicitly `final` in multi-catch
    - Actual type of `e` is the least upper bound (Eg - `IOException`), not a union type
    - Multi-catch generates a single bytecode block — more efficient
- Sealed exception hierarchies are NOT handled specially by `catch` — exhaustiveness is not checked

### Rethrowing and Chaining

- Rethrow to change exception type — use the original as the cause:

```java
catch (SQLException original) {
    var e = new ServletException("database error", original);
    throw e;
}
```

- Retrieve cause: `caughtException.getCause()`
- Log and rethrow unchanged:

```java
catch (Exception e) {
    logger.log(level, message, e);
    throw e;
}
```

- Compiler tracks that `e` originates from the try block — `throws SQLException` is valid if only `SQLException` is thrown in the block
- Use `UncheckedIOException` to wrap `IOException` inside methods that cannot throw checked exceptions

### `finally` Clause

- Executes whether or not an exception was caught — used for resource cleanup
- Three execution paths:
    - No exception: `try` body → `finally` → code after `finally`
    - Caught exception: `try` (partial) → `catch` → `finally` → code after `finally` (if `catch` doesn't rethrow)
    - Uncaught exception: `try` (partial) → `finally` → exception propagates to caller
- `finally` without `catch` is valid — exception propagates after cleanup runs

> [!NOTE]
> Never put `return`, `throw`, `break`, or `continue` inside `finally` — a `return` in `finally` masks the try block's return value and swallows exceptions.

### `try`-with-Resources

- Resource must implement `AutoCloseable` (`void close() throws Exception`) or `Closeable` (subinterface, `throws IOException`)
- `close()` is called automatically on block exit, whether normal or exceptional

```java
try (var in = new Scanner(Path.of("in.txt"));
     var out = new PrintWriter("out.txt")) {
    // work
}
```

- If both `try` block and `close()` throw, original exception is rethrown; `close()` exceptions are added as __suppressed exceptions__ via `addSuppressed`
- Retrieve suppressed: `getSuppressed()` returns `Throwable[]`
- Effectively final variables can be used in the try header directly: `try (out) { ... }`
- Can have `catch` and `finally` clauses — execute after resource `close()`

> [!TIP]
> Prefer `try`-with-resources over manual `finally` for all resource cleanup.

### Stack Traces

- `t.printStackTrace(new PrintWriter(out))` — captures stack trace as string
- `StackWalker.getInstance().forEach(frame -> ...)` — lazy stream-based traversal
- `StackWalker.walk(stream -> ...)` — process as `Stream<StackWalker.StackFrame>`
- `Throwable.getStackTrace()` — returns `StackTraceElement[]`; less efficient (captures entire stack eagerly, no `Class` objects)

---

## Tips for Using Exceptions

- Do not use exceptions for flow control — tests are orders of magnitude faster (Eg - `isEmpty()` before `pop()`)
- Do not wrap every statement in its own `try` block — wrap the whole logical operation
- Use the hierarchy correctly:
    - Throw specific subclasses, not plain `Exception` or `RuntimeException`
    - Catch only what you expect — catching `Exception` or `Throwable` hides bugs including `OutOfMemoryError`
    - Respect checked vs. unchecked distinction — checked exceptions are for recoverable environmental failures, not logic errors
- Do not squelch exceptions with empty `catch` blocks
- Fail fast — throw at point of failure rather than returning dummy values that cause mysterious NPEs later
- Propagating is not a sign of shame — higher-level callers are often better equipped to handle errors
- Use `Objects.requireNonNull`, `Objects.checkIndex`, `checkFromToIndex`, `checkFromIndexSize` for parameter validation
- Do not show stack traces to end users — log them, display only a summary message

---

## Assertions

### Syntax

```java
assert condition;
assert condition : expression;  // expression becomes the error message
```

- Throws `AssertionError` if condition is `false`
- `AssertionError` does not store the expression value — intentional, to prevent recovery

### Enabling and Disabling

- Disabled by default — enable with `-ea` or `-enableassertions`:

```java
java -ea MyProgram
java -ea:MyClass -ea:com.mycompany.mypackage... MyProgram
java -ea:... -da:MyClass MyProgram   // enable all, disable specific
```

- `-esa` / `-enablesystemassertions` — enables for system classes (no class loader)
- Enabling/disabling is a class loader function — no recompilation needed
- Assertion condition must be side-effect-free — condition is not evaluated when assertions are disabled

```java
// BAD — file.delete() not called when assertions off
assert file.delete();

// GOOD
boolean deleted = file.delete();
assert deleted;
```

### When to Use Assertions

- Assertions are for fatal, unrecoverable internal errors caught during development/testing only
- Use for:
    - __Preconditions__ — when contract explicitly states requirement (Eg - `assert a != null` if docs say "must not be null")
    - __Documenting assumptions__ — replace `// (i % 3 == 2)` comments with `assert i % 3 == 2`
- Do NOT use for:
    - Checking publicly documented parameter contracts — throw `IllegalArgumentException` or `ArrayIndexOutOfBoundsException` instead
    - Communicating recoverable errors to callers
    - User-facing error messages

### Three Error-Handling Mechanisms Compared

| Mechanism | When to Use |
|---|---|
| Exception | Recoverable errors, documented contracts, caller communication |
| Assertion | Internal invariants, preconditions in private code, development/testing |
| Logging | Diagnostic information across the entire program lifecycle |

---

## Logging

### Frontend vs. Backend

- __Frontend__ — API used by programmers to emit log messages (Eg - platform logging API `System.Logger`)
- __Backend__ — handles filtering, formatting, destination (Eg - `java.util.logging`, Log4j, Logback)
- __Façade__ — decouples frontend from backend (Eg - SLF4J, JEP 264 platform logging API)

### Platform Logging API (`System.Logger`)

```java
System.Logger logger = System.getLogger("com.mycompany.myapp");
logger.log(System.Logger.Level.INFO, "Opening file " + filename);
```

- Levels (decreasing severity): `ERROR`, `WARNING`, `INFO`, `DEBUG`, `TRACE`
- Deferred message computation: `logger.log(INFO, () -> "Opening file " + filename)`
- Logging an exception: `logger.log(WARNING, "Cannot open file", ex)`
- Formatted message (MessageFormat-style, not printf): `logger.log(WARNING, "Cannot open file {0}", filename)`
    - Placeholders: `{0}`, `{1}`, ...
    - Escape braces with single quotes: `'{'`
    - Literal single quote: `''`
- With resource bundle: `logger.log(WARNING, bundle, "file.bad", filename)`

### `java.util.logging` Backend Configuration

- Default config: `conf/logging.properties` in JDK
- Override: `java -Djava.util.logging.config.file=logging.properties MainClass`
- Cannot use `-D` to set logging properties — log manager initialises before `main`

```properties
handlers=java.util.logging.ConsoleHandler,java.util.logging.FileHandler
com.mycompany.myapp.level=WARNING
java.util.logging.ConsoleHandler.level=FINE
```

- Level name mapping between APIs:

| Platform Logging | `java.util.logging` |
|---|---|
| `ERROR` | `SEVERE` |
| `WARNING` | `WARNING` |
| `INFO` | `INFO` |
| `DEBUG` | `FINE` |
| `TRACE` | `FINER` |

- Logger hierarchy mirrors package hierarchy — disabling a parent disables all children
- `ConsoleHandler` default level is `INFO` — must set `ConsoleHandler.level=FINE` to see `DEBUG`/`FINE`

### Log Handlers

- `ConsoleHandler` — writes to `System.err`
- `FileHandler` — writes to file (XML by default)
    - Pattern variables: `%h` (home dir), `%t` (temp dir), `%u` (unique number), `%g` (rotation generation), `%%` (literal `%`)
    - Supports rotation: configure `count` and `limit` (bytes)
    - Recommended pattern: `%h/myapp%u.log`
- `SocketHandler` — sends to a host/port

Key `FileHandler` config properties:

| Property | Description | Default |
|---|---|---|
| `level` | Handler level | `ALL` |
| `append` | Append to existing file | `false` |
| `limit` | Max bytes before rotation | `0` (no limit) |
| `pattern` | File name pattern | `%h/java%u.log` |
| `count` | Files in rotation sequence | `1` |
| `formatter` | Log record formatter | `XMLFormatter` |

### Filters and Formatters

- Custom filter: implement `Filter` — `boolean isLoggable(LogRecord record)`
- Custom formatter: extend `Formatter` — override `String format(LogRecord record)`
    - Call `formatMessage(record)` to localise and format the message part
    - Override `getHead(Handler h)` and `getTail(Handler h)` for wrapping formats (Eg - XML)
- Register via config: `java.util.logging.ConsoleHandler.filter=com.mycompany.myapp.MyFilter`

### Logging Recipe

```java
// Declare logger as static field
private static final System.Logger logger
    = System.getLogger("com.mycompany.myprog");

// Replace console prints with logger calls
logger.log(System.Logger.Level.TRACE, "File open dialog canceled");

// Log unexpected exceptions
catch (SomeException e) {
    logger.log(System.Logger.Level.WARNING, "context description", e);
}
```

---

## Debugging Tips

- Print/log any variable: `logger.log(DEBUG, "x=" + x)` or `logger.log(DEBUG, "this=" + this)`
- Add a `void main()` to each class with demo/test code — graduate to JUnit for test suites
- __Logging proxy__ — anonymous subclass intercepting method calls:

```java
var generator = new Random() {
    public double nextDouble() {
        double result = super.nextDouble();
        logger.log(DEBUG, "nextDouble: " + result);
        return result;
    }
};
```

- Catch-log-rethrow pattern:

```java
catch (Throwable t) {
    t.printStackTrace();
    throw t;
}
```

- Print stack trace anywhere without an exception: `Thread.dumpStack()`
- Capture stack trace to string: `new Throwable().printStackTrace(new PrintWriter(out))`
- Redirect `System.err` to file: `java MyProgram 2> errors.txt`
- Capture both stdout and stderr: `java MyProgram 1> errors.txt 2>&1`
- Set default uncaught exception handler:

```java
Thread.setDefaultUncaughtExceptionHandler((t, e) ->
    logger.log(TRACE, "Uncaught exception in " + t, e));
```

- Watch class loading: `java -verbose MyProgram`
- Compiler warnings: `javac -Xlint:all,-fallthrough,-serial sourceFiles`; list all: `javac --help-lint`
- JVM monitoring tools:
    - `jconsole` — GUI for memory, threads, class loading, performance statistics
    - `jcmd` — command-line; list processes, VM flags, system properties, thread dumps, flight recording
        - `jcmd pidOrMainClass Thread.print` — thread dump
        - `jcmd pidOrMainClass VM.flags` — VM flags
        - `jcmd pidOrMainClass JFR.start filename=filename` — start flight recorder
    - VisualVM / Java Mission Control — collect and view flight recording data