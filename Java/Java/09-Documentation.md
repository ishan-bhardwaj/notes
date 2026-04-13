# Documentation

- `javadoc` tool generates HTML documentation from your source files from the comments that start with a ¬ and ends with a `*/`.
- Each comment is placed immediately above the member it describes. 
- The `javadoc` utility extracts information for the following items -
  - Modules
  - Packages
  - Public classes and interfaces
  - Public and protected fields
  - Public and protected constructors and methods
- Each `/** . . . */` documentation comment contains free-form text followed by tags. 
  - A tag starts with an `@`, such as `@since` or `@param`.
  - The first sentence of the free-form text should be a summary statement. The javadoc utility automatically generates summary pages that extract these sentences.
  - The most common javadoc tags are block tags. They must appear at the beginning of a line and start with `@`, optionally preceded by whitespace, the comment delimiter `/**`, or leading `*` which are often used for multiline comments. 
  - In contrast, inline tags are enclosed in braces - `{@tagname contents}`. 
  - The contents may contain braces, but they must be balanced. Examples are the `@code` and `@link` tags.

- JavaDoc text can contain HTML tags such as `<em>` or `ul` - they are passed through to the generated HTML pages.

> [!NOTE]
> Starting with Java 23, you can author JavaDoc comments in Markdown instead of plain text and HTML. Markdown comments are delimited with `///` in front of each line, instead of `/**` and `*/`. JavaDoc follows the CommonMarc specification, and in addition supports the Github Flavored Markdown extension for tables.

- __Class Comments__ - must be placed after any import statements, directly before the class definition.
- __Method Comments__ -
  - Each method comment must immediately precede the method that it describes.
  - In addition to the general-purpose tags, you can use the following tags -
    - `@param` variable description - 
      - This tag adds an entry to the “parameters” section of the current method. 
      - The description can span multiple lines and can use HTML tags. 
      - All `@param` tags for one method must be kept together.
    - `@return` description -
      - This tag adds a “returns” section to the current method. 
      - The description can span multiple lines and can use HTML tags.
    - `@throws` class description -
      - This tag adds a note that this method may throw an exception. 

- It can be tedious to write comments for methods whose description and return value are identical, such as -
  ```
  /**
  * Returns the name of the employee.
  * @return  the name of the employee
  */
  ```

  - In such cases, consider the inline form of @return introduced in Java 16 -
  ```
  /**
  * {@return the name of the employee}
  */
  ```

  - The description section becomes “Returns the name of the employee.”, and a “Returns” section with the same contents is added.

> [!NOTE]
> If you add `{@inheritDoc}` into a method description or the `@param`, `@return`, or `@throws` tag bodies, then the documentation from the method in a superclass or interface is copied verbatim.

- __Field Comments__ -
  - You only need to document public fields—generally that means static constants. 
  - For example -
  ```
  /**
  * The "Hearts" card suit
  */
  public static final int HEARTS = 1;
  ```

- __Package Comments__ -
  - To generate package comments, you need to add a separate file in each package directory.
  - Two choices -
    - Supply a Java file named `package-info.java` -
      - The file must contain an initial documentation comment, delimited with `/**` and `*/`, followed by a `package` statement. 
      - It should contain no further code or comments.
    - Supply an HTML file named `package.html` -
      - All text between the tags `<body>. . .</body>` is extracted.
  
- For multiline code displays in an HTML `pre` tag, you can use -
  ```
  /**
  * . . .
  * <pre>{@code
  *     . . .
  *     . . .
  * }</pre>
  */
  ```

  - or, since Java 18 -
  ```
  /**
  * . . .
  * {@snippet :
  *     . . .
  *     . . .
  * }
  */
  ```

- __Links__ -
  - You can use hyperlinks to other relevant parts of the javadoc documentation, or to external documents, with the `@see` and `@link` tags.
  - The tag `@see` reference adds a hyperlink in the “see also” section.
    - Where `reference` can be one of the following -
    ```
    package.class#member label
    <a href=". . .">label</a>
    "text"
    ```

  - Example - `@see com.example.Employee#raiseSalary(double)`
    - makes a link to the `raiseSalary(double)` method in the `com.example.Employee` class.
    - You can omit the name of the package, or both the package and class names. Then, the member will be located in the current package or class.
    - Note that you must use a #, not a period, to separate the class from the method or variable name.
  
  - Constructors have the special name `<init>`, not the name of the class, such as - `@see com.example.Employee#<init>()`

  - You can specify an optional `label` after the member that will appear as the link anchor. If you omit the label, the user will see the member name.

  - If the `@see` tag is followed by a `<` character, then you need to specify a hyperlink -
    - You can link to any URL you like, eg - `@see <a href="horstmann.com/corejava.html">The Core Java home page</a>`

  - If the `@see` tag is followed by a `"` character, then the text is displayed in the “see also” section. 
    - For example - `@see "Core Java Volume 2"`

  - You can add multiple `@see` tags for one member, but you must keep them all together.

  - You can place hyperlinks to other classes or methods anywhere in any of your documentation comments. 
    - Insert a tag of the form - `{@link package.class#member}` - anywhere in a comment. 
    - The member reference follows the same rules as for the `@see` tag.

  - In a code snippet, place the `@link` tag in the comment, so that it doesn’t interfere with the code -
  ```
  {@snippet
    IO.println(); // @link substring=println target=java.lang.IO#println()
  }
  ```

  - Since Java 20, ids are automatically generated for level 2 and level 3 headings.
    - Example - `<h2>General Principles</h2>`
    - Gets an id general-principles-heading, which you can refer from `@see` and `@link` tags. 
    - You need two `#` symbols to link to an id -
    ```
    {@link com.horstmann.corejava.Employee##general-principles-heading}
    ```

  - Use `@linkplain` instead of `@link` if a link should be displayed in the plain font instead of the code font.

> [!NOTE]
> If your comments contain links to other files, such as images (for example, diagrams or images of user interface components), place those files into a subdirectory, named `doc-files`, of the directory containing the source file. 
>
> The javadoc utility will copy the `doc-files` directories and their contents from the source directory to the documentation directory. 
>
> You need to use the `doc-files` directory in your `link`, for example` <img src="doc-files/uml.png" alt="UML diagram"/>`.

- __General Comments__ -
  
| Tag                | Purpose                                                                                  |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `@since text`      | Specifies the version when the feature was introduced (e.g. `@since 1.7.1`).             |
| `@author name`     | Lists the author(s) of the class (optional; VCS is preferred today).                     |
| `@version text`    | Specifies the current version of the class.                                              |
| `@deprecated text` | Explains why the feature is deprecated and what to use instead. Used with `@Deprecated`. |

- __Inline Tags__ -

| Tag                        | Purpose                                             |
| -------------------------- | --------------------------------------------------- |
| `{@value constant}`        | Inserts the value of a constant field.              |
| `{@value format constant}` | Inserts formatted constant value (Java 20+).        |
| `{@index entry}`           | Adds an entry to the generated documentation index. |

  - Example - `{@value %X Integer#MAX_VALUE}   // → 7FFFFFFF`

- __Code Snippets (Java 18+)__ -

| Syntax                                                 | Description                                 |
| ------------------------------------------------------ | ------------------------------------------- |
| `{@snippet file=EmployeeDemo.java}`                    | Includes an entire source file              |
| `{@snippet class=com.horstmann.corejava.EmployeeDemo}` | Includes a class                            |
| 📁                                                     | Files must be in `snippet-files/` directory |

- __Snippet Regions__ -
```
// @start region=default-employee
var e = new Employee();
String name = e.getName();
// @end
```

- __Snippet Decorations__ - 

| Decoration            | Example                                                 |
| --------------------- | ------------------------------------------------------- |
| Highlight (substring) | `// @highlight substring=new`                           |
| Highlight (regex)     | `// @highlight regex=get[A-Z][a-z]+`                    |
| Replace               | `// @replace regex=([^)]+) replacement="(..., ...)"`    |
| Link                  | `// @link substring=Card target=package#Class.<init>()` |

- __Comment Extraction (javadoc tool)__ -

| Task              | Command                       |
| ----------------- | ----------------------------- |
| Single package    | `javadoc -d docs packageName` |
| Multiple packages | `javadoc -d docs pkg1 pkg2`   |
| Unnamed package   | `javadoc -d docs *.java`      |

- __Important Options__ -

| Option           | Effect                           |
| ---------------- | -------------------------------- |
| `-d dir`         | Output directory                 |
| `-author`        | Includes `@author` tags          |
| `-version`       | Includes `@version` tags         |
| `-link URL`      | Links to external API docs       |
| `-linksource`    | Generates HTML source with links |
| `-overview file` | Adds project-wide overview       |

- __Overview File__ -

| File              | Purpose                          |
| ----------------- | -------------------------------- |
| `overview.html`   | Project-wide documentation       |
| Extracted content | Text inside `<body>...</body>`   |
| Shown as          | “Overview” tab in generated docs |
