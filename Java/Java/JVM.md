## JVM

- The **Java Virtual Machine (JVM)** is a virtual simulation of a physical computer that executes compiled Java programs (bytecode). The JVM runs as an application on top of an operating system and provides an environment for Java programs.
- Because they use virtual machines, Java programs are platform-independent and can be executed on different hardware and operating systems according to the **WORA (Write Once Run Anywhere)** principle.
- When the JVM executes a program, it translates the bytecode into platform native code.

## JVM Components

### Class Loader

- The **class loader subsystem** loads the Java bytecode for execution, verifies it and then allocates memory for the bytecode.
- It is a part of JRE responsible for dynamic loading classes into memory. 
- To verify bytecode there is a module called **bytecode verifier**. It checks that the instructions don’t require any dangerous actions like accessing private fields and methods of classes and objects.
- The task of a class loader is finding the needed class through `.class` files from a disc and loading a representing object into the RAM. However, classes are not loaded in bulk mode on the application startup. 
- A class loader loads them on-demand during an interpretation starting with a class containing the `main` method. - The on-demand approach means that the class will be loaded on its first invocation. It can be a constructor call, e.g. `new MyObject()` or a static reference to a class, e.g. `System.out`.
- A class loader concept is represented by `java.lang.ClassLoader` abstract class. There are 3 standard ClassLoader implementations -
    - Bootstrap loads JDK internal classes e.g. java.util package.
    - Extension/Platform loads classes from JDK extensions.
    - Application/System loads classes from application classpath.

- What comes first if classes are loaded by a class loader and the ClassLoader itself is a class? 
    - First, JRE creates the Bootstrap ClassLoader which loads core classes.
    - Then, Extension ClassLoader is created with Bootstrap as a parent. It loads classes for extensions if such exist.
    - Finally, the Application ClassLoader is created with Extension as a parent. It is responsible for loading application classes from a classpath.

- Each class loaded in memory is identified by a fully-qualified class name and ClassLoader that loaded this class. Moreover, Class has a method getClassLoader that returns the class loader which loads the given class.

- Example -
```
import java.sql.SQLData;
import java.util.ArrayList;

public class Main {
    public static void main(String[] args) {
        ClassLoader listClassLoader = ArrayList.class.getClassLoader();
        ClassLoader sqlClassLoader = SQLData.class.getClassLoader();
        ClassLoader mainClassLoader = Main.class.getClassLoader();

        System.out.println(listClassLoader);                // null
        System.out.println(sqlClassLoader.getName());       // platform
        System.out.println(mainClassLoader.getName());      // app
    }
}
```

- **Delegation Model** -
- When the class loader receives a request for loading a class -
    1. Check if the class has already been loaded.
    2. If not, delegate the request to a parent.
    3. If the parent class returns nothing, it attempts to find the class in its own classpath.
    
### Runtime Data Areas

- The **Runtime data areas** represents JVM memory -
    - **PC register** holds the address of the currently executing instruction.
    - **Stack area** is a memory place where method calls and local variables are stored.
    - **Native method** stack stores native method information.
    - **Heap** stores all created objects (instances of classes).
    - **Method area** stores all the class level information like class name, immediate parent class name, method information and all static variables.

> [!TIP]
> Every thread has its own PC register, stack, and native method stack, but all threads share the same heap and method area.

### Execution Engine

- **The Execution engine** is responsible for executing the program (bytecode). It interacts with various data areas of the JVM when executing a bytecode.
- Consists of -
    - **Bytecode Interpreter** interprets the bytecode line by line and executes it (rather slowly).
    - **Just-in-time Compiler (JIT compiler)** translates bytecode into native machine language while executing the program (it executes the program faster than the interpreter).
    - **Garbage collector** cleans unused objects from the heap.

> [!NOTE]
> Different JVM implementations can contain both a bytecode interpreter and a just-in-time compiler, or only one of them.

> [!NOTE]
> Other important parts of JVM for execution includes -
> - **Native method interface** provides an interface between Java code and the native method libraries.
> - **Native method library** consists of (C/C++) files that are required for the execution of native code

