# `javap`

- `javap` is a class file disassembler - decodes bytecode (binary) file according to the JVM Class File Specification.

- `javap` dissambled `.class` file structure -
  ```
  ClassFile
  ├── 1. File Metadata (javap-only)
  ├── 2. File Header (Magic + Versions)
  ├── 3. Class Declaration (flags + identity)
  ├── 4. Constant Pool (Symbol Table)
  ├── 5. Interfaces Section
  ├── 6. Fields Section
  ├── 7. Methods Section
  ├── 8. Code Attribute (Bytecode Engine)
  ├── 9. Class-Level Attributes
  └── 10. Bootstrap + InnerClasses (invokedynamic machinery)
  ```

- `Employee.java` -

```
package com.playground;

public class Employee {
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

## File Metadata

```
Classfile /C:/Users/.../src/main/java/com/playground/Employee.class
  Last modified 12 Apr 2026; size 1034 bytes
  SHA-256 checksum aea742ea...
  Compiled from "Employee.java"
```

- This section is _not_ part of the `class` file.
- It is added by `javap` -

  | Item                   | Source                 |
  | ---------------------- | ---------------------- |
  | `Classfile <filepath>` | OS                     |
  | `Last modified`        | OS                     |
  | `SHA-256`              | computed by `javap`    |
  | `Compiled from`        | `SourceFile` attribute |

- Only `Compiled from` exists inside the `.class` file - 
  
  ```
  SourceFile: "Employee.java"
  ```

  - Purpose of `SourceFile` attribute -
    - Debuggers map bytecode to source file.
    - Stack traces show file name.
  - Without this attribute, stack traces show `Unknown Source`.

## File Header

```
minor version: 0
major version: 69
```

- Actual `.class` file has hex format -
  - Starts with _magic number_ - `CAFEBABE`
  - JVM first checks the magic number and instantly rejects non-class files by throwing `ClassFormatError`.
  - This check happens before class loading even begins.

- Major version defines the bytecode instruction set the class uses.
  | Major  | Java        |
  | ------ | ----------- |
  | 52     | Java 8      |
  | 55     | Java 11     |
  | 61     | Java 17     |
  | 65     | Java 21     |
  | 69     | Java 25     |

- JVM checks this field before loading the class.
- If Major version is greater than VM version then it throws `UnsupportedClassVersionError`.

## Class Declaration

```
flags: ACC_PUBLIC, ACC_SUPER
this_class: Employee
super_class: Object
```

- The flags field is a bitmask.

  | Flag             | Hex    | Meaning                          |
  | ---------------- | ------ | -------------------------------- |
  | `ACC_PUBLIC`     | 0x0001 | `public` class                   |
  | `ACC_FINAL`      | 0x0010 | `final` class                    |
  | `ACC_SUPER`      | 0x0020 | modern `invokespecial` semantics |
  | `ACC_INTERFACE`  | 0x0200 | `interface`                      |
  | `ACC_ABSTRACT`   | 0x0400 | `abstract` class                 |
  | `ACC_SYNTHETIC`  | 0x1000 | compiler generated               |
  | `ACC_ANNOTATION` | 0x2000 | annotation type                  |
  | `ACC_ENUM`       | 0x4000 | enum                             |
  | `ACC_MODULE`     | 0x8000 | `module-info.class`              |

> [!NOTE]
> `ACC_SUPER` tells JVM to use modern super lookup algorithm.

- `this_class` and `super_class` values are symbolic references i.e. indexes in constant pool.
- Resolution only happens later during _Linking_ - JVM's lazy linking model.

## Interfaces / Fields / Methods count

```
interfaces: 0, fields: 2, methods: 3, attributes: 3
```

- These counts allow JVM to stream parse the file without backtracking.
- JVM reads the file sequentially.

## Constant Pool

```
Constant pool:
  #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
  #2 = Class              #4             // java/lang/Object
  #3 = NameAndType        #5:#6          // "<init>":()V
  ...
  ...
  #53 = Class              #54            // java/lang/invoke/MethodHandles
  #54 = Utf8               java/lang/invoke/MethodHandles
  #55 = Utf8               Lookup
```

- The constant pool is a per-class symbol resolution database + runtime linking contract + literal storage system.
- The JVM uses it for 3 phases -
  | Phase     | Usage                                        |
  | --------- | -------------------------------------------- |
  | Loading   | Reads symbolic references                    |
  | Linking   | Resolves symbolic → runtime pointers         |
  | Execution | Uses resolved entries for bytecode execution |

- __Internal memory model__ -
  - Each entry is - `#n = TYPE + DATA`
  - Physically -
    - `1 byte` → type tag
    - `2–4 bytes` → index / raw value
    - or nested references to other CP entries

> [!NOTE]
> Constant Pool is a graph, not a list. JVM resolves by traversing this graph.
>
> Example -
> ```
> Methodref
>     ↓
> Class + NameAndType
>     ↓
> Utf8 symbols
> ```

> [!NOTE]
> Everything reduces to `Utf8`, even class names, method names, descriptors, signatures etc - all eventually point to `Utf8`.

- __Constant pool types__ -

| **javap type**       | **Meaning**                          | **Example**                                                    |
| -------------------- | ------------------------------------ | -------------------------------------------------------------- |
| `Utf8`               | raw text                             | `"Employee"`, `"name"`, `"Ljava/lang/String;"`                 |
| `Integer`            | int literal                          | `30`, `42`, `-1`                                               |
| `Float`              | float literal                        | `2.5f`, `0.0f`                                                 |
| `Long`               | long literal                         | `100L`, `-999999999L`                                          |
| `Double`             | double literal                       | `1.0`, `3.14`                                                  |
| `String`             | string literal reference             | `"John"`, `"Employee name"`                                    |
| `Class`              | reference to a class name (via Utf8) | `Employee`, `java/lang/Object`                                 |
| `NameAndType`        | (name + descriptor pair)             | `getName:()Ljava/lang/String;`                                 |
| `Fieldref`           | symbolic field access                | `Employee.name`, `Employee.age`                                |
| `Methodref`          | symbolic method call                 | `Employee.getName()`, `Object.<init>()`                        |
| `InterfaceMethodref` | interface call                       | `List.add`, `Runnable.run`                                     |
| `MethodHandle`       | typed function pointer descriptor    | `REF_invokeStatic StringConcatFactory.makeConcatWithConstants` |
| `MethodType`         | method signature descriptor          | `(Ljava/lang/String;)Ljava/lang/String;`                       |
| `InvokeDynamic`      | dynamic call site                    | `"Employee name: " + john.getName()`                           |
| `Dynamic`            | computed constant (bootstrap-based)  | computed constants via bootstrap                               |
| `Module`             | JPMS module name                     | `java.base`, `java.sql`                                        |
| `Package`            | exported package metadata            | `com.playground`                                               |

## Field Section

```
java.lang.String name;
  descriptor: Ljava/lang/String;
  flags: (0x0000)

int age;
  descriptor: I
  flags: (0x0000)
```

- This is the field table of the class.
- Defines -
  - what data exists in each object instance
  - how JVM encodes it
  - how bytecode instructions access it (getfield, putfield)
- __Descriptor system__ - JVM internal encoding - 

  | Type    | JVM Descriptor |
  | ------- | -------------- |
  | int     | I              |
  | long    | J              |
  | float   | F              |
  | double  | D              |
  | boolean | Z              |
  | char    | C              |

- __Field flags__ - 
  - `flags: (0x0000)`
  - Bitmask stored in class file.
    | Bitmask | Flag            | Meaning                      |
    | ---------------------------------- | --------------- | ---------------------------- |
    | `0x0001`                           | `ACC_PUBLIC`    | public field                 |
    | `0x0002`                           | `ACC_PRIVATE`   | private field                |
    | `0x0004`                           | `ACC_PROTECTED` | protected field              |
    | `0x0008`                           | `ACC_STATIC`    | static field                 |
    | `0x0010`                           | `ACC_FINAL`     | constant field               |
    | `0x0040`                           | `ACC_VOLATILE`  | volatile memory semantics    |
    | `0x0080`                           | `ACC_TRANSIENT` | skipped during serialization |
    | `0x1000`                           | `ACC_SYNTHETIC` | compiler-generated           |

## Constructor & Methods Section

```
com.playground.Employee(java.lang.String, int);
    descriptor: (Ljava/lang/String;I)V
    flags: (0x0000)
    Code:
      stack=2, locals=3, args_size=3
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: aload_1
         6: putfield      #7                  // Field name:Ljava/lang/String;
         9: aload_0
        10: iload_2
        11: putfield      #13                 // Field age:I
        14: return
      LineNumberTable:
        line 7: 0
        line 8: 4
        line 9: 9
        line 10: 14
```

```
java.lang.String getName();
    descriptor: ()Ljava/lang/String;
    flags: (0x0000)
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: getfield      #7                  // Field name:Ljava/lang/String;
         4: areturn
      LineNumberTable:
        line 13: 0
```

- `Code` attribute describes stack-based execution model -
  - `stack=2` - max operand stack depth
  - `locals=3` - this + 2 parameters
  - `args_size=3` - implicit JVM argument layout

- `LineNumberTable` maps return instruction back to source line for debugging.

## `BootstrapMethods` attribute

```
BootstrapMethods:
  0: #44 REF_invokeStatic java/lang/invoke/StringConcatFactory.makeConcatWithConstants:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #42 Employee name: \u0001
```

- Registry for runtime-linked bytecode operations.
- Used by `invokedynamic`.
- __String Concatenation Bootstrap__ -
  - `StringConcatFactory.makeConcatWithConstants`
  - JVM generates optimized concatenation strategy at runtime.

## `InnerClasses` attribute

```
InnerClasses:
  public static final #55= #51 of #53;    // Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
```

- Records nested class relationships.
- Required for reflection + access rules.
- Example -
  - `MethodHandles$Lookup`
  - Used by `invokedynamic` bootstrap system.