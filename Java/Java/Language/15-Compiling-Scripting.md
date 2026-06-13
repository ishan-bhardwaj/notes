# Compiling and Scripting

## The Compiler API

### Invoking the Compiler

```java
JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
int result = compiler.run(null, outStream, errStream,
    "-sourcepath", "src", "MyProg.java");
// result == 0 means success
```

- `run` inherited from `javax.tools.Tool`
- First argument is an input stream — always `null` for the compiler (takes no console input)
- Remaining arguments are the same as `javac` command-line arguments
- Pass `null` for `outStream`/`errStream` to use `System.out`/`System.err`

### Launching a Compilation Task

```java
StandardJavaFileManager stdFileManager =
    compiler.getStandardFileManager(null, null, null);

Iterable<JavaFileObject> sources =
    stdFileManager.getJavaFileObjectsFromStrings(List.of("File1.java", "File2.java"));

Iterable<String> options = List.of("-d", "bin");

JavaCompiler.CompilationTask task = compiler.getTask(
    null,                   // error writer; null → System.err
    fileManager,            // null → standard file manager
    diagnosticListener,     // null → System.err
    options,                // null if no options
    null,                   // classes for annotation processing; null if none
    sources);               // source files

Boolean result = task.call();  // synchronous
// or submit to ExecutorService for async
```

- `CompilationTask` extends `Callable<Boolean>`
- `getTask` returns the task without starting compilation
- For annotation processing: call `task.processors(annotationProcessors)` before `call()`

### Capturing Diagnostics

```java
var collector = new DiagnosticCollector<JavaFileObject>();
compiler.getTask(null, fileManager, collector, null, null, sources).call();

for (Diagnostic<? extends JavaFileObject> d : collector.getDiagnostics()) {
    System.out.println(d.getKind() + ": " + d.getMessage(null));
    System.out.println(d.getSource() + " line " + d.getLineNumber());
}
```

- `Diagnostic.Kind`: `ERROR`, `WARNING`, `MANDATORY_WARNING`, `NOTE`, `OTHER`
- Also install a `DiagnosticListener` on `getStandardFileManager(listener, null, null)` to trap missing-file errors

### Reading Source Files from Memory

```java
public class StringSource extends SimpleJavaFileObject {
    private String code;

    StringSource(String name, String code) {
        super(URI.create("string:///" + name.replace(".", "/") + ".java"), Kind.SOURCE);
        this.code = code;
    }

    public CharSequence getCharContent(boolean ignoreEncodingErrors) {
        return code;
    }
}
```

- Pass a `List<StringSource>` as the sources argument to `getTask`
- Useful for compiling dynamically generated code without writing to disk

### Writing Bytecodes to Memory

```java
public class ByteArrayClass extends SimpleJavaFileObject {
    private ByteArrayOutputStream out;

    ByteArrayClass(String name) {
        super(URI.create("bytes:///" + name.replace(".", "/") + ".class"), Kind.CLASS);
    }

    public byte[] getCode() { return out.toByteArray(); }

    public OutputStream openOutputStream() {
        out = new ByteArrayOutputStream();
        return out;
    }
}
```

Intercept file manager output via `ForwardingJavaFileManager`:

```java
var classes = new ArrayList<ByteArrayClass>();
JavaFileManager fileManager = new ForwardingJavaFileManager<>(stdFileManager) {
    public JavaFileObject getJavaFileForOutput(Location location,
            String className, Kind kind, FileObject sibling) throws IOException {
        if (kind == Kind.CLASS) {
            var outfile = new ByteArrayClass(className);
            classes.add(outfile);
            return outfile;
        }
        return super.getJavaFileForOutput(location, className, kind, sibling);
    }
};
```

Load compiled classes with a custom `ClassLoader`:

```java
public class ByteArrayClassLoader extends ClassLoader {
    private Iterable<ByteArrayClass> classes;

    public ByteArrayClassLoader(Iterable<ByteArrayClass> classes) {
        this.classes = classes;
    }

    public Class<?> findClass(String name) throws ClassNotFoundException {
        for (ByteArrayClass cl : classes) {
            if (cl.getName().equals("/" + name.replace(".", "/") + ".class"))
                return defineClass(name, cl.getCode(), 0, cl.getCode().length);
        }
        throw new ClassNotFoundException(name);
    }
}

Class<?> cl = Class.forName(className, true, new ByteArrayClassLoader(classes));
```

---

## Scripting for the Java Platform

### Getting a Scripting Engine

```java
var manager = new ScriptEngineManager();

// List available engines
for (ScriptEngineFactory factory : manager.getEngineFactories())
    System.out.println(factory.getEngineName());

// Get engine by name, extension, or MIME type
ScriptEngine engine = manager.getEngineByName("rhino");
ScriptEngine engine = manager.getEngineByExtension("js");
ScriptEngine engine = manager.getEngineByMimeType("application/javascript");
```

- Engines discovered at JVM startup from classpath JARs
- Nashorn (Oracle's JS engine) removed in Java 15 — use Rhino or Nashorn standalone instead
- Include engine JAR on classpath at runtime

### Script Evaluation and Bindings

```java
Object result = engine.eval(scriptString);
Object result = engine.eval(reader);           // from file
```

- Variables, functions, and classes defined in one `eval` call persist for subsequent calls on the same engine

Bindings:

```java
// Engine scope
engine.put("k", 1728);
Object result = engine.eval("k + 1");          // 1729

engine.eval("n = 1728");
Object n = engine.get("n");                    // retrieve from engine scope

// Global scope (visible to all engines)
manager.put("key", value);

// Scoped bindings (do not persist after the call)
Bindings scope = engine.createBindings();
scope.put("d", LocalDate.now());
engine.eval(scriptString, scope);
```

Thread safety — check before sharing engines across threads:

```java
Object param = factory.getParameter("THREADING");
// null → not thread-safe
// "MULTITHREADED" → safe; effects visible across threads
// "THREAD-ISOLATED" → separate bindings per thread
// "STATELESS" → no variable bindings altered
```

### Redirecting Input and Output

```java
var writer = new StringWriter();
engine.getContext().setWriter(new PrintWriter(writer, true));
// JavaScript print/println now goes to writer
```

- `setReader` / `setWriter` / `setErrorWriter` on the `ScriptContext`
- Only affects the scripting engine's I/O — direct Java `System.out` calls are unaffected
- Rhino has no standard input concept — `setReader` has no effect

### Calling Scripting Functions and Methods

Engines that support this implement `Invocable`:

```java
// Define and call a top-level function
engine.eval("function greet(how, whom) { return how + ', ' + whom + '!' }");
Object result = ((Invocable) engine).invokeFunction("greet", "Hello", "World");

// Call a method on a scripting object
engine.eval("function Greeter(how) { this.how = how }\n" +
            "Greeter.prototype.welcome = function(whom) { return this.how + ', ' + whom + '!' }");
Object yo = engine.eval("new Greeter('Yo')");
Object result = ((Invocable) engine).invokeMethod(yo, "welcome", "World");
```

Implement a Java interface via scripting:

```java
public interface Greeter { String welcome(String whom); }

// Global function implementing the interface
engine.eval("function welcome(whom) { return 'Hello, ' + whom + '!' }");
Greeter g = ((Invocable) engine).getInterface(Greeter.class);
result = g.welcome("World");

// Object method implementing the interface
Greeter g = ((Invocable) engine).getInterface(yo, Greeter.class);
result = g.welcome("World");
```

- Avoids dealing with scripting language syntax from Java call sites
- `getMethodCallSyntax` on `ScriptEngineFactory` provides a language-independent alternative when `Invocable` is unavailable — all parameters must be bound to names

### Compiling a Script

Engines that support compilation implement `Compilable`:

```java
if (engine instanceof Compilable compilableEngine) {
    CompiledScript script = compilableEngine.compile(Files.newBufferedReader(path));
    script.eval();
    script.eval(bindings);  // with custom bindings
}
```

- Only worthwhile when the same script will be executed repeatedly
- `compile(String)` and `compile(Reader)` both available
