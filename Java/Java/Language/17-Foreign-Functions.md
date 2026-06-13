# Foreign Functions and Memory API

## JNI Overview

### When to Use Native Code

- Access to system features or devices not exposed through the Java platform
- Reuse of substantial existing native codebases (e.g. GPU AI/ML libraries in C++)
- Performance is rarely the reason today — JIT compiler is very effective
- Drawbacks:
    - Must supply platform-specific shared libraries for every target OS
    - No memory safety — invalid pointers can corrupt the program or OS
    - Since Java 22, must explicitly opt in to native access

```bash
# Non-modular program
java --enable-native-access=ALL-UNNAMED MyProgram

# Modular program
java --enable-native-access=module1,module2 MyProgram
```

### JNI Steps

1. Declare the method with the `native` keyword in a Java class
2. Run `javac -h <dir>` to generate a C header file
3. Implement the C function using the generated header
4. Compile into a platform-specific shared library (`.so` on Linux, `.dll` on Windows)
5. Load the library in Java with `System.loadLibrary`

```java
// Step 1: Java declaration
class HelloNative {
    public static native int printf(String str);
}

// Step 5: Load in static initializer
static {
    System.loadLibrary("HelloNative");
}
```

C naming convention for native methods:
- Full class name with `.` replaced by `_`
- Prefix `Java_`
- Eg - `Java_v2ch13_jnidemo_HelloNative_printf`
- Overloaded native methods append `__` + encoded argument types

C implementation pattern:

```c
#include "v2ch13_jnidemo_HelloNative.h"
#include <stdio.h>

JNIEXPORT jint JNICALL Java_v2ch13_jnidemo_HelloNative_printf(
        JNIEnv *env, jclass cl, jstring jstr) {
    const char *cstr = (*env)->GetStringUTFChars(env, jstr, NULL);
    int n = printf(cstr);
    (*env)->ReleaseStringUTFChars(env, jstr, cstr);
    return n;
}
```

- `env` — pointer to C API for native calls; needed for all JNI operations
- `cl` — descriptor of the Java class (for static methods); use `jobject` for instance methods
- Java `String` → C `jstring` (opaque); convert with `GetStringUTFChars` / `GetStringChars`
- Must call `ReleaseStringUTFChars` when done so the GC can reclaim the string

Compile to shared library:

```bash
# Linux (gcc)
gcc -fPIC -I $JAVA_HOME/include -I $JAVA_HOME/include/linux \
    -shared -o libHelloNative.so HelloNative.c

# Windows (MSVC)
cl -I %JAVA_HOME%\include -I %JAVA_HOME%\include\win32 \
    -LD HelloNative.c -FeHelloNative.dll
```

Set library path on Linux:

```bash
export LD_LIBRARY_PATH=v2ch13/jnidemo:$LD_LIBRARY_PATH
# or
java -Djava.library.path=v2ch13/jnidemo MyProgram
```

> [!NOTE]
> JNI is likely to be deprecated in a future Java version. Prefer the FFM API for new projects.

---

## FFM API (Foreign Functions and Memory)

### Hello World with FFM

```java
import java.lang.foreign.*;
import java.lang.invoke.*;

Linker linker = Linker.nativeLinker();
MethodHandle printf = linker.downcallHandle(
    linker.defaultLookup().findOrThrow("printf"),
    FunctionDescriptor.of(ValueLayout.JAVA_INT, ValueLayout.ADDRESS));

try (Arena arena = Arena.ofConfined()) {
    MemorySegment str = arena.allocateFrom("Hello, Foreign World!\n");
    int result = (int) printf.invoke(str);
}
```

- No C code needed — all bridging done in Java
- `Linker` — looks up and wraps C library functions as `MethodHandle` objects
- `Arena` — manages lifetime of off-heap memory
- `MemorySegment` — contiguous block of memory passed to/from C functions

---

## Arenas

| Type | Multi-thread access | How closed |
|---|---|---|
| Confined | No (owner thread only) | `close()` |
| Shared | Yes | `close()` |
| Automatic | Yes | Garbage collector |
| Global | Yes | Never |

```java
// Confined (most common; use in try-with-resources)
try (Arena arena = Arena.ofConfined()) {
    MemorySegment s = arena.allocateFrom("hello");
} // memory freed here

// Shared (explicit close required)
Arena shared = Arena.ofShared();
// ... use in multiple threads ...
shared.close();

// Automatic (GC decides when to free)
MemorySegment s = Arena.ofAuto().allocate(layout);

// Global (never freed)
MemorySegment s = Arena.global().allocate(layout);
```

- Cannot deallocate individual segments — must close the arena that allocated them
- Accessing a segment after its arena is closed throws an exception
- Confined arenas must be opened and closed on the same thread

---

## Memory Segments

### Allocating

```java
// UTF-8 string (null-terminated)
MemorySegment cStr = arena.allocateFrom(javaString);

// Primitive array from layout + count
MemorySegment floats = arena.allocate(ValueLayout.JAVA_FLOAT, sampleCount);

// From existing Java array
float[] samples = { 1.0f, 2.0f, 3.0f };
MemorySegment floats = arena.allocateFrom(ValueLayout.JAVA_FLOAT, samples);

// Struct layout
MemorySegment info = arena.allocate(SF_INFO);
```

### Reading and Writing

```java
// String (null-terminated UTF-8)
String s = cStr.getString(0);                              // offset 0

// Primitive at byte offset
float f = seg.get(ValueLayout.JAVA_FLOAT, byteOffset);
seg.set(ValueLayout.JAVA_FLOAT, byteOffset, 0f);

// Primitive at array index
float f = seg.getAtIndex(ValueLayout.JAVA_FLOAT, i);
seg.setAtIndex(ValueLayout.JAVA_FLOAT, i, 0f);

// Entire segment → Java array
float[] arr = seg.toArray(ValueLayout.JAVA_FLOAT);

// Stream of element segments
DoubleStream ds = seg.elements(ValueLayout.JAVA_FLOAT)
    .mapToDouble(m -> m.get(ValueLayout.JAVA_FLOAT, 0));
```

- Segments do not retain layout information — must provide layout on every access
- `ValueLayout` constants: `JAVA_INT`, `JAVA_LONG`, `JAVA_FLOAT`, `JAVA_DOUBLE`, `ADDRESS`

---

## Memory Layout

### Struct Layout

```java
import static java.lang.foreign.ValueLayout.*;

StructLayout SF_INFO = MemoryLayout.structLayout(
    JAVA_LONG.withName("frames"),
    JAVA_INT.withName("samplerate"),
    JAVA_INT.withName("channels"),
    JAVA_INT.withName("format"),
    JAVA_INT.withName("sections"),
    JAVA_INT.withName("seekable")
);

// Get byte offset of a field
long offset = SF_INFO.byteOffset(PathElement.groupElement("channels"));
int channels = info.get(JAVA_INT, offset);
```

### Sequence (Array) Layout

```java
SequenceLayout arrayOfPoints = MemoryLayout.sequenceLayout(count,
    MemoryLayout.structLayout(
        JAVA_FLOAT.withName("x"),
        JAVA_FLOAT.withName("y")));

// Offset of element i's y-coordinate
long offset = arrayOfPoints.byteOffset(
    PathElement.sequenceElement(i),
    PathElement.groupElement("y"));
```

### Layout Hierarchy

- `ValueLayout` — primitive value (`OfInt`, `OfLong`, `OfFloat`, `OfDouble`)
- `AddressLayout extends ValueLayout` — pointer/address
- `StructLayout extends GroupLayout` — C struct
- `UnionLayout extends GroupLayout` — C union
- `SequenceLayout` — C array
- `PaddingLayout` — alignment padding bytes

### Canonical Type Mappings

```java
Map<String, MemoryLayout> types = Linker.nativeLinker().canonicalLayouts();
// "size_t" → JAVA_LONG on 64-bit; "int" → JAVA_INT; "void*" → ADDRESS
```

---

## Looking Up and Invoking Foreign Functions

### Symbol Lookup

```java
Linker linker = Linker.nativeLinker();

// Standard library functions (libc, libm on Linux)
SymbolLookup stdlib = linker.defaultLookup();

// Specific library
SymbolLookup sndfile = SymbolLookup.libraryLookup(
    "/lib/x86_64-linux-gnu/libsndfile.so", arena);
// Library unloaded when arena closes

// System.loadLibrary-loaded libraries
SymbolLookup loader = SymbolLookup.loaderLookup();

// Look up a symbol
MemorySegment addr = stdlib.findOrThrow("printf");   // throws NoSuchElementException if absent
Optional<MemorySegment> addr = stdlib.find("printf"); // returns empty Optional if absent
```

### Downcall Handles

```java
// int printf(const char *format, ...)
MethodHandle printf = linker.downcallHandle(
    stdlib.findOrThrow("printf"),
    FunctionDescriptor.of(JAVA_INT, ADDRESS));

// void free(void *ptr)
MethodHandle free = linker.downcallHandle(
    stdlib.findOrThrow("free"),
    FunctionDescriptor.ofVoid(ADDRESS));

// SNDFILE* sf_open(const char *path, int mode, SF_INFO *sfinfo)
MethodHandle sf_open = linker.downcallHandle(
    sndfile.findOrThrow("sf_open"),
    FunctionDescriptor.of(ADDRESS, ADDRESS, JAVA_INT, ADDRESS));
```

- Pointer parameters and pointer return types use `ValueLayout.ADDRESS`
- Must describe the signature accurately — the linker cannot verify it; incorrect types can crash the JVM
- Foreign pointer return values are wrapped in a zero-length `MemorySegment`; use `reinterpret(size)` to give it bounds

```java
MemorySegment block = (MemorySegment) malloc.invoke(blockSize);
block = block.reinterpret(blockSize);  // dangerous — must know correct size
```

### Invoking

```java
int result = (int) printf.invoke(str);
MemorySegment file = (MemorySegment) sf_open.invoke(cPath, SFM_READ, info);
```

---

## Callbacks (Upcalls)

To pass a Java method as a C callback:

```java
// 1. The Java callback method
static int comp(MemorySegment a, MemorySegment b) {
    return Float.compare(a.get(JAVA_FLOAT, 0), b.get(JAVA_FLOAT, 0));
}

// 2. Get a MethodHandle to the Java method
MethodHandle compHandle = MethodHandles.lookup()
    .findStatic(UpcallDemo.class, "comp",
        MethodType.methodType(int.class, MemorySegment.class, MemorySegment.class));

// 3. Generate C stub that calls the Java method
MemorySegment compStub = linker.upcallStub(
    compHandle,
    FunctionDescriptor.of(JAVA_INT,
        ADDRESS.withTargetLayout(JAVA_FLOAT),
        ADDRESS.withTargetLayout(JAVA_FLOAT)),
    arena);

// 4. Pass the stub as a callback argument
qsort.invoke(csamples, samples.length, JAVA_FLOAT.byteSize(), compStub);
```

- `upcallStub` generates native stub code that bridges C → Java
- `withTargetLayout` on `ADDRESS` tells the linker the layout of the pointed-to memory — required for the callback to read values via `get`
- The stub's memory is reclaimed when the arena closes

---

## Advanced Topics

### Variable Handles for Struct Access

```java
// Instead of computing byte offsets manually:
VarHandle channelsHandle = SF_INFO.varHandle(PathElement.groupElement("channels"));
int channels = (int) channelsHandle.get(info, 0L);

// Array of structs with open path element
VarHandle xHandle = arrayOfPointLayout.varHandle(
    PathElement.sequenceElement(),    // open element → becomes a coordinate
    PathElement.groupElement("x"));

float xi = (float) xHandle.get(segment, 0L, (long) i);
```

- `PathElement.sequenceElement()` (no index) — open path element; adds a `long` coordinate to the variable handle
- `PathElement.dereferenceElement()` — follow a pointer to its target layout

### Calling Varargs Functions

```java
// int printf(const char *format, ...);
MethodHandle printf = linker.downcallHandle(
    stdlib.findOrThrow("printf"),
    FunctionDescriptor.of(JAVA_INT, ADDRESS, ADDRESS, JAVA_INT),
    Linker.Option.firstVariadicArg(1));  // variadic args start at index 1
```

- Must create a separate `MethodHandle` for each unique argument pattern
- `firstVariadicArg(n)` specifies where the fixed args end

### Capturing `errno`

```java
MethodHandle fopen = linker.downcallHandle(
    stdlib.findOrThrow("fopen"),
    FunctionDescriptor.of(ADDRESS, ADDRESS, ADDRESS),
    Linker.Option.captureCallState("errno"));

StructLayout captureStateLayout = Linker.Option.captureStateLayout();
MemorySegment captureState = arena.allocate(captureStateLayout);

MemorySegment result = (MemorySegment) fopen.invoke(
    captureState, arena.allocateFrom("file.txt"), arena.allocateFrom("r"));

VarHandle errnoHandle = captureStateLayout.varHandle(PathElement.groupElement("errno"));
int errno = (int) errnoHandle.get(captureState, 0L);
```

---

## FFM vs JNI Comparison

| Aspect | JNI | FFM API |
|---|---|---|
| Extra C code required | Yes — wrapper functions | No |
| Platform library compilation | Yes — per OS | No |
| Memory safety | None — C pointers | Segment bounds checked |
| Ergonomics | Low — C/Java impedance | High — pure Java |
| Stability | Stable (since Java 1.1) | Standard from Java 22 |
| Future | Likely to be deprecated | Preferred going forward |
