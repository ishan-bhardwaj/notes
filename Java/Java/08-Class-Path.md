## The Class Path

- The class path tells Java where to find `.class` and `.jar` files.
- Examples -  
  - Linux / macOS - `/home/user/classdir:.:/home/user/archives/archive.jar`
  - Windows - `c:\classdir;.;c:\archives\archive.jar`
  - where `.` = current directory

- Wildcard for JARs -
  - `/home/user/archives/*`
  - Includes all JARs
  - Does NOT include `.class` files

- Search Order -
  - Java API
  - Class path directories
  - JAR files

- Setting the Class Path -
  - Use command-line option - 
    - `java -classpath path MyProgram`
    - Aliases -
      - `-cp`
      - `--class-path`

  - Using `CLASSPATH` environment variable -
    - Unix shell - `export CLASSPATH=/home/user/classdir:.:/home/user/archives/archive.jar`
    - Windows shell - `set CLASSPATH=c:\classdir;.;c:\archives\archive.jar`
    - The class path is set until the shell exits.

> [!TIP]
> When using a JAR in jshell, launch it with - `CLASSPATH=archive.jar jshell`.

> [!TIP]
> Classes can also be loaded from the _module path_.

## JAR Files
  
- JAR = compressed archive.
- Contains -
  - `.class` files
  - Resources (images, sounds, config)
- Used to distribute applications and libraries.
- Built on ZIP format.

- Creating jar files -
  - Using `jar` tool - 
    - In the default JDK installation, it’s in the `$JAVA_HOME/bin` directory.
    - Syntax - `jar cvf jarFileName file1 file2 . . .`
    - Example - `jar cvf CalculatorClasses.jar *.class icon.png`

- `jar` program options -

| Option | Description                                                                                               |
| ------ | --------------------------------------------------------------------------------------------------------- |
| `c`    | Creates a new (or empty) JAR archive and adds files to it. Directories are processed __recursively__.     |
| `C`    | Temporarily changes the directory when adding files. <br>Example:<br>`jar cvf app.jar -C classes *.class` |
| `e`    | Specifies the __entry point__ (main class) in the manifest.                                               |
| `f`    | Specifies the __JAR file name__ as the next argument. <br>If omitted, `jar` uses standard input/output.   |
| `i`    | Creates an __index file__ for faster class lookup in large archives.                                      |
| `m`    | Adds a __custom manifest__ file to the JAR.                                                               |
| `M`    | Prevents creation of a manifest file.                                                                     |
| `t`    | Displays the __table of contents__ of the JAR.                                                            |
| `u`    | __Updates__ an existing JAR file.                                                                         |
| `v`    | Enables __verbose output__.                                                                               |
| `x`    | __Extracts__ files from the JAR. <br>If no file names are given, extracts everything.                     |
| `0`    | Stores files __without ZIP compression__.                                                                 |


## The Manifest

- Each JAR file contains a manifest file that describes special features of the archive.
- The manifest file is called `MANIFEST.MF` and is located in a special `META-INF` subdirectory of the JAR file.
- Minimum legal manifest - `Manifest-Version: 1.0`
- The manifest entries are grouped into sections -
  - The first section in the manifest is called the main section. It applies to the whole JAR file. 
  - Subsequent entries can specify properties of named entities such as individual files, packages, or URLs. 
  - Those entries must begin with a Name entry. 
  - Sections are separated by blank lines.
  - Example -
  ```
  Manifest-Version: 1.0
  lines describing this archive

  Name: Woozle.class
  lines describing this file
  Name: com/mycompany/mypkg/
  lines describing this package
  ```

- To edit the manifest - 
  - Place the lines that you want to add to the manifest into a text file. 
  - Then run - `jar cfm jarFileName manifestFileName . . .`
  - For example, to make a new JAR file with a manifest, run - 
  ```
  jar cfm MyArchive.jar manifest.mf com/mycompany/mypkg/*.class
  ```

- To update the manifest of an existing JAR file, place the additions into a text file and use a command such as - 
```
jar ufm MyArchive.jar manifest-additions.mf
```

- Executable jar files -
  - Use the `e` option of the jar command to specify the entry point of your program — the class that you would normally specify when invoking the java program launcher -
    - `jar cvfe MyProgram.jar com.mycompany.mypkg.MainAppClass files to add`
  - Alternatively, you can specify the main class of your program in the manifest, including a statement of the form -
    - `Main-Class: com.mycompany.mypkg.MainAppClass`
    - Do not add a `.class` extension to the main class name.

> [!WARNING]
> The last line in the manifest must end with a newline character. Otherwise, the manifest will not be read correctly. It is a common error to produce a text file containing just the Main-Class line without a line terminator.

- For backward compatibility, version-specific class files are placed in the `META-INF/versions` directory.
- To add versioned class files, use the `--release` flag - 
```
jar uf MyProgram.jar --release 9 Application.class
```

- To build a multi-release JAR file from scratch, use the `-C` option and switch to a different class file directory for each version - 
```
jar cf MyProgram.jar -C bin/8 . --release 9 -C bin/9 Application.class
```

- When compiling for different releases, use the `--release` flag and the `-d` flag to specify the output directory - 
  - `javac -d bin/8 --release 8 . . . `
  - The `-d` option creates the directory if it doesn’t exist.

> [!TIP]
> The sole purpose of multi-release JARs is to enable a particular version of your program or library to work with multiple JDK releases. If you add functionality or change an API, you should provide a new version of the JAR instead.

- Tools such as `javap` are not retrofitted to handle multi-release JAR files. 
  - If you call - `javap -classpath MyProgram.jar Application.class` - you get the base version of the class (which, after all, is supposed to have the same public API as the newer version). 
  - If you must look at the newer version, call - 
  ```
  javap -classpath MyProgram.jar\!/META-INF/versions/9/Application.class
  ```

> [!TIP]
> You can use the `JDK_JAVA_OPTIONS` environment variable to pass command-line options to the java launcher - `export JDK_JAVA_OPTIONS='--class-path /home/user/classdir -enableassertions'`
