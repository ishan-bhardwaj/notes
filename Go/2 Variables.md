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
