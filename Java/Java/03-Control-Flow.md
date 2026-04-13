# Control Flow

## If-else
```
if (condition1) {
  statement1
} else if (condition2) {
  statement2
} else {
  statement3
}
```

## While loops
```
while (condition) {
  statement
}
```

- __do while loops__ -
```
do {
  statement
} while (condition)
```

## for loops
  
- Syntax -
```
for (initialization; condition; update) {
  statement
}
```
  
- Example -
```
for (int i = 1; i <= 10; i++) {
  statement
}
  ```

> [!TIP]
> Java lets you put almost anything in those three parts, but a good programming practice is to use the for loop only to control a single counter variable. Otherwise the code becomes confusing and hard to read, eg - 
> ```
> for (int i = 0; i < 10; System.out.println(i++)) { ... }
> ```

- Initialization can declare multiple variables, provided they are of the same type and the update expression can contain multiple comma-separated expressions -
```
for (int i = 1, j = 10; i <= 10; i++, j--) { . . . }
```

## Labeled `break`

- The labeled `break` statement lets you break out of multiple nested loops.
- Example with `while` loop -
```
int n;

read_data:
while ( ... ) {                       // this loop statement is tagged with the label
    ...
    for ( ... ) {                     // this inner loop is not labeled
        if (n < 0) {                  
            break read_data;          // break out of read_data loop
        }
        ...
    }
}

// this statement is executed immediately after the labeled break
statement
```

- Example with `if` statement -
```
label: {
    ...
    if (condition) break label;       // exits block
    ...
}
// jumps here when the break statement executes
```

## `continue` statement

- The `continue` statement transfers control to the header of the innermost enclosing loop.
- Example with `while` -
```
while (sum < goal) {
    n = Integer.parseInt(IO.readln("Enter a number: "));
    if (n < 0) continue;
    sum += n;                         // not executed if n < 0
}
```

- Example with `for` - jumps to the “update” part of the for loop -
```
for (count = 1; count <= 100; count++) {
    n = Integer.parseInt(IO.readln("Enter a number, -1 to quit: "));
    if (n < 0) continue;
    sum += n;                         // not executed if n < 0 - jumps to the count++ statement
}
```

> [!TIP]
> There is also a labeled form of the `continue` statement that jumps to the header of the loop with the matching label.

## Switch Expressions


```
String seasonName = switch (seasonCode) {
  case 0 -> "Spring";
  case 1 -> "Summer";
  case 2 -> "Fall";
  case 3, 4, 5 -> "Winter";                 // multiple labels
  default -> "???";
};
```

- A case label must be a compile-time constant whose type matches the selector type.

> [!TIP]
> Java 23 Preview - Switch selector can be float, double, long, or boolean.

- __Enums in Switch__ -
  - Enum type labels can omit enum name.
    ```
    enum Size { SMALL, MEDIUM, LARGE, EXTRA_LARGE }

    String label = switch (itemSize) {
      case SMALL -> "S";                    // no need to use Size.SMALL
      case MEDIUM -> "M";
      case LARGE -> "L";
      case EXTRA_LARGE -> "XL";
    };
    ```

  - If all enum values covered, `default` is optional; otherwise required.
  - Switch with numeric or String selector must always have `default`.
  
- __Null Selector Handling__ -
  - If selector is `null`, `NullPointerException` is thrown.
  - To handle null, add - `case null -> "???";`
  - Note that `default` does NOT match `null`.

> [!TIP]
> If you cannot compute the result in a single expression, use braces and a `yield` statement -
> ```
> case "Spring" -> {
>    IO.println("spring time!");
>    yield 6;
> }
> ```

> [!NOTE]
> You cannot use `return`, `break`, or `continue` statements in a switch expression.

## Switch statements

```
switch (choice) {
    case 1:
        ...
        break;
    case 2:
        ...
        break;
    case 3:
        ...
        break;
    case 4:
        ...
        break;
    default:
        IO.println("Bad input");
}
```

- __Fallthrough behavior__ - if a case does not end with `break`, execution continues into the next `case`.

- To detect fallthrough mistakes - 
  - Compile with - `-Xlint:fallthrough`
  - The compiler will warn when a case does not end with break.
  - But if fallthrough is intentional, use - `@SuppressWarnings("fallthrough")` to suppress warnings for that method.
