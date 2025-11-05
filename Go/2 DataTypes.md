# DataTypes

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

- Go provides both signed and unsigned integers.
- Zero value - `0`

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

- Special Integer types -
    - `byte` -
        - Alias for `uint8`.
        - Legal to assign, compare, or perform mathematical operations with `uint8`.
        - Commonly used instead of `uint8`.
    - `int` -
        - Signed integer type.
        - Default type for integer literals.
        - On 32-bit CPUs → behaves like `int32`.
        - On most 64-bit CPUs → behaves like `int64`.
        - Because `int` isn’t consistent across platforms, assigning, comparing, or performing mathematical operations between `int` and `int32` or int`64 without an explicit type conversion is a compile-time error.
    - `uint` -
        - Unsigned integer.
        - Follows the same platform-dependent rules as `int` but values are always ≥ 0.
    - `rune` -
        - Alias for `int32`.
        - Represents a Unicode code point.
    - `uintptr` -
        - Unsigned integer type large enough to hold pointer values.
        - Mainly used for low-level or system programming.

> [!TIP]
> Prefer `int` and `byte` for general use; use platform-independent types like `int32`, `int64`, `uint32`, `uint64` when working with fixed-size data.

> [!TIP]
> Always use explicit type conversion when mixing `int` with `int32` or `int64` to avoid compile-time errors.

> [!NOTE]
> Before Go had generics, functions had to be duplicated for different types (e.g., `int64` vs `uint64`) to implement the same algorithm. Callers used type conversions to pass values. Example: `strconv.FormatInt` and `FormatUint`.

- Operators -
    - Arithmetic Operators -
        - `+`, `-`, `*`, `/`, `%` (modulus)
        - Integer division - Result is truncated toward zero. To get a floating-point result, convert integers to float first.
        - Division by zero - Causes a panic.
        - Compound assignment operators - `+=`, `-=`, `*=`, `/=`, `%=`
        - Comparison operators - `==`, `!=`, `>`, `>=`, `<`, `<=`
    - Bitwise Operators -
        - Bit shifts - `<<` (left), `>>` (right)
        - Bit masks - `&` (AND), `|` (OR), `^` (XOR), `&^` (AND NOT)
        - Compound assignments - `&=`, `|=`, `^=`, `&^=`, `<<=`, `>>=`

### Floating-point Types

- Two types -
    - `float32` - `~3.4e38` to `~1.4e-45`
    - `float64` - `~1.8e308` to `~4.9e-324`
- Zero value - `0`
- Default literal type - `float64`
- Go follows IEEE 754 for floating-point math.
- Precision -
    - `float32` → ~6–7 decimal digits
    - `float64` → ~15–16 decimal digits
- Recommendation - Use `float64` unless memory or format compatibility requires otherwise.
- Not suitable for values requiring exact decimal representation (e.g., money).
- Operators -
    - Arithmetic - `+`, `-`, `*`, `/`
    - Modulo `%` is not allowed for floats.
    - Comparison - `==`, `!=`, `<`, `<=`, `>`, `>=`
- Special Cases -
    - Dividing nonzero float by `0` = `+Inf` or `-Inf`
    - Dividing `0` by `0` = `NaN` (Not a Number)

> [!WARNING]
> Avoid using `==`/`!=` due to floating-point inaccuracy.
> Use a maximum allowed variance (epsilon) instead -
> ```
> if math.Abs(a - b) < epsilon
> ```

### Complex Types

- Go has first-class support for complex numbers, though they are rarely used.
- Two Types -
    - `complex64` - uses `float32` for real and imaginary parts
    - `complex128` - uses `float64` for real and imaginary parts
- Declared with the built-in complex function -
```
var complexNum = complex(20.3, 10.2)
```

- Zero value - Both `complex64` and `complex128` → real = 0, imag = 0
- Type Rules -
    - Untyped constants/literals → default type `complex128`
    - Both `float32` → `complex64`
    - One `float32` + compatible untyped constant → `complex64`
    - Otherwise → `complex128`
- Imaginary Literals -
    - Represent the imaginary portion of a complex number
    - Suffix `i` like a float literal - `var c = 5i`
- For serious numerical computing, consider `Gonum` library or other languages.
- Operators -
    - Standard floating-point arithmetic operators - `+`, `-`, `*`, `/`
    - Comparisons - `==`, `!=` (use epsilon technique due to precision limits)
    - Extract real/imaginary parts - `real(x)` & `imag(x)`
    - `math/cmplx` package provides additional functions for `complex128`

## Strings

- Zero value is the empty string `""`.
- Supports Unicode.
- Immutable - You can reassign a string variable but cannot modify the string itself.
- Comparison operators - `==`, `!=`, `<`, `<=`, `>`, `>=`
- Concatenation - `+`

## Runes

- Represents a single Unicode code point.
- `rune` is an alias for `int32`.
- Use rune to clarify intent when referring to characters -
```
var myFirstInitial rune = 'J'  // correct
var myLastInitial int32 = 'B'  // legal but confusing
```

## Explicit Type Conversion

- Go does not allow automatic type promotion between numeric types.
- Must explicitly convert variables to the same type before operations. Go favors explicit conversions for clarity over conciseness.
- Example -
```
var x int = 10
var y float64 = 30.2
var sum1 float64 = float64(x) + y
var sum2 int = x + int(y)
fmt.Println(sum1, sum2) // 40.2 40
```

- Applies to different-sized integers as well -
```
var x int = 10
var b byte = 100
var sum3 int = x + int(b)
var sum4 byte = byte(x) + b
```

- No truthiness -
    - Cannot convert other types to `bool`.
    - Must use comparison operators (`==`, `!=`, `<`, `>` etc.) to get a boolean value.

> [!NOTE]
> Literals in Go are untyped until assigned. For eg, you can assign integer literals to floating-point variables -
> ```
> var x float64 = 10
> var y float64 = 200.3 * 5
> ```
>
> Restrictions -
>   - Cannot assign a string literal to a numeric variable, or vice versa.
>   - Cannot assign a literal that exceeds the variable’s capacity (e.g., assigning 1000 to a `byte`).



## Variable Declarations

- Using `var` -
    - Explicit type and assignment - `var x int = 10`
    - Type inference (omit type if RHS matches expected type) - `var x = 10  // x inferred as int`
    - Zero-value declaration (omit assignment) - `var x int  // x = 0`
    - Multiple variables, same type -
    ```
    var x, y int = 10, 20
    var x, y int  // both zero values
    ```

    - Multiple variables, different types - `var x, y = 10, "hello"`
    - Declaration list (for readability, multiple variables at once) -
    ```
    var (
        x int
        y = 20
        z int = 30
        d, e = 40, "hello"
        f, g string
    )
    ```

- Short Declaration (`:=`) -
    - Used inside functions only.
    - Automatically infers type - `x := 10`
    - Declare multiple variables at once - `x, y := 10, "hello"`
    - Can reassign existing variables while declaring new ones - 
    ```
    x := 10
    x, y := 30, "hello"  // x reused, y new
    ```
    - Cannot be used at package level - use `var` instead.

### Best Practises 

- Prefer `:=` inside functions for brevity and clarity.
- Use var when -
    - Initializing to zero value (`var x int`)
    - Assigning an untyped constant/literal with a specific type (`var x byte = 20`)
    - Avoiding shadowing when mixing new & existing variables.
- Use multiple declarations only for multiple return values from a function.
- Avoid mutable package-level variables; prefer effectively immutable variables.

> [!TIP]
> `const` can be used for immutable values.

## `const`

- Immutable - cannot be reassigned.
- Can be package-level or function-level.
- Eg -
```
const x int64 = 10
x = x + 1 // compile-time error
```

- Can declare multiple constants together:
```
const (
    idKey   = "id"
    nameKey = "name"
)
```

- Constants are named literals. They can only hold values known at compile time. Cannot declare runtime-calculated values as constants -
```
x := 5
y := 10
const z = x + y // compile-time error
```

### Typed vs Untyped Constants

- Untyped constants - behave like literals, flexible with multiple numeric types.
```
const x = 10
var y int = x
var z float64 = x
var d byte = x
```

- Typed constants - enforce a specific type. Can only be assigned to variables of that type.
```
const typedX int = 10
var y float64 = typedX // compile-time error
```

> [!TIP]
> Keep constants untyped for flexibility. Use typed constants when you need type enforcement (e.g., enumerations with `iota`).

- Immutability Limitations -
    - Go has no immutable runtime variables: arrays, slices, maps, structs, and struct fields cannot be declared immutable.
    - Immutability is less critical in Go because of call-by-value semantics and explicit assignment.

> [!NOTE]
> Unused Variables and Constants -
>   - Variables - must be read at least once; unused local variables are compile-time errors.
>   - Assignments - compiler ignores unused writes if the variable is read elsewhere.
>   - Package-level variables - compiler allows unread variables (but avoid them for clarity).
>   - Constants - can be unread - compiler ignores them as they have no runtime impact.

