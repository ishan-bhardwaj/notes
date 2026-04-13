## Just In Time (JIT) Compilation

- JVM monitors which part of the code (methods or code blocks) are run more often, it will then compile that code into native machine code (executed directly by the OS) to speed up its execution.
- Therefore, some part of the application will be executed as interpreted bytecode, and the other part as compiled native machine code.
- The process that compiles the bytecode to native machine code runs on a separate thread. While the compilation is taking place, JVM will continue to use the interpreted version and once the compilation is complete, JVM switches to use the native compiled version.
- Find out methods or code blocks are being natively compiled when the app runs - `-XX:+PrintCompilation`
- Sample output -

```
24    1       3       java.lang.StringLatin1::hashCode (42 bytes)
25    3       3       java.lang.Object::<init> (1 bytes)
25    2       3       java.util.concurrent.ConcurrentHashMap::tabAt (22 bytes)
25    5       3       java.lang.String::isLatin1 (19 bytes)
...
44   63   !   3       java.lang.ref.ReferenceQueue::poll (28 bytes)
41   48     n 0       java.lang.invoke.MethodHandle::invokeBasic()L (native)
...
82  187 %     4       PrimeNumbers::isPrime @ 2 (35 bytes)
85  190       3       PrimeNumbers::getNextPrimeAbove (43 bytes)   made not entrant
85  192       3       PrimeNumbers::getNextPrimeAbove (43 bytes)
```

- Explanation -
  - 1st column - number of milliseconds since the JVM started.
  - 2nd column - order in which a method or code block is compiled.
  - Some columns may be missing, or they may have
    - `n` means native method
    - `s` means synchronized methods.
    - `!` means there is some exception handling going on.
    - `%` means code is natively compiled and is now running in special part of memory called code cache which is the most optimal way possible.
  - Next column has a number between 0 to 4 representing levels of compilation.
    - `0` means no compilation & the code is just interpreted.
    - `1` (low) to `4` (highest) represents progressively deeper level of compilation has occured.
  - Final column - line of code that is actually being compiled.

> [!TIP]
> Code with highest level of compilation (i.e. 4) but without `%` means it is compiled to native machine code but not executed in code cache.

- Two compilers built into JVM -

  - C1 / Client - performs 1 to 3 levels of compilation.
  - C2 / Server - performs 4th level of compilation.

- Short-lived applications are generally considered as "client", whereas long running applications are considered as "server".

- Client / C1 - compiles methods quickly, but emits machine code that is less optimized than the server compiler.
- Server / C2 - often takes more time (and memory) to compile the same methods, but generates better optimized machine code than the code produced by the client compiler.

- Client & Server compilers serve two different use cases -
  - Client - quick startup because of less compilation overhead and smaller memory footprint is more important than steady-state performance i.e. better for short-lived jobs.
  - Server - steady–state performance is more important than a quick startup i.e. better for long-running jobs.

> [!NOTE]
> 32-bit JVM has only C1 compiler, whereas 64-bit JVM has both C1 & C2 compilers.

- JVM compiler flags -

  - `-client` - forces JVM to use client compiler only.
  - `-server` - selects the 32-bit server compiler.
  - `-d64` - selects the 64-bit server compiler.

> [!NOTE]
> On 32-bit OS, we can choose `-client` & `-server` flags, but using `-d64` will throw an error. On 64-bit OS, we always get 64-bit server compiler - no matter the flag we choose.

- `-XX:-TieredCompilation` - turns off tiered compilation i.e. JVM will run in interpreted mode only - probably good for applications that run only one line of code like a serverless application. Tiered compilation is on by default.

- To log these info in a separate file - `-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation` - this will generate a file named - `hotspot_pid<pid>.log`.

- As the code cache has a limited size, when it gets full then some code is removed from it to make space for the next one to be inserted and the removed method could be recompiled and re-added later on.

- When the code cache is full, we get a warning - `VM warning: CodeCache is full. Compiler has been disabled.`

- Size of the code cache - `-XX:+PrintCodeCache`

```
CodeHeap 'non-profiled nmethods': size=120000Kb used=68Kb max_used=68Kb free=119931Kb
 bounds [0x00000273101e0000, 0x0000027310450000, 0x0000027317710000]
CodeHeap 'profiled nmethods': size=120000Kb used=248Kb max_used=248Kb free=119751Kb
 bounds [0x0000027308cb0000, 0x0000027308f20000, 0x00000273101e0000]
CodeHeap 'non-nmethods': size=5760Kb used=1107Kb max_used=1121Kb free=4652Kb
 bounds [0x0000027308710000, 0x0000027308980000, 0x0000027308cb0000]
 total_blobs=549 nmethods=209 adapters=253
 compilation: enabled
              stopped_count=0, restarted_count=0
 full_count=0
```

- CodeCache size dependent on JVM version -

  - Java 7 or below -
    - 32-bit - `32MB`
    - 64-bit - `48MB`
  - Java 8 or above with 64-bit JVM - upto `240MB`

- To change the code cache size -
  - `-XX:InitialCodeCacheSize` - size of the code cache when the application starts. The default size varies based on computer's available memory, but is generally around `160KB`.
  - `-XX:ReservedCodeCacheSize` - maximum size of the code cache.
  - `-XX:CodeCacheExpansionSize` - dictates how quickly the code cache should grow.

## JConsole

- Monitors the code cache on remote JVM - `Memory` tab > Select `Memory Pool "Code Cache"`
- On local JVM - when Java application runs, it stores its data in a particular folder which is then being monitored by the JConsole to find out what is the process ID of a running Java application.
- On windows - `C://Users/<username>/AppData/Local/Temp/hsperfdata_<username>/` - this should be writable for JConsole to work.

> [!WARNING]
> JConsole use `~2MB` of code cache size of application JVM during monitoring.
