# Golang

- Create a directory and intialize it as go module -
    - Using VS code - `Search` > Type `Go - Initialize go.mod` > Enter module name
    - This will create a `go.mod` file inside the directory.
    - The `go.mod` file declares your module’s import path and depedencies & their versions.

## Hello World

```
package main

import "fmt"

function main() {
    fmt.Println("Hello World!")
}
```

> [!NOTE]
> Every Go file must start with a package declaration to specify which package it belongs to, enabling the compiler and import system to organize and build your code. `package main` and `func main()` signifies that it is the entry point.

- To import multiple modules at the same time -
```
import (
    "fmt",
    "strings"
)
```

> [!NOTE]
> Anything starting with only upper-case letter is public. That's why we're able to access `fmt.Println`.

- Running the program - `go run helloWorld.go`
    - Compiles / builds the files into a temp directory and then executes it.

- Build - `go build`
    - If no path is mentioned here, it will write the output files into current working directory.
    - By default, builds the binaries for current OS, but to cross-compile for others (for eg - linux) - `GOOS=linux go build`

- Cleanup the binaries - `go clean`

> [!WARNING]
> Unused variables & methods are compilation-errors in Go.

## ` -=` vs `=`

- Short Variable Declaration (` -=`) -
    - Used for declaring and initializing new variables.
    - Can only be used inside functions.
    - Automatically infers the type.
- Assignment (`=`) -
    - Used to assign a new value to an already declared variable.
    - Can be used both inside and outside functions.
- Mixed Usage -
    - `a` is reassigned, `b` is newly declared -
    ```
    a  -= 1
    a, b  -= 2, 3
    ```

## Literals

- An explicitly specified number, character, or string.
- Common kinds - Integer, floating-point, rune, string (plus complex numbers which are less common).
- Integer Literals -
    - A sequence of numbers representing an integer.
    - Default base - Decimal (base 10)
    - Other bases (prefixed) -
        - Binary - 0b or 0B (base 2), eg 0b1010
        - Octal - 0o or 0O (base 8), eg 0o52 - leading 0 with no letter also represents octal but avoid it as it is confusing.
        - Hexadecimal - 0x or 0X (base 16), eg 0xFF

> [!TIP]
> Underscores for readability -
>   - Use underscores in numbers to improve readability.
>   - Allowed in the middle of the number - 1_234, 0b1010_1101
>   - Rules -
>       - Cannot start or end with underscore.
>       - Cannot have consecutive _ (1__234 invalid).

- Floating-Point Literals -
    - Numbers with a fractional portion, indicated by a decimal point.
    - Exponent notation - Use e (or E) followed by a positive or negative integer.
    - Example - 6.03e23 → 6.03 × 10²³
    - Hexadecimal floats -
        - Use 0x prefix and p for exponent.
        - Example - 0x12.34p5 → 582.5 in base 10
        - Underscores - Can be used to improve readability, same rules as integers.
        - Practical use - Prefer base 10; hexadecimal or binary rarely used unless working with networking or low-level operations.

- Rune Literals -
    - Represents a single character, enclosed in single quotes '...'
    - Forms of rune literals -
        - Single Unicode character - 'a'
        - 8-bit octal - '\141'
        - 8-bit hexadecimal - '\x61'
        - 16-bit hexadecimal - '\u0061'
        - 32-bit Unicode - '\U00000061'
    - Common escape sequences -
        - Newline - '\n'
        - Tab - '\t'
        - Single quote - '\''
        - Backslash - '\\'
    - Practical advice - Avoid numeric escapes unless they improve clarity.

- String Literals - two types -
    - Interpreted string literals (double quotes "...") -
        - Contain zero or more rune literals.
        - Called “interpreted” because escape sequences and numeric rune literals are converted to actual characters.
    - Raw string literals (backquotes `...`) -
        - No escape sequences; everything is included as-is.
        - Useful for including backslashes, quotes, and multiline text.
        - Example -
        ```
        s := `Greetings and
            "Salutations"`
        ```

> [!NOTE]
> All literals in Go are untyped by default. If the type isn’t explicitly declared (via var or :=), Go assigns a default type for the literal. Default types depend on the literal category (int, float, string, etc.)

## Booleans

- Type  `bool`
- Values - `true` or `false`
- Zero value - `false` (default if not initialized)
- Example -
```
var flag bool       // defaults to false
var isAwesome = true
```

## Numeric Types

### Integer Types

| Type   | Value Range                     |
|--------|---------------------------------|
| int8   | –128 to 127                     |
| int16  | –32768 to 32767                 |
| int32  | –2147483648 to 2147483647       |
| int64  | –9223372036854775808 to 9223372036854775807 |
| uint8  | 0 to 255                        |
| uint16 | 0 to 65535                       |
| uint32 | 0 to 4294967295                  |
| uint64 | 0 to 18446744073709551615       |

