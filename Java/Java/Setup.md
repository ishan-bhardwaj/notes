# Java

## Hello World

```
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

- Breakdown -
    - `public class HelloWorld` - In Java, all code is defined inside classes.
    - `public static void main(String[] args)` - Entry point of the program.
    - `static` - Method does not operate on any object.
    - `void` - Method does not return any value.
    - `System.out.println("Hello, World!");` - Prints text to the console.

## Compilation & Execution

- Compile - `javac HelloWorld.java` - converts source code to bytecode stored in `.class` files.
- Run - `java HelloWorld` - launches the JVM which loads class files and executes bytecode.

> [!NOTE]
> Java follows Write Once, Run Anywhere (WORA) — compiled bytecode runs on any JVM.

> [!TIP]
> For a single source file, you can skip manual compilation - `java HelloWorld.java`
> (Compiles and runs automatically without leaving `.class` files).

- Unix Executable Trick -
    - Rename file (remove `.java`) - `mv HelloWorld.java hello`
    - Make executable - `chmod +x hello`
    - Add shebang on top - `#!/path/to/jdk/bin/java --source 21`
    - Run with - `./hello`

## Java 21 Preview Features

- Hello World with simplified syntax -
```
class HelloWorld {
    void main() {
        System.out.println("Hello, World!");
    }
}
```

- Run with -
```
java --enable-preview --source 21 HelloWorld.java
```

- Even class can be omitted if only one entry point -
```
void main() {
   System.out.println("Hello, World!");
}
```

## JShell

- Provides a Read-Eval-Print Loop (REPL) for experimenting with Java code.
- Start with - `jshell`
- Features -
    - Tab completion for commands and identifiers.
    - Import suggestions - type class name + `Shift` + `Tab` + `I`.

## Comments in Java

- Single line - `// comment`
- Multi-line - `/* comment */`
- Documentation (Javadoc) - `/** comment */`

