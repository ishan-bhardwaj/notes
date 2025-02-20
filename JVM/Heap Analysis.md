# Heap Analysis

## Heap Histograms

- Displays the number of instances created for each object type within an application without doing a full heap dump.
- Generate a heap histogram of live objects -
```
jcmd <pid> GC.class_histogram
```

- Include `-all` flag to display unreferenced (garbage) objects also.

- Obtain list of  objects that are eligible to be collected (dead objects) -
```
jmap -histo <pid>
```

- Force a full GC prior to seeing the histogram -
```
jmap -histo:live <pid>
```

> [!NOTE]
> Avoid collecting histograms during the test execution phase, as they can distort results by introducing GC overhead. Instead, gather them before or after the performance test run.

## Heap Dumps

- Generate heap dump -
```
jcmd <pid> GC.heap_dump /path/to/heap_dump.hprof
```

or
```
jmap -dump:live,file=/path/to/heap_dump.hprof process_id
```

- Specify `-all` at the end of the `jcmd` to include other (dead) objects.

> [!WARNING]
> Including the `live` option in jmap will force a full GC to occur before the heap is dumped.

> [!NOTE]
> Even if you don’t force a full GC, the application will be paused for the time it takes to write the heap dump.

- The `.hprof` file generated can then read using tools like `jvisualvm`, `mat` etc.

> [!TIP]
> **Shallow and Deep Object Sizes**
> - The shallow size of an object is the size of the object itself. If the object contains a reference to another object, the 4 or 8 bytes of the reference is included, but the size of the target object is not included.
> - The deep size of an object includes the size of the object it references.

- Objects that retain a large amount of heap space are often called the **dominators** of the heap.

- Heap analysis tools provide a way to find the **GC roots** of a particular object. GC (Garbage Collection) Roots are special objects that are always reachable and serve as starting points for the garbage collector (GC) to trace object references. Any object that is reachable from a GC root is considered alive and won't be garbage collected.

- Types of GC roots -
    - Objects referenced by static fields of loaded classes (e.g., static variables) -
    ```
    public class Example {
        static Object staticVar = new Object(); // GC Root
    }
    ```

    - Local variables and references in active threads (method parameters, stack variables) -
    ```
    public void method() {
        Object localVar = new Object();     // This stays alive as long as the method is running
    }
    ```

    - Live thread objects are GC roots because they need to run.
    ```
    Thread t = new Thread(() -> {
        Object threadVar = new Object(); // threadVar is alive as long as the thread runs
    });
    ```

    - Objects locked by synchronized blocks or wait() / notify()
    - Objects referenced from native (C/C++) code through the Java Native Interface (JNI)

> [!TIP]
> As a general rule of thumb, start with collection objects (e.g., `HashMap`) rather than the entries (e.g., `HashMap$Entry`), and look for the biggest collections. Collection classes are the most frequent cause of a memory leak: the application inserts items into the collection and never frees them.

## Out-of-Memory Errors

- The JVM throws an out-of-memory error under these circumstances -
a. The Java heap itself is out of memory: the application cannot create any additional objects for the given heap size -
    - Error message -
    ```
    Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
    ```
    - Either the number of live objects that it is holding onto cannot fit in the heap space configured for it, or the application may have a memory leak: it continues to allocate additional objects without allowing other objects to go out of scope.

b. The JVM is spending too much time performing GC -
    - Error message -
    ```
    Exception in thread "main" java.lang.OutOfMemoryError: GC overhead limit exceeded
    ```
    - This error is thrown when all of the following conditions are met -
        1. The amount of time spent in full GCs exceeds the value specified by the `-XX:GCTimeLimit=N flag`. The default value is `98` (i.e., if 98% of the time is spent in GC).
        2. The amount of memory reclaimed by a full GC is less than the value specified by the `-XX:GCHeapFreeLimit=N` flag. The default value is `2`, meaning that if less than 2% of the heap is freed during the full GC, this condition is met.
        3. The preceding two conditions have held true for five consecutive full GC cycles (that value is not tunable).
        4. The value of the `-XX:+UseGCOverheadLimit` flag is `true` (which it is by default).

> [!NOTE]
> As a last-ditch effort to free memory, if the first two conditions hold for four consecutive full GC cycles, then all soft references in the JVM will be freed before the fifth full GC cycle.

c. No native memory is available for the JVM -
    - occurs for reasons unrelated to the heap.
    - Eg - In a 32-bit JVM, the maximum size of a process is 4 GB. Specifying a very large heap > 4GB will bring the application size dangerously close to that limit.

d. The metaspace is out of memory -
    - occurs because the metaspace native memory is full.
    - Because metaspace has no maximum size by default, this error typically occurs because you’ve chosen to set the maximum size.
    - If the metaspace is full, the error text will appear like this -
    ```
    Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
    ```
    - Two root causes -
        1. The application uses more classes than can fit in the metaspace you’ve assigned.
        2. `Classloader` memory leak -
            - occurs most frequently in a server that loads classes dynamically.
            - Each time the application is changed, it must be redeployed: a new classloader is created to load the new classes, and the old classloader is allowed to go out of scope. Once the classloader goes out of scope, the class metadata can be collected.
            - If the old classloader does not go out of scope, the class metadata cannot be freed, and eventually the metaspace will fill up and throw an out-of-memory error. 
            - In this case, increasing the size of the metaspace will help, but ultimately that will simply postpone the error.
            - To debug this situation, find all the instances of the `ClassLoader` class, and trace their GC roots to see what is holding onto them.

> [!TIP]
> Classloader leaks are the reason you should consider setting the maximum size of the metaspace. Left unbounded, a system with a classloader leak will consume all the memory on your machine.

- When out-of-memory error is thrown, the JVM does not exit, because the exception affects only a single thread in the JVM. 
    - Consider JVM with two threads performing a calculation. One of them may get the `OutOfMemoryError`.
    - By default, the thread handler for that thread will print out the stack trace, and that thread will exit.
    - But the JVM still has another active thread, so the JVM will not exit. And because the thread that experienced the error has terminated, a fair amount of memory can likely now be claimed on a future GC cycle: all the objects that the terminated thread referenced and that weren’t referenced by any other threads. 
    - So, the surviving thread will be able to continue executing and will often have sufficient heap memory to complete its task.
    - If instead you want the JVM to exit whenever the heap runs out of memory, you can set the `-XX:+ExitOnOutOfMemoryError` flag, which by default is `false`.

## Automatic Heap Dumps 

- Out-of-memory errors can occur unpredictably, making it difficult to know when to get a heap dump. 
- Several JVM flags can help -
    - `-XX:+HeapDumpOnOutOfMemoryError` - Turning on this flag (which is false by default) will cause the JVM to create a heap dump whenever an out-of-memory error is thrown.

    - `-XX:HeapDumpPath=<path>` - Specifies the location where the heap dump will be written; the default is `java_pid<pid>.hprof` in the application’s current working directory. The path can specify either a directory (in which case the default filename is used) or the name of the actual file to produce.

    - `-XX:+HeapDumpAfterFullGC` - generates a heap dump after running a full GC.
    - `-XX:+HeapDumpBeforeFullGC` - generates a heap dump before running a full GC.

> [!NOTE]
> When multiple heap dumps are generated (e.g., because multiple full GCs occur), a sequence number is appended to the heap dump filename.



