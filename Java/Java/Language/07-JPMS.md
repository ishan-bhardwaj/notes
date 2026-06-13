# The Java Platform Module System

## Why Modules

- Classes provide encapsulation at the type level; packages at the group level
- Public members of public classes are accessible everywhere — no way to reason about impact of changing them
- __Module__ — a collection of packages with explicit access control and dependency declarations
- Benefits over JAR files on classpath:
    - __Strong encapsulation__ — control which packages are accessible; unexported packages are truly hidden
    - __Reliable configuration__ — duplicate/missing classes caught at startup, not at runtime

> [!NOTE]
> The Java Platform Module System does not support module versioning or using multiple versions of the same module simultaneously.

---

## Module Anatomy

A Java platform module consists of:
- A collection of packages
- Optionally: resource files, native libraries
- A list of accessible packages (`exports`)
- A list of required modules (`requires`)

---

## Naming Modules

- Names follow the same rules as package names: letters, digits, underscores, periods
- No hierarchical relationship between modules — `com.horstmann` and `com.horstmann.corejava` are unrelated
- Convention: name a module after the top-level package it provides (Eg - module `org.slf4j` contains packages `org.slf4j`, `org.slf4j.spi`, etc.)
- This convention ensures globally unique names and prevents package name conflicts
- A given package can only be placed in one module
- No two packages in any modules (even unexported ones) can share the same name

---

## Module Declaration (`module-info.java`)

- Placed in the base directory of the module source tree (same level as the top-level package directory)
- Filename: `module-info.java` — compiled to `module-info.class`
- `module`, `requires`, `exports`, `opens`, etc. are __restricted keywords__ — only special inside module declarations
- Module declarations can be annotated (annotation `@Target` must include `ElementType.MODULE`)

```
v1ch12.hellomod/
├ module-info.java
└ com/horstmann/hello/HelloWorld.java
```

```java
module v1ch12.hellomod {
}
```

---

## Requiring Modules

```java
module v1ch12.requiremod {
    requires java.desktop;
}
```

- `java.base` is required by default — no need to declare
- `requires` is followed by a __module name__
- Module M __reads__ module N when:
    - M requires N
    - M requires a module that transitively requires N
    - N is M or `java.base`
- Requirements are NOT transitive by default — each module must explicitly declare all its requirements
- No cycles allowed in the module graph

---

## Exporting Packages

```java
module com.horstmann.greet {
    exports com.horstmann.greet;
}
```

- `exports` is followed by a __package name__
- Exported packages make their `public` and `protected` types and members accessible outside the module
- Non-exported packages are completely inaccessible outside the module — even `public` classes in them
- Unexported internal packages conventionally named `.internal`

---

## Modular JARs

- A JAR containing `module-info.class` at its root is a __modular JAR__
- Build with `javac -d outputDir ...` then `jar -c -f module.jar -C outputDir .`
- Specify main class: `jar -c -f app.jar -e com.example.Main -C outputDir .`
- Run: `java -p module.jar -m modulename/com.example.Main`
- Optional version: `jar -c -f module-1.0.jar --module-version 1.0 -C outputDir .`
    - Version not used by the module system for resolution but queryable via reflection:
        - `SomeClass.class.getModule().getDescriptor().rawVersion()`
- Split packages (same package across multiple JARs) are not allowed with modules
- The __layer__ is the module equivalent of a class loader; application modules load into the __boot layer__

---

## Reflective Access

- Non-exported packages block reflective access too — `setAccessible(true)` will throw `InaccessibleObjectException`
- Use `opens` to allow reflective access to a package's classes at runtime:

```java
module v1ch12.openpkg {
    requires com.horstmann.util;
    opens com.horstmann.places;
}
```

- `opens` enables reflective access to all instances of classes in the package
- __Open module__ — grants runtime reflective access to all packages, as if all were `opens`; only explicitly `exports`-ed packages accessible at compile time:

```java
open module v1ch12.openpkg {
    requires com.horstmann.util;
}
```

- Resources in package-matching directories require the package to be `opens`; resources elsewhere are accessible to all

### VarHandle Alternative

- Future libraries may use `VarHandle` instead of reflection — requires a `Lookup` object from the module owning the field
- `MethodHandles.lookup()` in a module captures that module's access rights — can pass to another module

---

## Automatic Modules

- Any JAR placed on the __module path__ without a `module-info.class` becomes an __automatic module__
- Automatic module properties:
    - Implicitly requires all other modules
    - All packages are exported and opened
    - Module name from `Automatic-Module-Name` entry in `META-INF/MANIFEST.MF` if present
    - Otherwise derived from JAR filename: trailing version number removed, non-alphanumeric sequences replaced with `.`
- Allows existing pre-module JARs to be referenced from explicit module declarations
    - Eg - `commons-csv-1.9.0.jar` → module name `commons.csv`
    - `commons-csv-1.10.0.jar` with `Automatic-Module-Name: org.apache.commons.csv` → module name `org.apache.commons.csv`

> [!TIP]
> Before placing third-party JARs on the module path, check if they are modular or have `Automatic-Module-Name` set. If not, be prepared to update the module name later when they modularise.

---

## The Unnamed Module

- Any class NOT on the module path belongs to the __unnamed module__
- Unnamed module can access all other modules; all its packages are exported and opened
- Explicit modules (with `module-info.class` on module path) CANNOT access the unnamed module
- Migration is therefore __bottom-up__:
    - Java platform modularised first
    - Libraries next (automatic or explicit modules)
    - Application code last

---

## Command-Line Flags for Migration

- `--add-exports module/package=targetModule` — export a package from a module to another (use `ALL-UNNAMED` for classpath code):
java --add-exports java.sql.rowset/com.sun.rowset=ALL_UNNAMED -jar MyApp.jar

- `--add-opens module/package=targetModule` — open a package for reflective access:
java --add-opens java.base/java.lang=ALL-UNNAMED

- `--illegal-access` flag (Java 9–16 only; removed in Java 17):
    - `permit` — warn once per offense (Java 9 default)
    - `warn` — warn on each offense
    - `debug` — warn + stack trace on each offense
    - `deny` — deny all illegal access (Java 16 default)

- Options files: pass multiple options via `@filename` to avoid "command line from hell":

java @options1 @options2 -jar MyProg.jar

- Options file syntax: spaces/tabs/newlines separate options; quote args with spaces; `\` at line end merges lines; backslashes escaped; `#` for comments

---

## Transitive and Static Requirements

### `requires transitive`

```java
module java.desktop {
    requires java.prefs;
    requires transitive java.datatransfer;
    requires transitive java.xml;
}
```

- Any module requiring `java.desktop` automatically also reads `java.datatransfer` and `java.xml`
- Use when types from another module appear in your public API
- __Aggregator module__ — no packages, only transitive requirements (Eg - `java.se` requires all Java SE modules transitively)

### `requires static`

- Module must be present at compile time but is optional at runtime
- Use cases:
    - Accessing compile-time-only annotations from another module
    - Optionally using a class if available, with a `NoClassDefFoundError` fallback

---

## Importing Modules

```java
import module java.desktop;   // imports all packages in java.desktop
```

- Equivalent to importing all packages in the module
- In compact compilation units, `java.base` is imported automatically
- Name clashes from importing many packages — resolve with specific import:

```java
import module java.se;
import java.util.List;  // disambiguate
```

- `import module java.se` only works in a modular program that requires `java.se`

---

## Qualified Exporting and Opening

```java
exports sun.net to java.net.http, jdk.naming.dns;
opens com.horstmann.places to com.horstmann.util;
```

- Restricts `exports` or `opens` to a specific set of named modules
- Other modules cannot access the package
- Excessive use indicates poor modular structure — typically arises when modularising existing code

---

## Service Loading

- Module system replaces `META-INF/services` text files with module descriptor statements
- Provider module uses `provides`:

```java
module com.horstmann.greetsvc {
    exports com.horstmann.greetsvc;
    provides com.horstmann.greetsvc.GreeterService with
        com.horstmann.greetsvc.internal.FrenchGreeter,
        com.horstmann.greetsvc.internal.GermanGreeterFactory;
}
```

- Consumer module uses `uses`:

```java
module v1ch12.useservice {
    requires com.horstmann.greetsvc;
    uses com.horstmann.greetsvc.GreeterService;
}
```

- The `provides`/`uses` declarations grant the consumer access to module-private implementation classes
- Implementation packages do NOT need to be exported

---

## Tools

### `jdeps`

- Analyses dependencies of JAR files
- `jdeps -s jar1.jar jar2.jar` — summary module graph
- `jdeps --generate-module-info /outputDir jar1.jar` — generates `module-info` stubs
- `-dotoutput /dir` — generates DOT graph files (visualise with `dot -Tpng`)

### `jlink`

- Produces a self-contained application image without a separate JDK installation
- `jlink --module-path modules:$JAVA_HOME/jmods --add-modules mymodule --output /outputDir`
- Output contains `bin/java` and `lib/modules` (only the required modules)
- `bin/java --list-modules` — lists included modules
- Resulting image significantly smaller than full JDK

### `jmod`

- Builds and inspects `.jmod` files (used by JDK modules in the `jmods` directory)
- `.jmod` files can contain: class files, native libraries, commands, headers, config files, legal notices
- Use ZIP tools to inspect contents
- Only needed for linking — no need to produce `.jmod` files unless bundling native libraries