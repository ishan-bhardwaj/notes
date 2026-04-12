# Execution Flow

- `Employee.java` -

```
class Employee {
    String name;
    int age;

    Employee(String n, int a) {
        name = n;
        age = a;
    }

    String getName() {
        return name;
    }

    static void main() {
        var john = new Employee("John", 30);
        IO.println("Employee name: " + john.getName());
    }
}
```

## Compilation

- `javac Employee.java` - converts Java source to _bytecode_ (`.class` file)
- `.class` file contains -
  1. __Constant Pool__ (symbol table), eg -
  
    ```
    #1 = Methodref Employee.<init>
    #2 = String "John"
    #3 = Fieldref Employee.name
    ```

  2. Bytecode instructions, eg - 
 
    ```
    new #Employee
    invokespecial <init>
    ```

- Use [`javap`](Java\Java\JDK Tools\javap.md) to look inside bytecode.

## Class Loading

- `java Employee` - JVM starts and loads classes.
- __ClassLoader__ system -
  - Bootstrap ClassLoader - JDK core
  - Platform ClassLoader
  - Application ClassLoader (your code)
- Steps -
  - JVM reads `Employee.class` from disk
  - loads byte array into memory
  - stores it in _Metaspace_
- `java -Xlog:class+load=info Employee`
  - Logging levels - `info`, `debug`, `trace`

## Class Linking

- Steps -
  1. __Bytecode Verification__ -
    - Before execution, JVM verifies - 
      - Stack safety
      - Type correctness
      - No illegal memory accesses
      - Control flow validity
    - If bytecode is corrupted, it will show - `java.lang.VerifyError`

  2. __Preparation__ -
    - JVM allocates memory for -
      - static variables
      - class structures

  3. __Resolution__ -
    - converting symbols to actual references
    - Example -
      - Symbol in bytecode - `#5 = Fieldref Employee.name`
      - JVM resolves it to - `memory offset inside object layout`
    - `-XX:+PrintCompilation` - prints methods compiled and optimized after resolution

## Class Initialization

- JVM runs `static {}` blocks.

## Execution

- JVM executes `main()` method.
- When `main()` starts, JVM creates a __Stack frame__ which contains -
  - local variables
  - operand stack
  - return address
  - intermediate computation values

## Object Creation (`new`)

- Finally, execution hits - `Employee john = new Employee("John", 30);`
- Steps - 
  1. Class check -
    - JVM checks if class is loaded.
    - If not, then it loads it.
  2. Heap Allocation -
    - JVM allocates memory in heap - `[ HEADER | FIELDS | PADDING ]`
    - Object `HEADER` contains -
      - Mark Word (GC + lock info)
      - Klass pointer (points to Employee.class)
  3. Default Initialization -
    - Before constructor -
      - `name = null`
      - `age = 0`
  4. Constructor Execution - `Employee("John", 30)`
    - JVM pushes parameters onto stack
    - runs bytecode of constructor
    - assign fields
  5. Reference Assignment -
    - Stack - `john → [heap address (opaque)]`
  6. Method call -
    - `john.getName()`
    - Stack loads reference - `john → object reference`
    - Dereference via klass pointer in the object header - `[ klass pointer → Employee.class ]`
    - Method lookup - JVM finds `Employee.getName()` in the method table.
    - Execution -
      - Creates new stack frame -
        - return address
        - local variables
        - operand stack

## Printing Output

- `IO.println(john.getName())` -
  - `getName()` executes
  - returns `String` reference
  - `println` converts to character stream
  - OS `syscall` writes to stdout

## Garbage Collection

- If object becomes unreachable i.e. `john = null` then -
  - object becomes GC eligible
  - marked during GC cycle
  - memory reclaimed

> [!NOTE]
> A "Reference" is a JVM-managed handle (compressed or direct pointer internally) which we cannot access directly.
