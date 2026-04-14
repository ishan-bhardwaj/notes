# Python

## Bytecode

- Python compiles source code into _bytecode_ -
    - Saves the bytecode in `.pyc` (`.py` source, compiled) files inside `_pycache_` directory.
    - `_pycache_` directory is located in the directory where the source files reside.
    - Also includes python version in the file name.
    - Eg - `script0.cpython-312.pyc` for `3.12`

- Bytecode acts as a _startup-speed optimization_ by reusing `.pyc` files if -
    - The bytecode file exists.
    - The source file hasn’t changed since the bytecode was created.
    - The Python interpreter version matches the one that generated the bytecode.

- __Recompiles when__ -
    - Source code changes -
        - Python stores metadata about the source file inside the `.pyc` file -
            - Last-modified timestamp and file size
            - (Optionally) a hash of the source file
        - When loading bytecode, Python compares this metadata with the current source file.
        - If the source has changed, Python automatically recompiles the file and writes a new `.pyc`.
    - Python version changes -
        - Bytecode filenames include a version tag (e.g., `script0.cpython-312.pyc`).
        - Ensures bytecode compiled by one Python version is not reused by another.
        - If the program is run with different Python version, it recompiles the source and generates a new `.pyc`.

> [!NOTE]
> If Python cannot write `.pyc` files, the program still runs.
>
> The bytecode is compiled in memory and discarded when the program exits, so startup may be slightly slower next time.

- Python can run programs from `.pyc` only (source not required), but packaging may need tools like `compileall` or frozen executables.
- Bytecode is mainly an import optimization -
    - Created for imported modules, not top-level scripts run directly.
    - A module is compiled only once per run - later imports reuse it.

> [!NOTE]
> No `.pyc` is saved for interactive prompt code.

## Python Virtual Machine (PVM)

- PVM is Python’s runtime engine that runs compiled bytecode.
- Always present inside the Python interpreter.
- Python bytecode is not CPU machine code (not Intel/ARM instructions); it’s a Python-specific format.

> [!TIP]
> `eval()` runs Python expressions and returns value.
>
> `exec()` runs statements/code blocks, but do not return value.

## Python Implementations

- __CPython__ (standard) -
    - Reference implementation written in C.
    - Default Python from python.org and most systems.
- __Jython__ (Python + Java) -
    - Compiles Python → Java bytecode → runs on JVM.
- __IronPython__ (Python + .NET) - 
    - Written in C#.
    - Runs on .NET / Mono and integrates with .NET libraries.
- __Stackless Python__ (concurrency) -
    - CPython variant focused on lightweight concurrency.
    - Efficient multiprocessing and small-stack environments.
    - Used in _EVE Online_.
- __PyPy__ (general speed) -
    - Python implementation with JIT compiler.
    - Converts bytecode → machine code during runtime.
    - Often faster and may use less memory.
- __Numba__ (numeric JIT) -
    - JIT compiler via decorators.
    - Speeds up NumPy, math loops, scientific code.
    - Supports parallel execution.
- __Shed Skin__ (AOT compiler) -
    - Compiles Python → C++ → machine code (before running).
    - Requires restricted, statically-typed subset of Python.
    - Can outperform CPython/JIT for compatible code.
- __PyThran__ (numeric AOT) -
    - AOT compiler for scientific/numeric Python.
    - Python → C++ → native modules.
- __Cython__ (Python/C hybrid) -
    - Python + optional C types.
    - Compiles to C extensions for high performance.
    - Great for wrapping C libraries.
- __MicroPython__ (embedded Python) -
    - Lightweight Python for microcontrollers & constrained devices.
    - Smaller language + standard library subset.
- Other notable projects -
    - __Cinder__ – Meta’s CPython with JIT.
    - __Pyston__ – CPython fork with optimizations.
    - __Nuitka__ – Python → C AOT compiler (can target WebAssembly).

## Standalone Executables / Frozen Binaries

- Python apps can be packaged into self-contained executables (no Python install needed).
- Bundle includes -
    - Your program bytecode
    - The Python Virtual Machine (interpreter)
    - Required libraries and dependencies
- Output examples -
    - Windows - `.exe`
    - macOS - `.app`
    - Android - `.apk` / `.aab`
- Popular tools -
    - __py2exe__ - Windows
    - __PyInstaller__ / __cx_Freeze__ - Windows, macOS, Linux
    - __py2app__ - macOS
    - __Buildozer__ / __Briefcase__ - Android & iOS
- Executable properties -
    - Not true native compilation; the program still runs bytecode on a virtual machine.
    - Performance is roughly the same as running the original source code.
    - Executables are larger because they include the Python runtime.
    - Source code is harder to inspect since it is bundled as bytecode.

## Variables

```
a = 10
print(a)        # 10

a = 20
print(a)        # 20

PI = 3.14
```

- Generally, we use upper-case letters to denote constants, though they are still variables.

> [!TIP]
> Division operator (`/`)  will always yield a float. To remove any decimal digits i.e. integer division, use `//`.
> ```
> 4 / 2 = 2.0
> 5 / 2 = 2.5
> 4 // 2 = 2
> 5 // 2 = 2.5
> ```

## Strings

- Can be represented either using double (`"`) or single quotes (`'`).
- Multi-line strings -
```
"""Hello

World!
"""
```

- Multi-line strings are useful for documentation.
- Concatenation -
```
"Hello" + " World"          # Hello World
"My age is: " + 30          # Error!
"My age is: " + str(30)     # My age is: 30
```

> [!TIP]
> `"X" * 5` returns `XXXXX`.

- f-Interpolation -
```
age = 30
print(f"Age: {age}")        # Age: 30
```

> [!WARNING]
> ```
> age = 20
> prompt = f"Age: {age}"
> print(prompt)             # Age: 20
> 
> age = 30
> print(prompt)             # Age: 20
> ```

- `format` interpolation -
```
age = 20
prompt = "Age: {}"
print(prompt.format(age))        # Age: 20

age = 30
print(prompt.format(age))        # Age: 30
```

- Using named parameters in `format` -
```
age = 30
prompt = "Age: {age}"
print(promt.format(age=age))     # Age: 30
```

## User Input

```
username = input("Enter your name:")
print(f"Hello {username}")
```

- `input` always return a `string`.

## Boolean

- `True` and `False`
- AND operator - `and`
- OR operator - `or`

```
bool(0)             # False
bool(10)            # True

bool("")            # False
bool("Hello")       # True

bool([])            # False
bool([1, 2, 3, 4])  # True
```

## If-else

```
if <condition1>:
    <...>
elif <condition2>:
    <...>
else:
    <...>
```

## Loops

### While loops

```
while <condition>:
    <...>
```

### for loops

- Iterating collections -
```
l = [1, 2, 3, 4]

for i in l:
    <...>
```

> [!TIP]
> Use underscore (`_`) if the variables are unused, eg - `for _ in l`

- Iterating collections by index - 
```
for i in range(len(l)):
    <...>
```

- Iterating dictionaries -
    - Iterating keys - `for k in my_dict`
    - Iterating values - `for v in my_dict.values()`
    - Iterating both - `for k, v in my_dict.items()`

> [!TIP]
> Use `break` to break out of the loop and `continue` to skip to the next iteration.

- `else` keyword with loops -
    - Applicable on both `for` and `while` loops -
    - Runs the block of code when loop iterates over all the elements without encountering a `break` or an error -
    ```
    for i in my_dict:
        <...>
    else:
        <...>
    ```
